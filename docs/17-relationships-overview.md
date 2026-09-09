# 🧰 EF Core Interview Toolbox — Topic 16: Relationships — Overview

> **Category:** 🏗️ Creating Model  
> **File Name:** `16-relationships-overview.md`

---

# 🔗 Relationships in EF Core

## 🔎 Core Concept

A **relationship** defines how two entity types are connected.

Example:

```text
Blog
  │
  └── Posts
        │
        ├── Post 1
        ├── Post 2
        └── Post 3
```

In C#, the relationship is represented using **navigation properties**.

```csharp
public class Blog
{
    public string Name { get; set; }

    public ICollection<Post> Posts { get; set; }
}

public class Post
{
    public string Title { get; set; }

    public Blog Blog { get; set; }
}
```

Here:

```text
Blog.Posts
    ↕
Post.Blog
```

> ⭐ `Blog.Posts` and `Post.Blog` represent **one relationship**, not two separate relationships.

These properties are called **navigations** in EF Core.

---

# 🧠 Object Model vs Database Model

The same relationship is represented differently in C# and in a relational database.

```text
OBJECT MODEL                     RELATIONAL DATABASE

Blog                             Blogs
 │                                │
 │ Posts                          │ Id (PK)
 ▼                                │
Post                             └──────────┐
                                            │
                                  BlogId (FK)│
                                            ▼
                                          Posts
```

### Object-Oriented View

```text
Blog
 ↓
Posts collection

Post
 ↓
Blog reference
```

### Relational View

```text
Blogs.Id
   ▲
   │
Posts.BlogId
```

> ⭐ **C# uses object references/navigation properties.**  
> **Relational databases use primary keys and foreign keys.**

---

# 🔑 Mapping a Relationship

EF Core connects the two representations.

At the basic level:

```text
Principal Entity
      │
      │ Primary Key
      ▼
Dependent Entity
      │
      │ Foreign Key
      ▼
Database Relationship
```

Typical model:

```csharp
public class Blog
{
    public int Id { get; set; }                 // PK

    public string Name { get; set; }

    public ICollection<Post> Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }                 // PK

    public string Title { get; set; }

    public int BlogId { get; set; }             // FK

    public Blog Blog { get; set; }
}
```

---

# 🔄 EF Core Relationship Synchronization

Once the relationship is mapped, EF Core keeps the object relationship and FK relationship synchronized.

```text
Navigation Changes
      ↓
EF Core updates FK
      ↓
SaveChanges()
      ↓
Database updated
```

And the reverse can also happen:

```text
Foreign Key Changes
      ↓
EF Core updates navigation
      ↓
Object relationship reflects FK
```

### 🧠 Mental Model

```text
          OBJECT MODEL
               │
       Navigation Properties
               │
               ↕
          EF Core Mapping
               │
               ↕
       Primary Key / Foreign Key
               │
               ▼
       RELATIONAL DATABASE
```

---

# 🔷 Principal vs Dependent

In:

```text
Blog → Posts
```

the usual interpretation is:

```text
Blog
 ↓
Principal

Post
 ↓
Dependent
```

Because:

```text
Blog.Id
   ↑
   │
Post.BlogId
```

The dependent contains the foreign key.

---

# 🛠️ Fluent API Relationship Configuration

EF Core can often discover simple relationships automatically by convention.

You can also configure them explicitly in `OnModelCreating()`.

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasMany(e => e.Posts)
        .WithOne(e => e.Blog)
        .HasForeignKey(e => e.BlogId)
        .HasPrincipalKey(e => e.Id);
}
```

### 🧠 Read It Left to Right

```text
Blog
 ↓
HasMany(Posts)
 ↓
Each Post
 ↓
WithOne(Blog)
 ↓
Post.BlogId = Foreign Key
 ↓
Blog.Id = Principal Key
```

### Each Method

```text
HasMany(e => e.Posts)
→ One Blog has many Posts.

WithOne(e => e.Blog)
→ Each Post has one Blog.

HasForeignKey(e => e.BlogId)
→ Post.BlogId is the FK.

HasPrincipalKey(e => e.Id)
→ Blog.Id is the principal key.
```

> ⭐ `HasPrincipalKey()` is usually unnecessary when the principal key is the normal primary key, but it makes the relationship explicit.

---

# 🔢 Types of Relationships

EF Core supports three fundamental relationship shapes:

```text
RELATIONSHIPS
│
├── 1️⃣ One-to-Many
│      │
│      └── One Blog → Many Posts
│
├── 2️⃣ One-to-One
│      │
│      └── One Entity → One Entity
│
└── 3️⃣ Many-to-Many
       │
       └── Many Entities ↔ Many Entities
```

---

# 🔍 Foreign Keys & Principal Keys

A relationship commonly consists of:

```text
Principal Key
      │
      ▼
Foreign Key
      │
      ▼
Dependent Entity
```

The **primary key** is not only used for relationships; it also uniquely identifies an entity.

EF Core can also use an **alternate key** as the principal key when configured appropriately.

---

# 🧭 Relationship Navigations

Navigations provide the object-oriented way to move through a relationship.

```text
Blog
 │
 └── Posts
       ↓
      Post
       │
       └── Blog
            ↓
           Blog
```

This means a relationship can usually be traversed in both directions:

```text
Blog → Posts
Posts → Blog
```

> ⭐ Both navigations describe the **same relationship**.

---

# 🔎 Ways Relationships Are Used

Relationships defined in the EF Core model are useful beyond database mapping.

## 1️⃣ Query Related Data

### Eager Loading

```csharp
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();
```

```text
Blog
 ↓
Include()
 ↓
Posts loaded with Blog
```

### Lazy Loading

```text
Access Navigation
      ↓
EF Core loads related data
```

This can be implemented using lazy-loading proxies or other lazy-loading mechanisms.

### Explicit Loading

```text
Entity
  ↓
Load() / LoadAsync()
  ↓
Related Data
```

---

# 🌱 Relationships & Data Seeding

Relationships can also be represented when seeding data.

```text
Blog
 │
 └── Id = 1
       ▲
       │
Post
 └── BlogId = 1
```

The relationship is established by matching:

```text
Principal PK
      ↕
Dependent FK
```

---

# 🌳 Relationships & Entity Graphs

Relationships allow EF Core to work with complete graphs of related entities.

```text
Blog
 │
 ├── Post 1
 ├── Post 2
 └── Post 3
```

The Change Tracker can then track the graph and detect relationship changes.

```text
Entity Graph
     ↓
Change Tracker
     ↓
Detect Changes
     ↓
Relationship Fixup
     ↓
SaveChanges()
```

---

# 🔄 Relationship Fixup

EF Core performs **relationship fixup** to keep navigations and foreign keys consistent.

Example:

```csharp
post.Blog = blog;
```

Conceptually:

```text
post.Blog
   ↓
New Blog Reference
   ↓
EF Core relationship fixup
   ↓
post.BlogId updated
```

Similarly, FK changes can update navigation properties.

> ⭐ **Relationship fixup keeps the in-memory object graph consistent with the configured relationship.**

---

# ⚙️ Configuration Options

EF Core relationships can be configured using:

```text
                 RELATIONSHIP CONFIGURATION
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ▼             ▼              ▼
        Conventions   Mapping Attributes   Fluent API
```

### Convention

EF Core automatically discovers common relationship patterns.

### Mapping Attributes

Some relationship configuration can be specified using attributes.

### Fluent API

```text
OnModelCreating()
       ↓
ModelBuilder
       ↓
Relationship Configuration
```

> ⭐ **Fluent API is the final source of truth and has the highest precedence over conventions and mapping attributes.**

It also provides the most complete configuration capabilities.

---

# 🧠 Big Picture

```text
                    RELATIONSHIP
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
     OBJECT MODEL                RELATIONAL MODEL
          │                             │
          ▼                             ▼
 Navigation Properties          PK + FK
          │                             │
          └──────────────┬──────────────┘
                         ▼
                    EF CORE MODEL
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
            Query      Track      Save
```

---

# ⭐ Key Rules to Remember

```text
1. A relationship defines how two entity types are connected.

2. Navigation properties represent relationships
   in the C# object model.

3. A relationship is one relationship even when it
   has navigations in both directions.

4. Relational databases represent relationships
   using primary keys and foreign keys.

5. EF Core maps navigations to PK/FK relationships.

6. The dependent usually contains the foreign key.

7. EF Core can discover simple relationships by convention.

8. Relationships can be configured explicitly with Fluent API.

9. HasMany() + WithOne() commonly represents one-to-many.

10. Relationship changes can cause FK values and navigation
    properties to be synchronized through relationship fixup.

11. Relationships can be used for:
    → Eager loading
    → Lazy loading
    → Explicit loading
    → Data seeding
    → Entity graph tracking

12. EF Core supports:
    → One-to-many
    → One-to-one
    → Many-to-many

13. Fluent API takes precedence over conventions
    and mapping attributes.

14. Fluent API provides the most complete relationship
    configuration capabilities.
```

---

# ⚡ 30-Second Mental Model

```text
C# OBJECT MODEL
───────────────

Blog
  ↕
Post

Blog.Posts
Post.Blog

        ↓

     EF CORE
        ↓

PK ↔ FK RELATIONSHIP

        ↓

DATABASE

Blogs.Id
   ▲
   │
Posts.BlogId


RELATIONSHIP TYPES

One Blog ───────→ Many Posts
One Entity ─────→ One Entity
Many Entities ←→ Many Entities


USAGE

Relationship
     ↓
Query Related Data
     ↓
Track Entity Graph
     ↓
Relationship Fixup
     ↓
SaveChanges()
```

> 🎯 **Core takeaway:** An EF Core relationship is the bridge between **navigation properties in the C# object model** and **primary-key/foreign-key relationships in the database**. Once mapped, EF Core uses that relationship for querying, graph tracking, relationship fixup, seeding, and saving changes.