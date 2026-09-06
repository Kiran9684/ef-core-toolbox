# 🧰 EF Core Interview Toolbox — Topic 13: Keyless Entity Types

> **Category:** 🏗️ Creating Model  
> **File Name:** `13-keyless-entity-types.md`

---

# 🔎 Keyless Entity Types

- A **keyless entity type** is an EF Core entity type without a primary key.
- It is mainly used to **query data that has no key**.
- Keyless entities are effectively **read-only projections/views of data**.
- Common use cases:
  - Database views
  - Raw SQL query results
  - Tables without primary keys
  - Reporting/query models
  - Denormalized data

```text
Database Object / Query
        ↓
Keyless Entity Type
        ↓
Read / Query Data

❌ No Primary Key
❌ No Normal Change Tracking
❌ No Insert / Update / Delete
```

---

# 🏷️ Defining a Keyless Entity

### Data Annotation

```csharp
[Keyless]
public class BlogPostsCount
{
    public string BlogName { get; set; }
    public int PostCount { get; set; }
}
```

### Fluent API

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<BlogPostsCount>()
        .HasNoKey();
}
```

> ⭐ Keyless entity types must be explicitly configured as keyless using `[Keyless]` or `.HasNoKey()`.

---

# 🧠 Key Characteristics

| Feature | Keyless Entity Type |
|---|---|
| Primary Key | ❌ None |
| Change Tracking | ❌ Not tracked for normal updates |
| INSERT / UPDATE / DELETE | ❌ Not supported |
| Convention Discovery | ❌ Not discovered as keyless automatically |
| Navigation | Limited |
| Owned-Type Navigation | ❌ Not allowed |
| Query Mapping | ✅ Views / SQL / Defining Queries |
| Inheritance | ✅ TPH only |
| Table Splitting | ❌ Not supported |

> ⭐ Think of a keyless entity as a **query-only model**, not a normal updateable entity.

---

# 🗄️ Keyless Entity + Migrations

- Keyless entities are generally treated as **query models**, not normal schema-owning entities.
- They are not included in migrations for table creation unless explicitly mapped to a database object such as a table or view.

```text
Keyless Entity
      ↓
Query Model
      ↓
Database Object already exists
      ↓
Read Data
```

---

# 📊 Common Usage Scenarios

## 1️⃣ Raw SQL Query Results

A keyless entity can represent the result of raw SQL.

```csharp
[Keyless]
public class SalesReport
{
    public string ProductName { get; set; }
    public int TotalSold { get; set; }
}
```

```csharp
var report = await context.Set<SalesReport>()
    .FromSqlRaw("""
        SELECT ProductName,
               SUM(Quantity) AS TotalSold
        FROM Orders
        GROUP BY ProductName
        """)
    .ToListAsync();
```

```text
Raw SQL
   ↓
Query Result
   ↓
Keyless Entity
   ↓
Read Results
```

---

## 2️⃣ Database Views

Views commonly have no primary key, making keyless entities a natural fit.

```csharp
[Keyless]
public class CustomerOrderSummary
{
    public string CustomerName { get; set; }
    public int OrdersCount { get; set; }
}
```

```csharp
modelBuilder.Entity<CustomerOrderSummary>()
    .ToView("View_CustomerOrderSummary");
```

> ⭐ EF Core assumes the referenced view already exists; it does not create the view automatically through a migration.

---

## 3️⃣ Tables Without a Primary Key

A keyless entity can also map to a table that has no primary key.

```csharp
[Keyless]
public class LegacyAuditLog
{
    public DateTime LogDate { get; set; }
    public string Action { get; set; }
    public string UserName { get; set; }
}
```

```csharp
modelBuilder.Entity<LegacyAuditLog>()
    .ToTable("AuditLog");
```

Because it is keyless:

```text
AuditLog Table
      ↓
Keyless Entity
      ↓
Query Only
```

---

## 4️⃣ Defining Queries

A keyless entity can be mapped to a LINQ query defined in the model.

```csharp
[Keyless]
public class ActiveCustomer
{
    public string Name { get; set; }
    public DateTime LastOrderDate { get; set; }
}
```

```csharp
modelBuilder.Entity<ActiveCustomer>()
    .HasNoKey()
    .ToQuery(() =>
        from c in Customers
        where c.IsActive
        select new ActiveCustomer
        {
            Name = c.Name,
            LastOrderDate =
                c.Orders.Max(o => o.OrderDate)
        });
```

```text
Defining LINQ Query
        ↓
Keyless Entity
        ↓
Read Results
```

---

# 🆚 `ToView()` vs `ToTable()` with Keyless Entities

### Keyless + `ToView()`

```csharp
modelBuilder.Entity<BlogPostsCount>()
    .HasNoKey()
    .ToView("View_BlogPostCounts");
```

```text
Entity
  ↓
View
  ↓
Read Only
```

### Keyless + `ToTable()`

```csharp
modelBuilder.Entity<LegacyAuditLog>()
    .HasNoKey()
    .ToTable("AuditLog");
```

```text
Entity
  ↓
Table
  ↓
Still Read Only
```

> ⭐ **The keyless configuration is what makes the entity query-only.** Mapping it with `ToTable()` does not make it updateable.

---

# 🔗 Navigation Limitations

Keyless entity types have **limited navigation support**.

```text
Keyless Entity
      │
      └── Can reference regular entity types

Regular Entity
      ✖
      │
      └── Cannot reference keyless entity
```

They also:

- ❌ Cannot contain navigations to owned entity types.
- ❌ Cannot participate in table splitting.

---

# 🧬 Inheritance

Keyless entity types support inheritance, but only using:

```text
TPH
Table Per Hierarchy
```

```text
Base Keyless Entity
        │
   ┌────┴────┐
   ↓         ↓
Type A     Type B
        ↓
      TPH
```

---

# 🪟 Complete View-Mapping Example

### Normal Entities

```csharp
public class Blog
{
    public int BlogId { get; set; }
    public string Name { get; set; }
    public string Url { get; set; }
    public ICollection<Post> Posts { get; set; }
}

public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public int BlogId { get; set; }
}
```

### Database View

```sql
CREATE VIEW View_BlogPostCounts AS
SELECT b.Name,
       COUNT(p.PostId) AS PostCount
FROM Blogs b
JOIN Posts p ON p.BlogId = b.BlogId
GROUP BY b.Name;
```

### Keyless Result Model

```csharp
public class BlogPostsCount
{
    public string BlogName { get; set; }
    public int PostCount { get; set; }
}
```

### Configure Mapping

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<BlogPostsCount>(eb =>
    {
        eb.HasNoKey();
        eb.ToView("View_BlogPostCounts");
        eb.Property(v => v.BlogName)
            .HasColumnName("Name");
    });
}
```

### Register DbSet

```csharp
public DbSet<BlogPostsCount> BlogPostCounts { get; set; }
```

### Query

```csharp
var postCounts =
    await context.BlogPostCounts
        .ToListAsync();
```

```text
Blogs + Posts
      ↓
Database View
      ↓
View_BlogPostCounts
      ↓
BlogPostsCount
      ↓
LINQ Query
      ↓
Results
```

---

# ⭐ Important Rules

```text
1. Keyless Entity = Entity Type without a primary key.

2. Main purpose = querying data without key values.

3. Keyless entities are read-only/query-oriented.

4. Configure with [Keyless] or HasNoKey().

5. They are useful for:
   → Views
   → Raw SQL
   → Keyless tables
   → Reporting models
   → Query projections

6. Keyless entities do not participate in normal
   INSERT / UPDATE / DELETE operations.

7. ToView() commonly maps a keyless entity to a database view.

8. ToTable() can also map a keyless entity to a table,
   but the entity remains query-only.

9. Keyless entities are not automatically discovered
   like regular entity types.

10. Navigation support is limited.

11. Keyless entities cannot have navigations to owned types.

12. Keyless entities support inheritance only through TPH.

13. Keyless entities cannot participate in table splitting.

14. A database view used by EF Core is assumed to already exist.

15. A view's returned columns must match the entity's mapped properties.

16. Extra view columns can be ignored.

17. Missing expected columns can cause query failures.
```

---

# ⚡ 30-Second Mental Model

```text
                 KEYLESS ENTITY
                       │
                       ▼
              No Primary Key
                       │
                       ▼
                Query-Oriented
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
     View           Raw SQL         Table
       │               │               │
       └───────────────┼───────────────┘
                       ▼
                 Read Results
```

```text
REGULAR ENTITY
→ Has Key
→ Can be tracked
→ INSERT / UPDATE / DELETE


KEYLESS ENTITY
→ No Key
→ Query / Read
→ No normal DML
→ Ideal for Views / Reports / Raw SQL
```

> 🎯 **Core takeaway:** A **keyless entity type** is an EF Core model type designed primarily for **reading query results that do not have a primary key**. It is especially useful for views, raw SQL results, reporting models, and legacy tables without keys.