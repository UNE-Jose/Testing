# 1. Neo4j & Cypher: Patrones y Consultas

En Cypher, no se consultan filas, se **describen formas y patrones** dentro del grafo.

### Estructura Básica y Creación
*   **CREATE**: Crea nodos o relaciones. Si se usa varias veces con los mismos datos, genera duplicados.
    *   `(n:Persona {nombre: "Ana", edad: 25})`
*   **MERGE**: Actúa como un "get or create". Separa la identidad de las propiedades para evitar duplicados.
    *   `MERGE (p:Persona {nombre: "Luis"}) ON CREATE SET p.creado = timestamp()`
*   **Propiedades en relaciones**: Las relaciones pueden (y a menudo deben) tener información como intensidad o año.
    *   `(a)-[r:AMIGO_DE {intensidad: 5}]->(b)`

### Direccionalidad y Recorridos
*   **Dirección estricta**: `(a)-[:REL]->(b)` o `(a)<-[:REL]-(b)`.
*   **Ignorar dirección**: `(a)-[:REL]-(b)` (útil para relaciones simétricas como amistad).
*   **Longitud variable (Paths)**:
    *   ``: Exactamente 2 saltos.
    *   `[*1..3]`: Entre 1 y 3 saltos.
    *   `[*]`: Longitud indefinida (¡Cuidado! Puede causar explosión combinatoria).

### Lógica Intermedia y Agregación
*   **WITH**: Palabra clave fundamental para dividir la consulta en etapas. Define qué variables "sobreviven" al siguiente paso.
    *   `MATCH (a)-->(b) WITH a, count(b) AS amigos WHERE amigos > 1 ...`
*   **OPTIONAL MATCH**: Equivalente a un `LEFT JOIN`. Si no encuentra el patrón, devuelve `null` en lugar de descartar la fila completa.
*   **DISTINCT**: Elimina duplicados en los resultados, esencial cuando los paths generan múltiples caminos al mismo nodo.
*   **Agregación**: Se agrupa automáticamente por las variables no agregadas (Group By implícito).
    *   `RETURN p.nombre, count(amigos) AS total`

### Análisis de Grafos (Consultas de "Negocio")
*   **Nodos Puente**: Identificar nodos que aparecen frecuentemente en caminos entre otros.
    *   `MATCH p=(a)-[*1..3]-(b) UNWIND nodes(p) AS n WITH n, count(*) AS apariciones RETURN n ORDER BY apariciones DESC`
*   **Centralidad**: Medir la importancia por el número de conexiones.
*   **Inconsistencias**: Buscar patrones que no deberían existir (ej. trabajar en una empresa pero participar en proyectos de otra).

---

# 2. Bases de Datos Vectoriales & FAISS (Python)

El paso fundamental es de la coincidencia exacta (SQL/Grafos) a la **proximidad semántica**.

### Conceptos de Embeddings
*   **Embedding**: Función que convierte texto en un vector numérico (lista de números).
*   **Dirección vs. Tamaño**: La dirección representa el significado; el tamaño (magnitud) puede ser irrelevante. La **similitud del coseno** ignora la magnitud.
*   **Dimensiones**: Cada dimensión captura matices. En sistemas reales se usan cientos (384, 768), aunque en clase se simularon pocas para entender el concepto.
*   **Contexto**: El embedding cambia según la frase completa. Pequeñas variaciones en la query desplazan el vector en el espacio.

### Implementación con FAISS
FAISS no entiende texto, solo **almacena y busca vectores rápidamente** mediante cálculos optimizados en C++.

1.  **Modelo**: Se usa `SentenceTransformer` (ej: `all-MiniLM-L6-v2`) para generar embeddings reales.
2.  **Indexación**:
    *   `IndexFlatL2`: Índice de búsqueda exacta (distancia euclidiana). No aproxima, compara contra todos pero de forma veloz.
    *   `index.add(vectors)`: Inserta los vectores en el índice.
3.  **Búsqueda**:
    *   `distances, indices = index.search(query_vector, k)`
    *   **Distancia**: En `IndexFlatL2`, **menor distancia = más similar**.

### Refinamiento del Sistema
*   **Score Semántico**: Transformación para que el resultado sea intuitivo (0 a 1).
    *   `score = 1 / (1 + distancia)`.
    *   Aquí, **mayor score = más similar**.
*   **Umbral (Threshold)**: Filtrado para eliminar resultados poco relevantes.
    *   `if score > umbral: return resultado`
*   **K (Top-K)**: Número de vecinos a recuperar. Un K mayor aumenta la cantidad pero puede bajar la precisión.

---

# 3. Tips para el Examen (Estrategia Operativa)

*   **No memorizar la arquitectura**: El sistema base (imports, clase `BuscadorSemantico`, etc.) será entregado.
*   **Análisis del Ranking**: Si el sistema devuelve resultados inesperados, analiza cómo la **ambigüedad** o los **pesos** del embedding influyen en la posición del vector.
*   **Pruebas en papel**: Aunque se permite usar VSCode/Jupyter para debugging, la respuesta final debe ser **escrita manualmente**.
*   **Variables en Cypher**: Un error común es perder variables en un `WITH` o confundir una propiedad con una variable (ej. usar `nombre` en lugar de `p.nombre`).
*   **Caminos (Paths)**: Para encontrar conexiones indirectas, usa siempre una variable de path `p = (a)-[*..]->(b)` y luego analiza su longitud con `length(p)`.
