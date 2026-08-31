# ⚡ ExecuteUpdate & ExecuteDelete — EF Core Toolbox

**File:** `docs/06-executeupdate-executedelete.md`

---

## 🎯 Core Concept

EF Core provides **two approaches** for modifying database data:

```text
Approach 1 → Load + Track + Modify + SaveChanges()

Approach 2 → Execute SQL Directly

             ┌──────────────────────┐
             │ ExecuteUpdate        │
             │ ExecuteDelete        │
             └──────────┬───────────┘
                        │
                        ▼
                 Direct Database SQL

          ❌ No Entity Loading
          ❌ No Change Tracking
          ❌ No SaveChanges()
          ⚡ Immediate Execution
```

### 🧠 Mental Model

```text
SaveChanges Approach

Database
   │
   ▼
Load Entities
   │
   ▼
Change Tracker
   │
   ▼
Modify Entities
   │
   ▼
SaveChanges()
   │
   ▼
SQL


ExecuteUpdate / ExecuteDelete

LINQ Query
   │
   ▼
Direct SQL
   │
   ▼
Database

⚡ No entity objects involved
```

---

# 🗑️ ExecuteDelete

## 🔎 What Is ExecuteDelete?

`ExecuteDelete()` / `ExecuteDeleteAsync()` deletes matching rows **directly in the database**.

It does **not**:

- Load entities into memory.
- Track entities.
- Use the Change Tracker.
- Require `SaveChanges()`.

---

## 🐢 Traditional Delete

```csharp
await foreach (var blog in context.Blogs
    .Where(b => b.Rating < 3)
    .AsAsyncEnumerable())
{
    context.Blogs.Remove(blog);
}

await context.SaveChangesAsync();
```

### Flow

```text
Database
   │
   ▼
Load Matching Blogs
   │
   ▼
Track Entities
   │
   ▼
Mark As Deleted
   │
   ▼
SaveChanges()
   │
   ▼
DELETE Statements
```

---

## ⚡ ExecuteDelete

```csharp
await context.Blogs
    .Where(b => b.Rating < 3)
    .ExecuteDeleteAsync();
```

### Generated SQL Conceptually

```sql
DELETE FROM [Blogs]
WHERE [Rating] < 3;
```

### Flow

```text
LINQ Filter
    │
    ▼
EF Core SQL Translation
    │
    ▼
Single DELETE Statement
    │
    ▼
Database
```

---

## 🆚 Traditional Delete vs ExecuteDelete

| Aspect | SaveChanges() | ExecuteDelete |
|---|---|---|
| Entities loaded? | ✅ Yes | ❌ No |
| Change tracker? | ✅ Yes | ❌ No |
| SaveChanges required? | ✅ Yes | ❌ No |
| Execution | Deferred until SaveChanges | Immediate |
| SQL | Entity-based deletes | Direct set-based DELETE |
| Large datasets | 🐢 Slower | ⚡ Efficient |

> 💡 **Use ExecuteDelete when you want to delete many rows directly and don't need entity-level tracking.**

---

# ✏️ ExecuteUpdate

## 🔎 What Is ExecuteUpdate?

`ExecuteUpdate()` / `ExecuteUpdateAsync()` updates matching rows directly in the database.

It:

- Uses LINQ for filtering.
- Uses `SetProperty()` to define updates.
- Generates SQL `UPDATE`.
- Executes immediately.

It does **not**:

- Load entities.
- Track entities.
- Require `SaveChanges()`.

---

## 🐢 Traditional Update

```csharp
await foreach (var blog in context.Blogs
    .Where(b => b.Rating < 3)
    .AsAsyncEnumerable())
{
    blog.IsVisible = false;
}

await context.SaveChangesAsync();
```

### Flow

```text
Query
  │
  ▼
Load Entities
  │
  ▼
Track Entities
  │
  ▼
Modify Properties
  │
  ▼
SaveChanges()
  │
  ▼
UPDATE Database
```

---

## ⚡ ExecuteUpdate

```csharp
await context.Blogs
    .Where(b => b.Rating < 3)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(b => b.IsVisible, false));
```

### Generated SQL Conceptually

```sql
UPDATE [Blogs]
SET [IsVisible] = 0
WHERE [Rating] < 3;
```

---

## 🔄 Updating Multiple Properties

```csharp
await context.Blogs
    .Where(b => b.Rating < 3)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(b => b.IsVisible, false)
        .SetProperty(b => b.Rating, 0));
```

Conceptually:

```sql
UPDATE [Blogs]
SET [IsVisible] = 0,
    [Rating] = 0
WHERE [Rating] < 3;
```

> 💡 Multiple properties can be updated in **one SQL UPDATE statement**.

---

## ➕ Using Existing Values

You can calculate the new value from the existing database value.

```csharp
await context.Blogs
    .Where(b => b.Rating < 3)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(
            b => b.Rating,
            b => b.Rating + 1));
```

Conceptually:

```sql
UPDATE [Blogs]
SET [Rating] = [Rating] + 1
WHERE [Rating] < 3;
```

---

# 🔗 Navigations & Related Entities

## ❌ Direct Navigation Usage

You generally cannot directly use navigation expressions like this:

```csharp
.SetProperty(
    b => b.Rating,
    b => b.Posts.Average(p => p.Rating))
```

### Why?

```text
Navigation Property
       │
       ▼
Related Table
       │
       ▼
Requires JOIN / Subquery
       │
       ▼
Cannot be used directly as a simple property setter
```

---

## ✅ Workaround — Projection

Pre-compute the value using `Select()`.

```csharp
await context.Blogs
    .Select(b => new
    {
        Blog = b,
        NewRating = b.Posts.Average(p => p.Rating)
    })
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(
            b => b.Blog.Rating,
            b => b.NewRating));
```

### 🧠 Mental Model

```text
Blog
 │
 ├── Calculate Posts Average
 │
 ▼
Anonymous Projection
 │
 ├── Blog
 └── NewRating
        │
        ▼
ExecuteUpdate
        │
        ▼
UPDATE Blog.Rating
```

EF Core can translate this into a SQL subquery.

---

## 📌 Important: `DbSet<T>` Is Queryable

```text
context.Blogs
      │
      ▼
DbSet<Blog>
      │
      ▼
IQueryable<Blog>
      │
      ▼
Expression Tree
      │
      ▼
SQL Translation
```

When writing:

```csharp
context.Blogs
    .Select(...)
    .ExecuteUpdateAsync(...);
```

the anonymous object is **not necessarily created in memory**.

It is part of the **query expression** that EF Core translates to SQL.

> 💡 `Select()` can be used to shape a database query before `ExecuteUpdate`.

---

# 🧠 Change Tracking — Important!

## Traditional `SaveChanges()`

```text
Entity Loaded
     │
     ▼
Tracked by DbContext
     │
     ▼
Modify Property
     │
     ▼
SaveChanges()
     │
     ▼
EF Detects Changes
     │
     ▼
SQL UPDATE / DELETE
```

EF Core knows:

- Original values.
- Current values.
- Entity states.

---

## ExecuteUpdate / ExecuteDelete

```text
LINQ Query
    │
    ▼
Execute Immediately
    │
    ▼
Direct SQL
    │
    ▼
Database

❌ Change Tracker Bypassed
```

Each call executes independently.

---

## ⚠️ Mixing Bulk Operations with Tracked Entities

```csharp
var blog = await context.Blogs
    .SingleAsync(b => b.Name == "SomeBlog");

// Tracked entity

await context.Blogs.ExecuteUpdateAsync(setters =>
    setters.SetProperty(
        b => b.Rating,
        b => b.Rating + 1));

// Database updated immediately

blog.Rating += 2;

await context.SaveChangesAsync();
```

### 🧠 What Happens?

```text
Initial Rating

Database = 5
Tracked Entity = 5

        │
        ▼

ExecuteUpdate()

Database = 6
Tracked Entity = 5  ❗ Still unchanged

        │
        ▼

blog.Rating += 2

Tracked Entity = 7

        │
        ▼

SaveChanges()

Database = 7
```

⚠️ The bulk update result can be **overwritten by stale tracked data**.

### Rule

> 🚨 Be careful when mixing `ExecuteUpdate` / `ExecuteDelete` with tracked entities in the same `DbContext`.

---

# 🔄 Transactions

## Default Behavior

Each `ExecuteUpdate` or `ExecuteDelete` executes as an independent database command.

```csharp
await context.Blogs.ExecuteUpdateAsync(...);

await context.Blogs.ExecuteUpdateAsync(...);

await context.Blogs.ExecuteDeleteAsync(...);
```

If you need multiple operations to succeed or fail together, use an explicit transaction.

---

## ✅ Explicit Transaction

```csharp
await using var transaction =
    await context.Database.BeginTransactionAsync();

await context.Blogs.ExecuteUpdateAsync(...);

await context.Blogs.ExecuteUpdateAsync(...);

// Other operations

await transaction.CommitAsync();
```

### Mental Model

```text
BEGIN TRANSACTION
        │
        ├── ExecuteUpdate
        │
        ├── ExecuteUpdate
        │
        ├── ExecuteDelete
        │
        ▼
COMMIT

OR

        ▼
ROLLBACK
```

> 💡 Explicit transactions provide **atomicity** for multiple operations.

---

# 🔐 Concurrency Control

## Traditional `SaveChanges()`

Tracked entities can use concurrency tokens.

```text
Load Entity
    │
    ▼
Original Concurrency Token
    │
    ▼
Modify Entity
    │
    ▼
SaveChanges()
    │
    ▼
Token Checked
```

If another user changed the row:

```text
Token Mismatch
      │
      ▼
DbUpdateConcurrencyException
```

---

## ExecuteUpdate / ExecuteDelete

Because these APIs bypass the Change Tracker:

❌ No automatic concurrency handling.

Instead, they return the **number of affected rows**.

---

## ✅ Manual Concurrency Pattern

```csharp
var rowsAffected = await context.Blogs
    .Where(b =>
        b.Id == id &&
        b.ConcurrencyToken == concurrencyToken)
    .ExecuteUpdateAsync(setters =>
        setters.SetProperty(
            b => b.Title,
            "Updated Title"));

if (rowsAffected == 0)
{
    throw new Exception(
        "Update failed due to concurrency conflict.");
}
```

### Flow

```text
WHERE

Id = Requested Id
AND
ConcurrencyToken = Expected Token

        │
        ▼

Rows Updated?

   ┌───────────────┐
   │               │
   ▼               ▼

Rows > 0       Rows = 0
Success        Concurrency Conflict
```

> 💡 With bulk operations, **affected row count becomes your manual concurrency check**.

---

# ⚖️ SaveChanges vs ExecuteUpdate / ExecuteDelete

| Feature | SaveChanges | ExecuteUpdate / ExecuteDelete |
|---|---|---|
| Entity loading | Usually required | ❌ Not required |
| Change tracking | ✅ Yes | ❌ No |
| SaveChanges required | ✅ Yes | ❌ No |
| Immediate execution | ❌ No | ✅ Yes |
| Multiple changes accumulated | ✅ Yes | ❌ No |
| Automatic concurrency | ✅ Supported | ❌ Manual |
| Large data operations | 🐢 Can be expensive | ⚡ Efficient |
| Tracked entity state updated | ✅ Yes | ❌ No |

---

# ⚠️ Important Limitations

## 1️⃣ Only Update & Delete

```text
ExecuteUpdate  → UPDATE
ExecuteDelete  → DELETE

No ExecuteInsert
```

For inserts:

```csharp
context.Blogs.Add(blog);

await context.SaveChangesAsync();
```

---

## 2️⃣ No Change Tracker Synchronization

After a bulk operation:

```text
Database = Updated

Tracked Entity = Old Value ❗
```

EF Core does not automatically refresh tracked entities.

---

## 3️⃣ Each Call Is a Separate Database Roundtrip

```text
ExecuteUpdate()
        │
        ▼
Roundtrip 1

ExecuteUpdate()
        │
        ▼
Roundtrip 2
```

Multiple bulk calls are not automatically combined into one SQL command.

---

## 4️⃣ Single-Table Target

Each `ExecuteUpdate` / `ExecuteDelete` targets the mapped table for the entity operation.

You cannot use one call to directly update multiple tables.

---

## 5️⃣ Relational Providers

These APIs are designed for relational database providers such as:

- SQL Server
- PostgreSQL
- SQLite
- Other relational EF Core providers

They are not supported in the same way by non-relational providers such as Cosmos DB.

---

# ⭐ Key Rules to Remember

```text
1. ExecuteUpdate / ExecuteDelete bypass Change Tracker.

2. No SaveChanges() is required.

3. Operations execute immediately.

4. No entities are loaded into memory.

5. Excellent for large set-based UPDATE / DELETE operations.

6. Tracked entities are NOT automatically synchronized.

7. Mixing bulk operations and tracked entities can overwrite data.

8. Automatic concurrency handling is bypassed.

9. Use affected row count for manual concurrency checks.

10. Use explicit transactions when multiple operations must be atomic.

11. ExecuteUpdate uses SetProperty().

12. ExecuteDelete / ExecuteUpdate are relational database operations.
```

---

# ⚡ 30-Second Revision

```text
ExecuteUpdate / ExecuteDelete

        │
        ▼

Bulk Database Operations

        │
        ├── No Entity Loading
        ├── No Change Tracking
        ├── No SaveChanges()
        ├── Immediate SQL Execution
        └── Efficient for Large Datasets


ExecuteUpdate
    │
    └── UPDATE rows using SetProperty()

ExecuteDelete
    │
    └── DELETE matching rows


⚠️ Remember:

Database changes immediately
BUT
Tracked entities remain stale

→ Be careful mixing bulk operations + SaveChanges()
```

---

> 🎯 **Interview Mental Model:**  
> **`SaveChanges()` is entity-centric and tracking-based.**  
> **`ExecuteUpdate()` and `ExecuteDelete()` are database-centric, set-based, immediate bulk operations.**