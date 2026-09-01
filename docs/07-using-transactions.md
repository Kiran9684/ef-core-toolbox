# 🔄 Using Transactions — EF Core Toolbox

**File:** `docs/07-transactions.md`

---

# 🎯 Core Concept

A **transaction** groups multiple database operations into a single **atomic unit of work**.

### 🧠 Mental Model

```text
Multiple Database Operations
            │
            ▼
       TRANSACTION
            │
     ┌──────┴──────┐
     ▼             ▼
  COMMIT        ROLLBACK
     │             │
     ▼             ▼
Everything      Nothing
is saved        is saved
```

### ⭐ Atomicity

```text
ALL succeed  → COMMIT → All changes are applied
ANY fails    → ROLLBACK → All changes are undone
```

> 💡 **Transaction = Everything succeeds together, or everything fails together.**

---

# 🟦 Default Transaction Behavior

For relational database providers that support transactions:

```text
SaveChanges()
      │
      ▼
BEGIN TRANSACTION
      │
      ▼
Execute INSERT / UPDATE / DELETE
      │
      ├── Success → COMMIT
      │
      └── Failure → ROLLBACK
```

### Important Rules

- A **single `SaveChanges()` call** is transactional by default.
- If one database operation fails, the transaction is rolled back.
- The database is not left partially updated.
- For most applications, this default behavior is sufficient.

> ⭐ **You usually do NOT need to manually create a transaction for one `SaveChanges()` call.**

---

# 🤔 When Do We Need Manual Transactions?

Manual transactions are useful when **multiple operations must behave as one unit**.

For example:

```text
SaveChanges() #1
      +
SaveChanges() #2
      +
Other Database Operations

        │
        ▼

One Transaction

        │
   ┌────┴────┐
   ▼         ▼
Success     Failure
COMMIT      ROLLBACK
```

Without a manual transaction:

```text
SaveChanges #1 → ✅ Committed

SaveChanges #2 → ❌ Failed

Result → Partial data remains ❗
```

With a manual transaction:

```text
SaveChanges #1 → Pending

SaveChanges #2 → Failed

Result → Everything rolled back ✅
```

---

# 🛠️ Manual Transaction

EF Core provides transaction APIs through:

```csharp
context.Database
```

### Example

```csharp
await using var transaction =
    await context.Database.BeginTransactionAsync();

try
{
    context.Blogs.Add(new Blog
    {
        Url = "http://blogs.msdn.com/dotnet"
    });

    await context.SaveChangesAsync();

    context.Blogs.Add(new Blog
    {
        Url = "http://blogs.msdn.com/visualstudio"
    });

    await context.SaveChangesAsync();

    var blogs = await context.Blogs
        .OrderBy(b => b.Url)
        .ToListAsync();

    await transaction.CommitAsync();
}
catch
{
    // Transaction rolls back if not committed
}
```

---

## 🔄 Flow

```text
BeginTransactionAsync()
          │
          ▼
     Operation 1
          │
          ▼
    SaveChanges()
          │
          ▼
     Operation 2
          │
          ▼
    SaveChanges()
          │
          ▼
       Query
          │
     ┌────┴────┐
     ▼         ▼
Success      Failure
   │            │
   ▼            ▼
Commit      Rollback
```

### What Happens?

1. `BeginTransactionAsync()` starts the transaction.
2. Multiple database operations execute inside it.
3. If everything succeeds → `CommitAsync()`.
4. If something fails before commit → changes are rolled back.

---

# ⚠️ Important: Commit Is the Final Decision

```text
BEGIN TRANSACTION

Operation 1 → Successful
Operation 2 → Successful
Operation 3 → Successful

        │
        ▼

COMMIT

        │
        ▼

Changes become permanent
```

Until the transaction is committed:

```text
Changes are still part of the transaction.

No Commit
    │
    ▼
Rollback / Disposal
    │
    ▼
Changes are undone
```

---

# 🗄️ Provider Support

### Relational Providers

Generally support transactions:

```text
SQL Server
PostgreSQL
SQLite
MySQL
```

### Non-Relational Providers

May not support EF Core transaction APIs in the same way.

For example:

```text
Cosmos DB
```

may throw or behave differently depending on the provider and operation.

> ⭐ Transactions are primarily a **relational database concept** in EF Core.

---

# 🔁 Transactions & Execution Strategies

EF Core may use **execution strategies** for connection resiliency.

Example scenarios:

```text
Temporary network failure
        │
        ▼
EF Core retries operation
```

### ⚠️ Important Rule

Manual transactions can conflict with retrying execution strategies.

Why?

```text
Retry Strategy

Operation
   │
   ▼
Failure
   │
   ▼
Retry Entire Unit of Work
```

But with a manually controlled transaction:

```text
EF Core does not automatically control
the complete transaction boundary.
```

Therefore, EF Core cannot safely replay the entire unit of work unless the execution strategy itself is used to execute that unit appropriately.

### 🧠 Revision Rule

> ⚠️ **Don't casually mix manually managed transactions with retrying execution strategies.**

For simple scenarios, let EF Core manage the transaction automatically.

---

# 📍 Savepoints

## 🔎 What Is a Savepoint?

A **savepoint** is a marker inside an existing transaction.

### Mental Model

```text
BEGIN TRANSACTION
        │
        ▼
Operation 1
        │
        ▼
SAVEPOINT 📍
        │
        ▼
Operation 2
        │
        ▼
Operation 3 ❌
        │
        ▼
ROLLBACK TO SAVEPOINT
        │
        ▼
Operation 1 remains ✅
Operation 2 & 3 undone
```

Unlike rolling back the entire transaction:

```text
ROLLBACK TRANSACTION

❌ Operation 1
❌ Operation 2
❌ Operation 3
```

A savepoint allows:

```text
Rollback only a portion
of the transaction.
```

---

# 🤖 EF Core Automatic Savepoints

If a transaction is already active:

```text
BEGIN TRANSACTION
        │
        ▼
SaveChanges()
```

EF Core can automatically create a savepoint before `SaveChanges()`.

### If `SaveChanges()` Fails

```text
Transaction
     │
     ▼
Savepoint Created 📍
     │
     ▼
SaveChanges()
     │
     ▼
Failure ❌
     │
     ▼
Rollback to Savepoint
```

Result:

```text
Transaction remains usable
as if that SaveChanges()
operation had not started.
```

This is particularly useful for:

- Optimistic concurrency conflicts.
- Retrying a failed operation without losing the entire transaction.

---

# ⚠️ Savepoints & SQL Server MARS

Savepoints are incompatible with SQL Server:

```text
Multiple Active Result Sets (MARS)
```

When MARS is enabled:

```text
EF Core does NOT create savepoints.
```

If an error occurs:

```text
Transaction State
      │
      ▼
May become unknown / unusable ❗
```

> 🚨 **Important:** Be aware of MARS when relying on automatic savepoint behavior in SQL Server.

---

# 🛠️ Manual Savepoints

You can create your own savepoint.

```csharp
await using var transaction =
    await context.Database.BeginTransactionAsync();

try
{
    // First operation

    context.Blogs.Add(new Blog
    {
        Url = "https://devblogs.microsoft.com/dotnet/"
    });

    await context.SaveChangesAsync();


    // Create savepoint

    await transaction.CreateSavepointAsync(
        "BeforeMoreBlogs");


    // More operations

    context.Blogs.Add(new Blog
    {
        Url = "https://devblogs.microsoft.com/visualstudio/"
    });

    context.Blogs.Add(new Blog
    {
        Url = "https://devblogs.microsoft.com/aspnet/"
    });

    await context.SaveChangesAsync();


    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackToSavepointAsync(
        "BeforeMoreBlogs");
}
```

---

## 🧠 Manual Savepoint Flow

```text
BEGIN TRANSACTION
        │
        ▼
Insert Blog A
        │
        ▼
SaveChanges() ✅
        │
        ▼
SAVEPOINT 📍
"BeforeMoreBlogs"
        │
        ▼
Insert Blog B
        │
        ▼
Insert Blog C
        │
        ▼
SaveChanges()

     ┌──┴──┐
     ▼     ▼

Success   Failure
   │         │
   ▼         ▼

COMMIT   RollbackToSavepoint()
            │
            ▼
        Blog A remains ✅
        Blog B & C undone ❌
```

---

# 🆚 Transaction vs Savepoint

| Feature | Transaction | Savepoint |
|---|---|---|
| Scope | Entire unit of work | Part of a transaction |
| Rollback | Everything | Only after savepoint |
| Commit | Makes entire transaction permanent | Does not commit transaction |
| Purpose | Atomicity | Partial recovery |

### 🧠 Simple Model

```text
Transaction = Entire Journey 🚗

Savepoint = Checkpoint 📍

Rollback Transaction
→ Go back to the beginning

Rollback to Savepoint
→ Go back to the checkpoint
```

---

# ⭐ Key APIs

### Begin Transaction

```csharp
await context.Database.BeginTransactionAsync();
```

### Commit

```csharp
await transaction.CommitAsync();
```

### Create Savepoint

```csharp
await transaction.CreateSavepointAsync("MySavepoint");
```

### Rollback to Savepoint

```csharp
await transaction.RollbackToSavepointAsync("MySavepoint");
```

### Full Rollback

```csharp
await transaction.RollbackAsync();
```

---

# 🚫 Advanced Transaction Topics

These topics are separate and can be studied later:

```text
Using Transactions
│
├── Basic Transactions          ✅
│
├── Savepoints                  ✅
│
├── Cross-Context Transactions  ⬜ Later
│
├── External DbTransactions     ⬜ Later
│
└── System.Transactions         ⬜ Later
```

---

# ⚖️ Default vs Manual Transactions

| Scenario | Recommended Approach |
|---|---|
| One `SaveChanges()` | Default EF Core transaction |
| Multiple `SaveChanges()` as one unit | Manual transaction |
| Multiple EF/database operations must succeed together | Manual transaction |
| Need partial rollback | Savepoints |
| Simple CRUD | Default behavior |

---

# ⭐ Key Rules to Remember

```text
1. Transaction = Atomic Unit of Work.

2. COMMIT → All operations succeed.

3. ROLLBACK → All operations are undone.

4. A single SaveChanges() is transactional by default
   for providers that support transactions.

5. Manual transactions are mainly needed when
   multiple operations must behave as one unit.

6. BeginTransactionAsync() starts a manual transaction.

7. CommitAsync() makes all changes permanent.

8. Savepoints allow partial rollback.

9. EF Core can automatically create savepoints
   before SaveChanges() inside an active transaction.

10. SQL Server MARS is incompatible with EF Core savepoints.

11. Be careful mixing manual transactions with
    retrying execution strategies.

12. Transactions are primarily supported by
    relational database providers.
```

---

# ⚡ 30-Second Revision

```text
TRANSACTION

Multiple Operations
        │
        ▼
One Atomic Unit
        │
   ┌────┴────┐
   ▼         ▼
COMMIT     ROLLBACK
   │         │
   ▼         ▼
ALL saved  NOTHING saved


DEFAULT:

SaveChanges()
     │
     ▼
Transaction Automatically


MANUAL:

BeginTransaction()
     │
     ├── SaveChanges()
     ├── SaveChanges()
     └── Other Operations
            │
            ▼
          Commit()


SAVEPOINT:

Transaction
     │
     ▼
Operation A
     │
     ▼
📍 Savepoint
     │
     ▼
Operation B ❌
     │
     ▼
Rollback to Savepoint

A remains ✅
B is undone ❌
```

---

> 🎯 **Interview Mental Model:**  
> **`SaveChanges()` gives you automatic transaction safety for a single save operation. Manual transactions are used when multiple database operations must succeed or fail together. Savepoints provide partial rollback inside a transaction.**