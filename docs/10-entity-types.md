# 🧰 EF Core Interview Toolbox — Topic 10: Entity Types

> **Category:** 🏗️ Creating Model  
> **File Name:** `10-entity-types.md`

---

## 🔎 What Is an Entity Type?

- A type exposed through a `DbSet<TEntity>` is included in the EF Core model and is usually called an **entity type**.
- EF Core can read/write instances of entity types to the database.
- With relational databases, EF Core can create corresponding tables through **Migrations**.

```text
CLR Entity Type
      ↓
EF Core Model
      ↓
Database Mapping
      ↓
Table
```

---

# 🧩 1. How Entity Types Are Included in the Model

EF Core can discover entity types in several ways:

```text
                         EF Core Model
                              ▲
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
         DbSet<TEntity>  OnModelCreating   Navigation Property
              │               │                │
              ▼               ▼                ▼
           Entity          Entity         Related Entity
```

### By Convention

```csharp
public DbSet<Blog> Blogs { get; set; }
```

`Blog` is included because it is exposed through a `DbSet`.

### Explicitly in `OnModelCreating()`

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<AuditEntry>();
}
```

`AuditEntry` is included even without a `DbSet`.

### Through Navigation Properties

```text
Blog
 │
 └── Posts
       ↓
      Post
```

If `Blog` is already discovered, EF Core recursively explores its navigation properties and can discover `Post`.

### ⭐ Example

```csharp
public class MyContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }

    protected override void OnModelCreating(
        ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<AuditEntry>();
    }
}
```

```text
Blog
 → DbSet → INCLUDED

Post
 → Blog.Posts navigation → INCLUDED

AuditEntry
 → OnModelCreating() → INCLUDED
```

> ⭐ **A missing `DbSet<T>` does NOT automatically mean the type is excluded from the model.**

---

# 🚫 2. Excluding Entity Types

A type can be explicitly excluded from the EF Core model.

### Data Annotation

```csharp
[NotMapped]
public class BlogMetadata
{
    public DateTime LoadedFromDatabase { get; set; }
}
```

### Fluent API

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Ignore<BlogMetadata>();
}
```

---

## 🧠 `DbSet` Omission vs `Ignore<T>()`

```text
No DbSet<T>
     ↓
Not necessarily excluded
     ↓
May still be discovered through
navigation properties or configuration
```

To explicitly exclude it:

```text
Ignore<T>()
     ↓
Remove type from EF Core model
```

### Example

```text
Blog
 │
 └── Posts
       ↓
      Post
```

Without:

```csharp
modelBuilder.Ignore<Post>();
```

EF Core may:

```text
Discover Post
     ↓
Include Post in Model
     ↓
Map Post to Database
```

With:

```csharp
modelBuilder.Ignore<Post>();
```

```text
Post
 ↓
Ignored
 ↓
Not part of EF Core model
```

Therefore:

- ❌ No `Posts` table from this entity.
- ❌ No FK relationship generated for it.
- ❌ `Blog.Posts` is not persisted as an EF relationship.

---

# 🚫 3. Excluding an Entity From Migrations

Sometimes an entity should remain part of the EF Core model, but its table should **not be managed by Migrations**.

```csharp
modelBuilder.Entity<IdentityUser>()
    .ToTable(
        "AspNetUsers",
        t => t.ExcludeFromMigrations());
```

### 🧠 Important Distinction

```text
EF Core Model
      │
      ├── Entity included ✅
      │
      └── Migrations ignore table ✅
```

The entity can still be:

- Queried.
- Updated.
- Used in LINQ.

But migrations will not create or alter its table.

### Useful For

- Shared tables across multiple `DbContext` types.
- Existing/pre-created tables.
- Legacy or externally managed tables.
- Reporting/read-oriented schemas.

> ⭐ **Excluded from migrations ≠ excluded from the EF Core model.**

---

# 🏷️ 4. Table Name

By convention:

```text
DbSet Property Name
       ↓
Database Table Name
```

Example:

```csharp
public DbSet<Blog> Blogs { get; set; }
```

```text
DbSet Name = Blogs
      ↓
Table = Blogs
```

If there is no `DbSet`:

```text
Entity Class Name
       ↓
Table Name
```

### Override Table Name

#### Data Annotation

```csharp
[Table("blogs")]
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
}
```

#### Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .ToTable("blogs");
```

---

# 🗄️ 5. Table Schema

For relational databases, tables are created in the database's default schema by convention.

Example:

```text
SQL Server → dbo
SQLite     → No schema support
```

### Configure Schema Per Entity

```csharp
[Table("blogs", Schema = "blogging")]
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
}
```

Or:

```csharp
modelBuilder.Entity<Blog>()
    .ToTable("blogs", schema: "blogging");
```

### Configure Default Schema

Instead of configuring every table:

```csharp
modelBuilder.HasDefaultSchema("blogging");
```

```text
Model
  ↓
Default Schema = blogging
  ↓
Tables use blogging schema
```

> ⭐ Setting a default schema can also affect other database objects such as sequences.

---

# 👁️ 6. View Mapping

An entity can be mapped to a database view:

```csharp
modelBuilder.Entity<Blog>()
    .ToView("blogsView", schema: "blogging");
```

### Important

EF Core assumes the view already exists.

```text
EF Core
   ↓
Map Entity → Existing View
```

> ⚠️ EF Core does **not** automatically create the view through a migration.

---

## 🔄 View Mapping vs Table Mapping

An entity can also have an explicit table mapping.

```text
Entity
  │
  ├── Query → View
  │
  └── Insert/Update/Delete → Table
```

### Mental Model

```text
LINQ Query
    ↓
blogsView

Save Changes
    ↓
Blogs Table
```

So:

- **View mapping** is used for queries.
- **Table mapping** is used for updates.

---

## 📌 View Compatibility

The view should return columns corresponding to the entity's mapped properties.

```text
Entity Property
      ↕
View Column
```

- Extra columns returned by the view can be ignored.
- Missing columns expected by the entity can cause query failures at runtime.
- Keyless entities are common for views because many views do not have a primary key.

Example:

```csharp
modelBuilder.Entity<Blog>()
    .HasNoKey();
```

> ⭐ `HasNoKey()` makes the entity keyless and suitable for query-only scenarios.

---

# 📊 7. Table-Valued Function (TVF) Mapping

An entity type can be mapped to a **parameterless table-valued function** instead of a table.

Example entity:

```csharp
public class BlogWithMultiplePosts
{
    public string Url { get; set; }
    public int PostCount { get; set; }
}
```

Example SQL function:

```sql
CREATE FUNCTION dbo.BlogsWithMultiplePosts()
RETURNS TABLE
AS
RETURN
(
    SELECT b.Url,
           COUNT(p.BlogId) AS PostCount
    FROM Blogs AS b
    JOIN Posts AS p ON b.BlogId = p.BlogId
    GROUP BY b.BlogId, b.Url
    HAVING COUNT(p.BlogId) > 1
)
```

Map it:

```csharp
modelBuilder.Entity<BlogWithMultiplePosts>()
    .HasNoKey()
    .ToFunction("BlogsWithMultiplePosts");
```

> ⭐ The TVF must be **parameterless** for this entity mapping.

---

## 🔎 Querying the TVF-Mapped Entity

```csharp
var query =
    from b in context.Set<BlogWithMultiplePosts>()
    where b.PostCount > 3
    select new
    {
        b.Url,
        b.PostCount
    };
```

Conceptually:

```text
LINQ
  ↓
TVF
  ↓
Filtered Result
```

EF Core maps entity properties to matching columns returned by the TVF.

If column names differ, configure them using `HasColumnName()`.

---

# 💬 8. Table Comments

A database table can have a descriptive comment.

### Data Annotation

```csharp
[Comment("Blogs managed on the website")]
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
}
```

### Fluent API

```csharp
modelBuilder.Entity<Blog>()
    .ToTable(
        tableBuilder => tableBuilder
            .HasComment("Blogs managed on the website"));
```

```text
Entity
  ↓
Table
  ↓
Database Comment
```

---

# 🧱 9. Shared-Type Entity Types

Normally:

```text
1 Entity Type
      ↓
1 CLR Class
```

Example:

```text
Blog → Blog class
Post → Post class
```

A **shared-type entity** allows multiple EF Core entity types to reuse the same CLR type.

Example:

```text
              Dictionary<string, object>
                    ▲             ▲
                    │             │
                  Blog           Post
               Entity Type    Entity Type
```

### ⭐ Example

```csharp
public DbSet<Dictionary<string, object>> Blogs
    => Set<Dictionary<string, object>>("Blog");

protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.SharedTypeEntity<Dictionary<string, object>>(
        "Blog",
        bb =>
        {
            bb.Property<int>("BlogId");
            bb.Property<string>("Url");
            bb.Property<DateTime>("LastUpdated");
        });
}
```

Query:

```csharp
var blogs = context.Blogs
    .Where(b => (string)b["Url"] == "example.com")
    .ToList();
```

---

## 🔄 Multiple Logical Entities Using One CLR Type

The same `Dictionary<string, object>` can represent different EF entity types:

```csharp
public DbSet<Dictionary<string, object>> Blogs
    => Set<Dictionary<string, object>>("Blog");

public DbSet<Dictionary<string, object>> Posts
    => Set<Dictionary<string, object>>("Post");
```

```text
Dictionary<string, object>
        │
        ├── "Blog" Entity Type
        │      ├── BlogId
        │      ├── Url
        │      └── LastUpdated
        │
        └── "Post" Entity Type
               ├── PostId
               ├── Title
               ├── Content
               └── BlogId
```

### Useful For

- Dynamic schemas.
- Metadata-driven systems.
- Extensible applications.
- Multiple logical entities without separate CLR classes.
- Scenarios where the data shape is not known at compile time.

---

# ⭐ Key Distinctions

| Concept | Key Point |
|---|---|
| `DbSet<T>` | Includes `T` in the model |
| `OnModelCreating()` | Can explicitly include entity types |
| Navigation Property | Can cause related entity types to be discovered |
| `Ignore<T>()` | Removes a type from the EF Core model |
| `[NotMapped]` | Excludes a type/property from mapping |
| `ExcludeFromMigrations()` | Keeps entity in model but migrations ignore its table |
| Table Name | Usually comes from `DbSet` name; otherwise entity type name |
| Schema | Can be configured per table or as model default |
| `ToView()` | Maps an entity for querying through a database view |
| `ToFunction()` | Maps an entity to a parameterless TVF |
| `HasNoKey()` | Configures a keyless entity |
| `EntityTypeConfigurationAttribute` | Not automatically discovered from an assembly |
| Shared-Type Entity | Multiple EF entity types can reuse one CLR type |

---

# 🧠 30-SECOND MENTAL MODEL

```text
                 ENTITY TYPES
                      │
        ┌─────────────┼──────────────┐
        │             │              │
        ▼             ▼              ▼
     DbSet       OnModelCreating   Navigation
        │             │              │
        └─────────────┼──────────────┘
                      ↓
                 EF CORE MODEL
                      │
          ┌───────────┼────────────┐
          │           │            │
          ▼           ▼            ▼
        Table       View          TVF
```

```text
INCLUDE
───────
DbSet<T>
OnModelCreating()
Navigation discovery


EXCLUDE
───────
[NotMapped]
Ignore<T>()


MIGRATIONS
──────────
Entity included
      +
ExcludeFromMigrations()
      ↓
EF can use entity
but migrations ignore table


MAPPING
───────
Entity → Table
Entity → View
Entity → Parameterless TVF


SHARED TYPE
───────────
Multiple EF entity types
        ↓
Same CLR type
        ↓
Dictionary<string, object>
```

> 🎯 **Core takeaway:** EF Core builds its entity model through `DbSet` properties, explicit configuration, and navigation discovery. You can then control exactly how each entity maps to tables, schemas, views, TVFs, or shared CLR types — and separately control whether its database object is managed by migrations.