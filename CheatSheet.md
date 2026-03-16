## 🍃 MongoDB Cheat Sheet: DB2 Essentials

### 1. The Basics (Navigation)

| Action | Command |
| --- | --- |
| **Show all databases** | `show dbs` |
| **Switch database** | `use <db_name>` |
| **Show collections** | `show collections` |
| **Check current DB** | `db` |

---

### 2. Create (Insert)

Avoid using the old `db.collection.insert()`. Use these specific methods for better error handling:

* **Single document:**
```javascript
db.users.insertOne({ name: "Lorenzo", role: "Instructor" })

```


* **Multiple documents:**
```javascript
db.users.insertMany([{ name: "Alice" }, { name: "Bob" }])

```



---

### 3. Read (Querying)

The core of your database interaction.

* **Find all:** `db.collection.find({})`
* **Filter by field:** `db.collection.find({ status: "active" })`
* **Format output:** `db.collection.find().pretty()`
* **Limit/Skip (Pagination):** ```javascript
db.collection.find().skip(10).limit(5)


#### Common Query Operators

| Operator | Description | Example |
| --- | --- | --- |
| `$gt` / `$gte` | Greater than (or equal) | `{ age: { $gt: 18 } }` |
| `$lt` / `$lte` | Less than (or equal) | `{ price: { $lte: 100 } }` |
| `$in` | Matches any value in array | `{ tags: { $in: ["db", "nosql"] } }` |
| `$or` | Matches any condition | `{ $or: [{ status: "A" }, { qty: 0 }] }` |
| `$exists` | Checks for field existence | `{ email: { $exists: true } }` |

---

### 4. Update

Remember: the first argument is the **filter**, the second is the **update operator**.

* **Update one field:**
```javascript
db.collection.updateOne({ _id: 1 }, { $set: { status: "Done" } })

```


* **Add to an array:**
```javascript
db.collection.updateOne({ _id: 1 }, { $push: { tags: "new-tag" } })

```


* **Increment a value:**
```javascript
db.collection.updateOne({ _id: 1 }, { $inc: { views: 1 } })

```



---

### 5. Aggregation Framework

In DB2, this is where most of your grade usually lives. Think of it as a factory assembly line.

```javascript
db.orders.aggregate([
  // Step 1: Filter
  { $match: { status: "A" } },
  
  // Step 2: Group by a field and calculate totals
  { $group: { _id: "$cust_id", total: { $sum: "$amount" } } },
  
  // Step 3: Sort results
  { $sort: { total: -1 } }
])

```

**Common Pipeline Stages:**

* `$match`: Filter documents (like `WHERE`).
* `$group`: Aggregate data (like `GROUP BY`).
* `$project`: Reshape documents (include/exclude/rename fields).
* `$unwind`: Deconstructs an array field into multiple documents.
* `$lookup`: Performs a "join" with another collection.

---

### 6. Indexes (Performance)

Essential for large datasets in a "Base de Datos 2" context.

* **Create Index:** `db.collection.createIndex({ email: 1 })` (1 for Ascending, -1 for Descending).
* **Unique Index:** `db.collection.createIndex({ username: 1 }, { unique: true })`
* **View Indexes:** `db.collection.getIndexes()`
* **Analyze Performance:** Append `.explain("executionStats")` to your query.

---

### 7. Delete

* **Delete one:** `db.collection.deleteOne({ _id: 1 })`
* **Delete many:** `db.collection.deleteMany({ status: "expired" })`

> **Note:** Always run your filter in a `.find()` before running a `.deleteMany()` to avoid accidental "mass extinctions" of your data.
