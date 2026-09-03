# 🧰 EF Core Interview Toolbox — Topic 11: Entity Properties

> **Category:** 🏗️ Creating Model  
> **File Name:** `11-entity-properties.md`

---

# 🔎 Entity Properties — Core Concept

- Every entity type contains a set of **properties** that EF Core reads from and writes to the database.
- With relational databases, entity properties normally map to **table columns**.

```text
Entity
  │
  ├── Property 1 ──→ Column 1
  ├── Property 2 ──→ Column 2
  └── Property 3 ──→ Column 3
```

---

# ✅ Included vs Excluded Properties

By convention:

> **Public properties with both getter and setter are included in the EF Core model.**

```csharp
public class Blog
{
    public int BlogId { get; set; }     // Included
    public string Url { get; set; }     // Included

    [NotMapped]
    public DateTime LoadedFromDatabase { get; set; } // Excluded
}
```

A property can be excluded using:

### Data Annotation

```csharp
[NotMapped]
public DateTime LoadedFromDatabase { get; set; }
```

### Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .Ignore(b => b.LoadedFromDatabase);
```

---

# 🏷️ Column Names

By convention:

```text
Entity Property Name
        ↓
Database Column Name
```

Example:

```csharp
public int BlogId { get; set; }
```

→ Column: `BlogId`

The column name can be changed.

### Data Annotation

```csharp
[Column("blog_id")]
public int BlogId { get; set; }
```

### Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.BlogId)
    .HasColumnName("blog_id");
```

---

# 🧠 Column Data Types

For relational databases:

```text
.NET Property Type
       +
Other EF Metadata
       ↓
Database Provider
       ↓
Database Column Type
```

The provider considers information such as:

- .NET property type.
- Maximum length.
- Whether the property is part of a key.
- Other configured metadata.

### Example — SQL Server

```text
DateTime
   ↓
datetime2(7)

string
   ↓
nvarchar(max)

string used as key
   ↓
nvarchar(450)
```

You can explicitly configure the column type.

### Data Annotation

```csharp
[Column(TypeName = "varchar(200)")]
public string Url { get; set; }

[Column(TypeName = "decimal(5, 2)")]
public decimal Rating { get; set; }
```

### Fluent API

```csharp
modelBuilder.Entity<Blog>(eb =>
{
    eb.Property(b => b.Url)
        .HasColumnType("varchar(200)");

    eb.Property(b => b.Rating)
        .HasColumnType("decimal(5, 2)");
});
```

---

# 📏 Maximum Length

Maximum length can be configured for array types such as:

- `string`
- `byte[]`

### Example

```csharp
[MaxLength(500)]
public string Url { get; set; }
```

or:

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .HasMaxLength(500);
```

For SQL Server this can result in:

```text
HasMaxLength(500)
       ↓
nvarchar(500)
```

> ⚠️ EF Core does **not** perform maximum-length validation before sending the value to the database. The provider/database is responsible for enforcing the limit.

---

# 🎯 Precision & Scale

Some relational data types support **precision** and **scale**.

```text
Precision
→ Total number of digits

Scale
→ Number of digits after the decimal point
```

### Decimal Example

```text
decimal(14,2)

14 → Precision
 2 → Scale
```

### DateTime Example

For `DateTime`:

```text
Precision
→ Precision of fractional seconds

Scale
→ Not used
```

> ⚠️ Support for precision and scale is **database/provider dependent**.

---

## ⭐ Configure Precision

### Data Annotation

```csharp
[Precision(14, 2)]
public decimal Score { get; set; }

[Precision(3)]
public DateTime LastUpdated { get; set; }
```

### Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Score)
    .HasPrecision(14, 2);

modelBuilder.Entity<Blog>()
    .Property(b => b.LastUpdated)
    .HasPrecision(3);
```

Example SQL Server result:

```text
Score
  ↓
decimal(14,2)

LastUpdated
  ↓
datetime2(3)
```

> ⭐ **Scale is specified together with precision:** `HasPrecision(precision, scale)`.

> ⚠️ EF Core does not validate precision/scale before sending values to the provider.

---

# 🔤 Unicode vs Non-Unicode

Some databases distinguish between Unicode and non-Unicode text.

SQL Server example:

```text
nvarchar → Unicode
varchar  → Non-Unicode
```

Text properties are configured as **Unicode by default**.

### Make a Property Non-Unicode

```csharp
[Unicode(false)]
[MaxLength(22)]
public string Isbn { get; set; }
```

or:

```csharp
modelBuilder.Entity<Book>()
    .Property(b => b.Isbn)
    .IsUnicode(false);
```

```text
string
  ↓
Unicode by default

IsUnicode(false)
  ↓
Non-Unicode
```

---

# ❓ Required vs Optional Properties

A property is:

```text
Optional
→ NULL is a valid value

Required
→ NULL is NOT a valid value
```

When mapped to a relational database:

```text
Required Property
      ↓
Non-nullable Column


Optional Property
      ↓
Nullable Column
```

### Mental Model

```text
C# / EF Property
       ↓
Can it contain NULL?
       │
   ┌───┴───┐
   │       │
  YES      NO
   │       │
Optional Required
   │       │
NULL      NOT NULL
```

---

# 🔤 Column Collation

A **collation** determines how text values are **compared and ordered**.

Example:

```csharp
modelBuilder.Entity<Customer>()
    .Property(c => c.Name)
    .UseCollation("SQL_Latin1_General_CP1_CI_AS");
```

```text
Text Column
    ↓
Collation
    ↓
Comparison + Ordering Rules
```

> 💡 If the same collation is required for all columns, it can be configured at the **database level** instead.

---

# 💬 Column Comments

A column can have a database comment for documentation.

### Data Annotation

```csharp
[Comment("The URL of the blog")]
public string Url { get; set; }
```

### Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .HasComment("The URL of the blog");
```

```text
Property
   ↓
Column
   ↓
Database Comment
```

---

# 🔢 Column Order

When Migrations create a table, EF Core has a default ordering for columns.

Conceptually:

```text
Primary Key
    ↓
Entity / Owned-Type Properties
    ↓
Base-Type Properties
```

You can explicitly specify the order.

### Data Annotation

```csharp
public class EntityBase
{
    [Column(Order = 0)]
    public int Id { get; set; }
}

public class PersonBase : EntityBase
{
    [Column(Order = 1)]
    public string FirstName { get; set; }

    [Column(Order = 2)]
    public string LastName { get; set; }
}
```

### Fluent API

```csharp
modelBuilder.Entity<Employee>(x =>
{
    x.Property(e => e.Id)
        .HasColumnOrder(0);

    x.Property(e => e.FirstName)
        .HasColumnOrder(1);

    x.Property(e => e.LastName)
        .HasColumnOrder(2);
});
```

> ⚠️ In general, most databases support column ordering only when the table is created. Changing the configured order does **not necessarily reorder columns in an existing table**.

---

# 🧠 Property Configuration Mental Model

```text
                 ENTITY PROPERTY
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
      Mapping       Data Type     Behavior
          │            │            │
          │            │            ├── Required / Optional
          │            │            ├── Unicode
          │            │            └── Collation
          │            │
          │            ├── Column Type
          │            ├── Max Length
          │            └── Precision / Scale
          │
          ├── Column Name
          ├── Column Order
          └── Column Comment
```

---

# ⚖️ Data Annotation vs Fluent API

| Configuration | Data Annotation | Fluent API |
|---|---|---|
| Exclude property | `[NotMapped]` | `Ignore()` |
| Column name | `[Column("...")]` | `HasColumnName()` |
| Column type | `[Column(TypeName = "...")]` | `HasColumnType()` |
| Max length | `[MaxLength()]` | `HasMaxLength()` |
| Precision | `[Precision()]` | `HasPrecision()` |
| Unicode | `[Unicode()]` | `IsUnicode()` |
| Column comment | `[Comment()]` | `HasComment()` |
| Column order | `[Column(Order = ...)]` | `HasColumnOrder()` |

---

# ⭐ Key Rules to Remember

```text
1. Public properties with getter + setter are included by convention.

2. [NotMapped] and Ignore() exclude properties from the model.

3. Property names map to column names by convention.

4. Database column types are selected by the provider
   based on .NET type + other model metadata.

5. HasMaxLength() configures maximum length but EF Core
   does not validate the value before sending it to the provider.

6. Precision = total digits.
   Scale = decimal places.

7. Scale is configured together with precision.

8. Text properties are Unicode by default.

9. IsUnicode(false) configures non-Unicode text.

10. Required property → non-nullable database column.

11. Optional property → nullable database column.

12. Collation controls text comparison and ordering.

13. Column comments document the database schema.

14. Column order mainly matters when a table is created.

15. Provider/database behavior can differ for data types,
    precision, scale, Unicode, and other column facets.
```

---

# ⚡ 30-Second Mental Model

```text
ENTITY PROPERTY
      ↓
┌─────────────────────────────┐
│ Included / Excluded         │
│ Column Name                 │
│ Column Type                 │
│ Max Length                  │
│ Precision / Scale           │
│ Unicode                     │
│ Required / Optional         │
│ Collation                   │
│ Comment                     │
│ Column Order                │
└─────────────────────────────┘
      ↓
DATABASE COLUMN
```

```text
.NET Type
   +
EF Configuration
   +
Database Provider
   ↓
Final Database Column Definition
```

> 🎯 **Core takeaway:** EF Core maps entity properties to database columns and lets you control their **name, type, length, precision, Unicode behavior, nullability, collation, comments, and order** through conventions, Data Annotations, or Fluent API.