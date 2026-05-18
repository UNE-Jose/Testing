# Arquitectura de Datos: Grafos y Recuperación Vectorial

---

## 1. Fundamentos y Modelado de Grafos en Neo4j

### 1.1. Introducción Estratégica: El Fin de los Joins Implícitos
La transición del paradigma relacional al de grafos marca el fin de la reconstrucción costosa de relaciones en tiempo de ejecución. 

* **Modelo Relacional (SQL):** Las conexiones son implícitas y dependen de claves foráneas.
* **Modelo de Grafos (Neo4j):** Las relaciones son entidades de primer nivel con un coste de navegación constante ($O(1)$ por salto). 

Esta estructura permite el análisis de relaciones complejas y profundas que en modelos relacionales colapsarían debido a la explosión de *joins*.

### 1.2. Anatomía de Cypher (`CREATE`, `MATCH`, `MERGE`)
Cypher es un lenguaje de declaración de patrones. Para un arquitecto, la distinción entre crear y asegurar identidad es crítica para la integridad del sistema.

| Comando | Función Principal | Gestión de Identidad vs. Propiedades |
| :--- | :--- | :--- |
| **`CREATE`** | Generación absoluta. | No verifica existencia. Crea duplicados si se repite. Útil para inserciones masivas de datos únicos. |
| **`MERGE`** | *"Get or Create"* determinista. | El patrón en el `MERGE` actúa como Identidad. Se debe usar `ON CREATE SET` para asignar propiedades iniciales y `ON MATCH SET` para actualizaciones. |
| **`MATCH`** | Recuperación de patrones. | Define la topología a buscar. No es un filtro de tabla, es una búsqueda de sub-grafos. |

#### Snippet Crítico de `MERGE`
El siguiente bloque separa la identidad (`id`) de los datos variables (`nombre`, `fecha`):

```cypher
MERGE (p:Persona {id: "USR-123"})
ON CREATE SET p.nombre = "Ana", p.registrado = timestamp()
ON MATCH SET p.ultimo_acceso = timestamp()

```

### 1.3. Patrones y Direccionalidad

La direccionalidad define el flujo semántico y la eficiencia del índice:

* **Relación dirigida (Salida):** `(n:Persona)-[:TRABAJA_EN]->(e:Empresa)`
* **Relación inversa (Entrada):** `(n:Persona)<-[:JEFE_DE]-(m:Persona)`
* **Relación no dirigida (Amplitud máxima):** `(n:Persona)-[:CONOCE]-(m:Persona)`

> 💡 **Impacto Arquitectónico:** Omitir la flecha en la consulta obliga al motor a buscar en ambas direcciones, duplicando el espacio de búsqueda. En grafos densos, use siempre direcciones explícitas para optimizar la latencia.

### 1.4. Propiedades en Relaciones

La relación debe tratarse como una entidad con contexto propio. Modelar atributos como la *intensidad de una amistad* o el *año de inicio en un cargo* permite filtrar el grafo antes de que el recorrido se vuelva computacionalmente inmanejable.

```cypher
MATCH (p:Persona)-[r:TRABAJA_EN]->(e:Empresa)
WHERE r.intensidad > 5 AND r.anio > 2020
RETURN p.nombre, e.nombre

```

> *La estructura del grafo establece los caminos posibles; la lógica de agregación determina la inteligencia extraída.*

---

## 2. Consultas Avanzadas y Lógica de Agregación

### 2.1. Diseño con Intención

El diseño de consultas avanzadas debe priorizar el **análisis estructural (patrones)** sobre la simple recuperación. Un arquitecto diseña para identificar anomalías, flujos y agrupaciones automáticas.

### 2.2. Filtrado y Proyección (`WHERE`, `DISTINCT`, `RETURN`)

* **`WHERE`:** Aplica predicados sobre propiedades una vez que el patrón estructural ha sido validado.
* **`DISTINCT`:** Indispensable en grafos. Debido a que múltiples caminos pueden llevar al mismo nodo, `DISTINCT` garantiza que el *pipeline* posterior no procese duplicados innecesarios.
* **Proyección:** Evite `RETURN n`. Proyecte solo las propiedades requeridas (ej. `RETURN n.id`) para reducir el *overhead* de serialización y la carga en memoria.

### 2.3. El Rol de `WITH` y `OPTIONAL MATCH`

* **`WITH` (El Puente):** Permite encadenar consultas.
* ⚠️ *Advertencia de Arquitecto:* Cualquier variable no incluida en el `WITH` se elimina del contexto actual. Es la herramienta clave para realizar agregaciones intermedias antes de continuar el recorrido.


* **`OPTIONAL MATCH` (Left Join):** Busca un patrón, pero si no lo encuentra, devuelve `null` en lugar de detener la ejecución de toda la consulta. Fundamental para enriquecer datos sin perder la entidad raíz.

### 2.4. Agregaciones y Group By Implícito

Cypher agrupa automáticamente por todas las variables no agregadas.

```cypher
MATCH (p:Persona)-[:PARTICIPA_EN]->(proj:Proyecto)
WITH p, count(proj) AS total_proyectos
WHERE total_proyectos > 2
RETURN p.nombre, total_proyectos

```

---

## 3. Análisis de Caminos (Paths) y Centralidad

### 3.1. Recorrido frente a Consulta Plana

Los *paths* (caminos) revelan la conectividad indirecta, permitiendo descubrir intermediarios y dependencias ocultas que no son visibles en registros aislados.

### 3.2. Longitud Variable y Explosión Combinatoria

La sintaxis de saltos variables (`*1..n`) es poderosa pero peligrosa.

* `*1..3`: Busca de 1 a 3 niveles de profundidad.
* `*..`: **Prohibido en producción.**

> 🔥 **Senior Tip:** Nunca use caminos sin límites (`*..`) en grafos densos. Una profundidad de solo 5 niveles en un grafo con alto grado de salida puede provocar un *memory crash* instantáneo. Siempre defina un límite superior estricto (ej. `*1..5`).

### 3.3. Variables de Path e Inspección

Capturar el camino completo permite auditar y trazar la lógica del recorrido:

```cypher
MATCH p = (a:Persona)-[:CONOCE*1..3]-(b:Persona)
RETURN nodes(p), relationships(p), length(p)

```

### 3.4. Identificación de Nodos Puente y Hubs

Para detectar **"Puentes Estructurales"** (nodos que conectan comunidades separadas), usamos `UNWIND` para descomponer caminos y contar la frecuencia de aparición de nodos intermedios. Esto representa una métrica de *Centralidad de Intermediación* simplificada:

```cypher
MATCH p = (:Persona)-[*1..3]-(:Persona)
UNWIND nodes(p) AS n
WITH n, count(*) AS frecuencia
RETURN n.nombre, frecuencia 
ORDER BY frecuencia DESC

```

---

## 4. Bases de Datos Vectoriales: Embeddings y Espacio Geométrico

### 4.1. Transición Semántica: Similitud por Proximidad

Las bases de datos vectoriales sustituyen la coincidencia exacta por la **proximidad geométrica**. Los datos se proyectan en un espacio N-dimensional donde la cercanía refleja similitud de significado.

### 4.2. Métricas de Similitud: Euclidiana vs. Coseno

* **Distancia Euclidiana (L2):** Mide la separación física lineal entre puntos. Es altamente sensible a la magnitud (longitud del texto).
* **Similitud del Coseno:** Mide el ángulo entre vectores (rango de -1 a 1). En Procesamiento de Lenguaje Natural (NLP) es preferible porque la dirección captura el significado semántico, ignorando la magnitud o la extensión del texto.

### 4.3. Dimensiones, Pesos e Interpretabilidad

* **Dimensiones:** En modelos reales como `BERT` o `all-MiniLM-L6-v2`, las dimensiones (384 o 768) son patrones estadísticas abstractos **no interpretables humanamente** (a diferencia de los modelos pedagógicos basados en variables fijas como "animal/vehículo").
* **Pesos:** Los valores negativos en dimensiones manuales representan tensiones semánticas (opuestos). En modelos de producción, la relevancia se gestiona mediante el ajuste dinámico de la *query*.

### 4.4. Inestabilidad y Ambigüedad

El espacio vectorial puede sufrir de "ruido" o "colapso", donde términos semánticamente distintos ocupan regiones geométricas similares.

* **Estrategia de Mitigación:** La desambiguación se logra añadiendo términos de contexto explícitos a la consulta (ej. *"banco financiero"* vs. *"banco de plaza"*) para desplazar el vector hacia la región correcta del espacio.

---

## 5. Implementación Técnica con FAISS y Transformers

### 5.1. Arquitectura del Pipeline

Un pipeline robusto desacopla la generación del embedding de la indexación:

$$\text{Texto} \longrightarrow \text{SentenceTransformer} \longrightarrow \text{Vector} \longrightarrow \text{FAISS} \longrightarrow \text{ID del Documento}$$

### 5.2. Uso de SentenceTransformers

Utilizamos modelos pre-entrenados para garantizar que el espacio vectorial posea relaciones lingüísticas e interconexiones coherentes desde el inicio.

```python
from sentence_transformers import SentenceTransformer

# Inicialización del modelo y codificación
model = SentenceTransformer('all-MiniLM-L6-v2')
vector = model.encode("Arquitectura de sistemas")  # Genera un vector float32

```

### 5.3. Indexación con FAISS (`IndexFlatL2`)

> ⚠️ **Advertencia de Arquitecto:** FAISS no almacena texto, solo vectores. El desarrollador es estrictamente responsable de mantener un mapeo externo (diccionario, caché o DB convencional) entre el índice de FAISS y el contenido original.

* `IndexFlatL2` realiza una búsqueda exacta optimizada en C++. No reduce la complejidad asintótica ($O(N)$), pero minimiza drásticamente la latencia mediante paralelismo a bajo nivel.

### 5.4. Búsqueda y Transformación de Score

Para que las distancias matemáticas sean interpretables por las reglas de negocio, se transforman en un **score de relevancia** normalizado:

```python
distancias, indices = index.search(query_vector, k)

# Transformación de distancia L2 (Euclidiana) a Score de similitud
score = 1 / (1 + distancias[0])

```

---

## 6. Modelado Estratégico y Resolución de Casos Reales

### 6.1. Modelar para la Consulta

El mayor error de arquitectura es replicar esquemas relacionales directamente en grafos. **Se debe modelar en función de la pregunta.** Si necesita conocer qué empleados participan en proyectos de empresas externas, esas relaciones deben elevarse a conexiones explícitas en el modelo.

### 6.2. Refactorización e Inconsistencias

Es posible detectar fallos de integridad operacional mediante el análisis de patrones estructurales del grafo:

```cypher
// Detectar inconsistencia: Personas en proyectos de empresas donde no trabajan
MATCH (p:Persona)-[:PARTICIPA_EN]->(proj:Proyecto)-[:GESTIONADO_POR]->(e:Empresa)
WHERE NOT (p)-[:TRABAJA_EN]->(e)
RETURN p.nombre, proj.nombre, e.nombre

```

### 6.3. Guía Rápida para el Examen (Snippets Críticos)

#### Cypher: Control de Flujo y Agregación

```cypher
// Búsqueda con salto variable y limpieza de resultados
MATCH (a:Persona {nombre: "Luis"})-[:CONOCE*1..2]-(b:Persona)
WITH b, count(*) AS fuerza_conexion
WHERE b.ciudad = "Madrid"
RETURN b.nombre, fuerza_conexion 
ORDER BY fuerza_conexion DESC

```

#### Python: Búsqueda Semántica con Umbral

```python
def buscar_filtrado(query, model, index, base_textos, umbral=0.4, k=5):
    # Generar y tipar el vector de consulta
    vq = model.encode([query]).astype('float32')
    distancias, indices = index.search(vq, k)
    
    resultados = []
    for d, idx in zip(distancias[0], indices[0]):
        score = 1 / (1 + d)  # Normalización
        if score >= umbral:
            resultados.append({
                "texto": base_textos[idx], 
                "score": score
            })
    return resultados

```

---

## 6.5. Cierre Final

La integridad de un sistema de recuperación híbrido reside en el equilibrio: **el grafo impone la verdad estructural** y las reglas duras de negocio, mientras que **el espacio vectorial aporta la flexibilidad semántica** necesaria para la interpretación del lenguaje humano. La excelencia técnica se alcanza cuando ambos sistemas convergen para transformar datos aislados en conocimiento accionable.
