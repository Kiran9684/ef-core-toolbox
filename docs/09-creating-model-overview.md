# 🧰 EF Core Interview Toolbox — Topic 9: Creating Model — Overview

## 🏗️ Creating Model

```text
EF CORE MODEL
│
├── Metadata Model
│      ↓
│   Describes how C# entities
│   map to the database
│
├── Built-in Conventions
│      ↓
│   Recognize common patterns
│
├── Data Annotations
│      ↓
│   Customize / override conventions
│
└── Fluent API
       ↓
    Customize / override everything above
```

---

## 🔎 EF Core Model

- EF Core uses a **metadata model** to describe how application entity types are mapped to the underlying database.
- The model contains the mapping information EF Core needs to work with the database.
- Most model configuration can target any supported data store.

### 🧠 Mental Model

```text
C# Entity Types
      ↓
EF Core Metadata Model
      ↓
Mapping Information
      ↓
Database
```

---

## 🔧 How the Model Is Configured

EF Core starts with **conventions** and allows customization through:

1. **Conventions**
2. **Data Annotations**
3. **Fluent API**

```text
                 EF CORE MODEL
                      ▲
                      │
          ┌───────────┼───────────┐
          │           │           │
          │           │           │
     Conventions   Data       Fluent API
                   Annotations
```

### ⭐ Configuration Precedence

```text
Conventions
     ↓
Data Annotations
     ↓
Fluent API
```

> **Fluent API has the highest precedence.**

Therefore:

- Data Annotations override conventions.
- Fluent API overrides both conventions and Data Annotations.

---

## 🔷 Fluent API

- Configure the model by overriding `OnModelCreating()`.
- Uses `ModelBuilder`.
- This is the **most powerful configuration approach**.
- Allows configuration without modifying entity classes.
- Has the highest configuration precedence.

### ⭐ Example

```csharp
using Microsoft.EntityFrameworkCore;

public class MyContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }

    protected override void OnModelCreating(
        ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>()
            .Property(b => b.Url)
            .IsRequired();
    }
}

public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
}
```

### 🧠 Fluent API Flow

```text
OnModelCreating()
       ↓
ModelBuilder
       ↓
Entity<Blog>()
       ↓
Property(b => b.Url)
       ↓
IsRequired()
       ↓
Model Configuration
```

---

## 🔄 Fluent API Configuration Order

Fluent API configuration is applied in the order the configuration calls are made.

```text
Configuration A
      ↓
Configuration B
      ↓
Configuration C
      ↓
Final Configuration
```

> ⭐ If conflicting Fluent API configuration is applied, the **latest applicable configuration wins**.

---

## 📦 Grouping Entity Configuration

Large `OnModelCreating()` methods can become difficult to maintain.

Move configuration for an entity into a separate class implementing:

```text
IEntityTypeConfiguration<TEntity>
```

### ⭐ Example

```csharp
public class BlogEntityTypeConfiguration
    : IEntityTypeConfiguration<Blog>
{
    public void Configure(
        EntityTypeBuilder<Blog> builder)
    {
        builder
            .Property(b => b.Url)
            .IsRequired();
    }
}
```

Apply it from `OnModelCreating()`:

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    new BlogEntityTypeConfiguration()
        .Configure(modelBuilder.Entity<Blog>());
}
```

### 🧠 Mental Model

```text
OnModelCreating()
      ↓
Separate Configuration Class
      ↓
Configure(EntityTypeBuilder<TEntity>)
      ↓
Entity Configuration
```

> 💡 This keeps `OnModelCreating()` smaller and more organized.

---

## 🏭 Apply All Configurations From an Assembly

When many `IEntityTypeConfiguration<TEntity>` classes exist, EF Core can discover and apply them automatically.

```csharp
modelBuilder.ApplyConfigurationsFromAssembly(
    typeof(BlogEntityTypeConfiguration).Assembly);
```

### 🧠 Mental Model

```text
Assembly
   ↓
Scan Configuration Classes
   ↓
Find IEntityTypeConfiguration<T>
   ↓
Apply Configurations
```

> ⚠️ The order in which configurations are applied is **undefined**. Use this approach when configuration order does not matter.

---

## 🏷️ EntityTypeConfigurationAttribute

A configuration class can also be associated with an entity using:

```csharp
[EntityTypeConfiguration(typeof(BookConfiguration))]
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Isbn { get; set; }
}
```

Configuration class:

```csharp
public class BookConfiguration
    : IEntityTypeConfiguration<Book>
{
    public void Configure(
        EntityTypeBuilder<Book> builder)
    {
        builder
            .Property(b => b.Isbn)
            .IsRequired();
    }
}
```

### How EF Core Finds It

The entity must first be included in the model, for example through:

```csharp
public DbSet<Book> Books { get; set; }
```

or:

```csharp
modelBuilder.Entity<Book>();
```

Then EF Core can discover the `EntityTypeConfigurationAttribute` and apply the referenced configuration.

### ⚠️ Important Limitation

`EntityTypeConfigurationAttribute` types are **not automatically discovered by scanning an assembly**.

```text
Entity included in model?
        │
   ┌────┴────┐
   │         │
  YES        NO
   │         │
Check      Attribute
Attribute  not applied
   │
   ▼
Apply Configuration
```

### 🆚 Attribute vs Assembly Scanning

```text
EntityTypeConfigurationAttribute
→ Checked when the entity is included in the model.

ApplyConfigurationsFromAssembly()
→ Scans the assembly for configuration classes.
```

---

## 📝 Data Annotations

- Data Annotations are attributes applied to entity classes or properties.
- They can configure certain model aspects directly on the entity.
- They override conventions.
- Fluent API overrides Data Annotations.

### ⭐ Example

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

[Table("Blogs")]
public class Blog
{
    public int BlogId { get; set; }

    [Required]
    public string Url { get; set; }
}
```

### 🧠 Precedence

```text
Convention
    ↓
[Required]
    ↓
Fluent API
```

---

## 🔧 Built-in Conventions

EF Core contains many built-in conventions that are enabled by default.

Examples include conventions that help discover:

- Keys
- Relationships
- Foreign Keys
- Indexes
- Other common model patterns

These conventions are implemented through types supporting the convention system, including `IConvention`.

### ⭐ Important

EF Core allows applications to:

- Remove built-in conventions.
- Replace conventions.
- Add custom conventions.

---

## 🚫 Removing a Convention

Sometimes a built-in convention is not appropriate.

### Example: Foreign Key Index Convention

EF Core normally creates indexes for foreign key columns.

Example:

```text
Post
│
├── BlogId   → Foreign Key + Index
└── AuthorId → Foreign Key + Index
```

Indexes improve lookup performance but also have storage and maintenance overhead.

The convention can be removed:

```csharp
protected override void ConfigureConventions(
    ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Conventions.Remove(
        typeof(ForeignKeyIndexConvention));
}
```

### Result

```text
Before
──────

BlogId   → FK + Index
AuthorId → FK + Index


After removing convention
────────────────────────

BlogId   → FK
AuthorId → FK
```

> ⭐ The convention is removed globally from convention-based model building, but indexes can still be added explicitly when required.

Indexes can be configured explicitly using:

- `IndexAttribute`
- Fluent API

---

## 🐞 Model Debug View

EF Core provides a **model debug view** that helps inspect the metadata EF Core built.

It can be viewed:

- In the IDE debugger.
- Directly from code.

### ⭐ Short Debug View

```csharp
Console.WriteLine(
    context.Model.ToDebugString());
```

### ⭐ Long Debug View

```csharp
Console.WriteLine(
    context.Model.ToDebugString(
        MetadataDebugStringOptions.LongDefault));
```

### 🧠 Mental Model

```text
Entity Classes
      ↓
Conventions
      ↓
Data Annotations
      ↓
Fluent API
      ↓
Final EF Core Model
      ↓
ToDebugString()
      ↓
Inspect Metadata
```

### Short vs Long View

```text
Short View
→ Main model information

Long View
→ More detailed information
→ Includes annotations
→ Useful for relational/provider-specific metadata
```

---

## ⭐ Key Rules to Remember

```text
1. EF Core builds a metadata model that maps
   application entities to the database.

2. Conventions provide the default configuration.

3. Data Annotations override conventions.

4. Fluent API has the highest precedence.

5. Fluent API configuration is written in OnModelCreating().

6. Later conflicting Fluent API configuration
   overrides earlier configuration.

7. IEntityTypeConfiguration<TEntity> helps separate
   entity configuration from OnModelCreating().

8. ApplyConfigurationsFromAssembly() automatically finds
   IEntityTypeConfiguration<T> classes in an assembly.

9. The order of ApplyConfigurationsFromAssembly()
   configuration application is undefined.

10. EntityTypeConfigurationAttribute is checked only
    after the entity is included in the model.

11. EntityTypeConfigurationAttribute does NOT automatically
    scan an assembly for configuration classes.

12. Built-in conventions can be removed, replaced,
    or extended with custom conventions.

13. Removing ForeignKeyIndexConvention prevents
    automatic FK indexes from being created by convention.

14. Required indexes can still be configured explicitly.

15. ToDebugString() helps inspect the final EF Core model.
```

---

## ⚡ 30-Second Mental Model

```text
              C# ENTITIES
                   ↓
             CONVENTIONS
                   ↓
          DATA ANNOTATIONS
                   ↓
             FLUENT API
                   ↓
            FINAL MODEL
                   ↓
          DATABASE MAPPING


Configuration Priority:

Conventions
    ↓
Data Annotations
    ↓
Fluent API
    ⭐ Highest


Organization:

OnModelCreating()
        ↓
IEntityTypeConfiguration<T>
        ↓
ApplyConfigurationsFromAssembly()


Debugging:

Final Model
    ↓
ToDebugString()
    ↓
Inspect EF Core Metadata
```

> 🎯 **Core takeaway:** EF Core first builds a model using conventions, then allows customization through Data Annotations and Fluent API. **Fluent API has the highest precedence**, while configuration classes and convention customization help keep larger models organized and maintainable.