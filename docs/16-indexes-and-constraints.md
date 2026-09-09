# 🧰 EF Core Interview Toolbox — Topic 15: Indexes and Constraints

> **Category:** 🏗️ Creating Model  
> **File Name:** `15-indexes-and-constraints.md`

---

# 📌 Indexes

## 🔎 Core Concept

- An **index** makes lookups based on one or more columns more efficient.
- Index implementation differs by database provider.
- EF Core creates an index in the database based on model configuration.
- By convention, EF Core creates indexes for **foreign key properties**.

```text
Without Index

Query
  ↓
Scan many rows
  ↓
Find matching data
  ↓
🐢 Slower


With Index

Query
  ↓
Index Lookup
  ↓
Find matching rows
  ↓
⚡ Faster
```

---

# 🛠️ Creating an Index

## Data Annotation

```csharp
[Index(nameof(Url))]
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
}
```

## Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url);
```

> ⭐ **Foreign key properties are indexed by convention**, unless that convention is removed.

---

# 🧩 Composite Index

An index can cover multiple columns.

```csharp
[Index(nameof(FirstName), nameof(LastName))]
public class Person
{
    public int PersonId { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

Or:

```csharp
modelBuilder.Entity<Person>()
    .HasIndex(p => new
    {
        p.FirstName,
        p.LastName
    });
```

### 🧠 Mental Model

```text
Composite Index

FirstName + LastName
        │
        ▼
┌──────────────────────┐
│ FirstName │ LastName │
└──────────────────────┘
```

A composite index helps queries filtering on:

```text
FirstName
FirstName + LastName
```

But not necessarily a query that uses only:

```text
LastName
```

> ⭐ **Column order matters.** The first column is the most important for using a composite index efficiently.

---

# 🔐 Index Uniqueness

Indexes are **not unique by default**.

```text
Url
──────
a.com
a.com
a.com

✅ Allowed
```

Make an index unique:

## Data Annotation

```csharp
[Index(nameof(Url), IsUnique = true)]
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
}
```

## Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url)
    .IsUnique();
```

Now:

```text
Url
──────
a.com
a.com

❌ Duplicate value
```

> ⭐ Attempting to insert duplicate values for a unique index/constraint causes a database exception.

---

# ↕️ Index Sort Order

Indexes can define ascending or descending ordering.

```text
Ascending

A → B → C → D


Descending

D → C → B → A
```

For a **single-column index**, sort direction usually matters less because many databases can scan the index in reverse.

The difference becomes more important for **composite indexes**.

```text
Composite Index

(BlogId ASC, CreatedOn ASC)

       ↓

BlogId
  ↓
CreatedOn
```

This is well suited to:

```sql
SELECT *
FROM Posts
WHERE BlogId = 10
ORDER BY CreatedOn ASC;
```

### 🧠 Key Rule

```text
Composite Index
        ↓
Column Order Matters
        ↓
First Column
        ↓
Second Column
        ↓
...
```

> ⭐ Queries generally benefit most when filtering/sorting follows the index definition from **left to right**.

---

# 🔽 Configuring Descending Indexes

## All Columns Descending

### Data Annotation

```csharp
[Index(
    nameof(Url),
    nameof(Rating),
    AllDescending = true)]
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
    public int Rating { get; set; }
}
```

### Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => new
    {
        b.Url,
        b.Rating
    })
    .IsDescending();
```

## Per-Column Direction

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => new
    {
        b.Url,
        b.Rating
    })
    .IsDescending(false, true);
```

```text
Url     → ASC
Rating  → DESC
```

---

# 🏷️ Index Naming

By convention:

```text
IX_<EntityName>_<PropertyName>
```

For composite indexes:

```text
IX_<EntityName>_<Property1>_<Property2>
```

Example:

```text
IX_Blog_Url
IX_Person_FirstName_LastName
```

Custom name:

## Data Annotation

```csharp
[Index(
    nameof(Url),
    Name = "Index_Url")]
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
}
```

## Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.Url)
    .HasDatabaseName("Index_Url");
```

---

# 🔁 Multiple Index Configurations

Calling `HasIndex()` again with the **same property set** continues configuring the same index.

```csharp
modelBuilder.Entity<Person>()
    .HasIndex(p => new
    {
        p.FirstName,
        p.LastName
    })
    .HasDatabaseName("IX_Names_Ascending");

modelBuilder.Entity<Person>()
    .HasIndex(p => new
    {
        p.FirstName,
        p.LastName
    })
    .HasDatabaseName("IX_Names_Descending")
    .IsDescending();
```

The second configuration overrides the first.

```text
HasIndex(FirstName, LastName)
        ↓
Index A
        ↓
Same HasIndex(...)
        ↓
Index A reconfigured
```

> ⭐ This behavior is useful when further configuring an index that already exists by convention.

---

# ➕ Multiple Indexes on the Same Properties

To create **multiple indexes over the same properties**, give each index a unique model name.

```csharp
modelBuilder.Entity<Person>()
    .HasIndex(
        p => new
        {
            p.FirstName,
            p.LastName
        },
        "IX_Names_Ascending");

modelBuilder.Entity<Person>()
    .HasIndex(
        p => new
        {
            p.FirstName,
            p.LastName
        },
        "IX_Names_Descending")
    .IsDescending();
```

```text
FirstName + LastName
        │
        ├── IX_Names_Ascending
        │
        └── IX_Names_Descending
```

> 💡 The name supplied to `HasIndex()` identifies the index in the EF Core model and is also used as the default database name.

---

# 🎯 Index Filters

> ⏳ **Index filters are a separate topic and will be studied later.**

---

# 📦 Included Columns

Some relational databases allow **included columns**.

These columns are stored with the index but are **not part of the index key**.

```text
Index
│
├── Key Columns
│     └── Used for lookup
│
└── Included Columns
      └── Extra data returned by query
```

### Example

```csharp
modelBuilder.Entity<Post>()
    .HasIndex(p => p.Url)
    .IncludeProperties(p => new
    {
        p.Title,
        p.PublishedOn
    });
```

Here:

```text
Index Key
→ Url

Included Columns
→ Title
→ PublishedOn
```

### 🧠 Why This Helps

If a query needs only indexed/included columns:

```text
Query
  ↓
Index contains everything needed
  ↓
No need to access base table
  ↓
⚡ Better performance
```

---

# 🔑 Key Columns vs Non-Key Columns

```text
Index
│
├── Key Columns
│     → Participate in the index key
│     → Used for lookup/order
│
└── Non-Key / Included Columns
      → Not part of the key
      → Store additional data
```

> 💡 A non-key column can contain duplicate values; uniqueness is determined by the actual key/constraint configuration.

---

# ✅ Check Constraints

A **check constraint** requires every row to satisfy a SQL condition.

```text
INSERT / UPDATE
      ↓
Check Constraint
      ↓
Condition valid?
   ┌──────┴──────┐
  YES            NO
   ↓              ↓
Allowed        ❌ Rejected
```

### Example

```csharp
modelBuilder.Entity<Product>()
    .ToTable(b =>
        b.HasCheckConstraint(
            "CK_Prices",
            "[Price] > [DiscountedPrice]"));
```

This ensures:

```text
Price > DiscountedPrice
```

Otherwise the database rejects the operation.

---

# 🧠 Check Constraint vs Other Constraints

```text
NOT NULL
→ Prevents NULL values


UNIQUE
→ Prevents duplicate values


CHECK
→ Enforces an arbitrary condition
```

A check constraint is useful when the rule cannot be expressed merely as nullability or uniqueness.

---

# 🔄 Multiple Check Constraints

Multiple check constraints can exist on the same table.

```text
Product
│
├── CK_Prices
│     → Price > DiscountedPrice
│
├── CK_Stock
│     → Stock >= 0
│
└── CK_...
```

Each constraint has its own name.

---

# 🧠 Indexes + Constraints — Big Picture

```text
                 DATABASE TABLE
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       INDEXES      UNIQUE       CHECK
          │        CONSTRAINTS   CONSTRAINTS
          │            │            │
          ▼            ▼            ▼
      Fast Lookup   No Duplicates  Valid Data
```

---

# ⚖️ Index vs Unique Index vs Check Constraint

| Concept | Main Purpose |
|---|---|
| Index | Improve lookup/query performance |
| Unique Index | Improve lookup + enforce uniqueness |
| Check Constraint | Enforce a data rule/condition |

---

# ⭐ Key Rules to Remember

```text
1. Indexes improve lookup efficiency.

2. EF Core creates indexes for foreign keys by convention.

3. Composite indexes contain multiple columns.

4. Composite index column order matters.

5. A composite index is most useful when query predicates
   follow its leading columns.

6. Indexes are non-unique by default.

7. IsUnique() creates a unique index.

8. A unique index rejects duplicate key values.

9. Index sort order can be configured globally or per column.

10. HasIndex() with the same property set reconfigures
    the existing index.

11. Give indexes distinct model names when multiple indexes
    use the same property set.

12. Included columns are non-key columns stored with an index
    to help cover queries.

13. Check constraints enforce arbitrary SQL conditions.

14. Multiple check constraints can exist on one table.

15. Indexes mainly help performance;
    constraints mainly enforce data rules.

16. Index filter configuration is a separate topic.
```

---

# ⚡ 30-Second Revision

```text
INDEX
  ↓
Faster Lookup

COMPOSITE INDEX
  ↓
Multiple Columns
  ↓
LEFT-TO-RIGHT ORDER MATTERS

UNIQUE INDEX
  ↓
Fast Lookup
  +
No Duplicate Values

INCLUDED COLUMNS
  ↓
Extra non-key data
  ↓
Can avoid table access for covered queries

CHECK CONSTRAINT
  ↓
SQL Condition
  ↓
Invalid Data → Rejected
```

> 🎯 **Core takeaway:** **Indexes improve how efficiently the database finds data, while constraints protect data integrity.** For interviews, remember the key distinctions: **composite index order matters, unique indexes enforce uniqueness, included columns are non-key coverage columns, and check constraints enforce business rules at the database level.**

> 📚 **Source:** These notes are based on the provided EF Core reference material. :contentReference[oaicite:0]{index=0}