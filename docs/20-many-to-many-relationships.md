# 🧰 EF Core Interview Toolbox — Topic 19: Many-to-Many Relationships

> **Category:** 🏗️ Creating Model  
> **File Name:** `19-many-to-many-relationships.md`

---

# 🔗 Many-to-Many Relationships

## 🔎 Core Concept

A **many-to-many relationship** means:

```text
Many Posts
    ↕
Many Tags
```

Example:

```text
Post 1 ───┬── Tag 1
          └── Tag 2

Post 2 ───┬── Tag 1
          └── Tag 3
```

- A `Post` can have many `Tags`.
- A `Tag` can belong to many `Posts`.

---

# 🧠 Why a Join Entity/Table Is Needed

A relational database cannot represent many-to-many relationships using only a single foreign key.

Instead:

```text
Posts
   │
   │
   ▼
PostTag   ← Join Table
   ▲
   │
   │
Tags
```

The join table contains two foreign keys:

```text
PostTag
├── PostsId → Posts.Id
└── TagsId  → Tags.Id
```

Each row represents **one association**:

```text
PostsId = 1
TagsId  = 3

→ Post 1 is associated with Tag 3
```

---

# 🗄️ Database Mental Model

```text
┌──────────────┐
│    Posts     │
│--------------│
│ Id (PK)      │
└──────┬───────┘
       │
       │ FK
       ▼
┌────────────────────┐
│      PostTag       │
│--------------------│
│ PostsId (FK)       │
│ TagsId  (FK)       │
│ PK = PostsId+TagsId│
└─────────┬──────────┘
          │
          │ FK
          ▼
┌──────────────┐
│     Tags     │
│--------------│
│ Id (PK)      │
└──────────────┘
```

> ⭐ A join-table row represents **one Post ↔ Tag association**.

---

# 🧩 EF Core Mapping Options

There are two main ways to represent the join entity in EF Core.

```text
Many-to-Many
     │
     ├── Simple mapping
     │      ↓
     │   EF hides join entity
     │
     └── Explicit mapping
            ↓
        Create PostTag class
```

---

# 1️⃣ Basic Many-to-Many

The simplest model uses only collection navigations.

```csharp
public class Post
{
    public int Id { get; set; }

    public List<Tag> Tags { get; } = [];
}

public class Tag
{
    public int Id { get; set; }

    public List<Post> Posts { get; } = [];
}
```

EF Core discovers this relationship by convention.

Equivalent Fluent API:

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Post>()
        .HasMany(e => e.Tags)
        .WithMany(e => e.Posts);
}
```

### 🧠 Mental Model

```text
Post
 └── Tags
      ↕
     Tag
 └── Posts
```

EF Core manages the join table behind the scenes.

> ⭐ This is what is typically meant by a simple EF Core **many-to-many relationship**.

---

# 🔀 Skip Navigations

The direct navigations:

```csharp
Post.Tags
Tag.Posts
```

are called **skip navigations**.

Why?

```text
Post
 │
 │ Skip
 ▼
PostTag
 │
 │ Skip
 ▼
Tag
```

They allow you to work directly with the other entity while EF Core manages the join entity internally.

```text
Post.Tags
     ↓
Join Entity (hidden)
     ↓
Tag
```

---

# 🧱 2️⃣ Explicit Join Entity

Sometimes the join table itself matters.

Create a CLR class:

```csharp
public class PostTag
{
    public int PostId { get; set; }
    public int TagId { get; set; }
}
```

Then configure:

```csharp
modelBuilder.Entity<Post>()
    .HasMany(e => e.Tags)
    .WithMany(e => e.Posts)
    .UsingEntity<PostTag>();
```

EF Core automatically recognizes:

```text
PostId → Post.Id
TagId  → Tag.Id
```

and configures them as a **composite primary key**:

```text
PostId + TagId
        ↓
Composite PK
```

---

# ⭐ Why Use an Explicit Join Entity?

A dedicated join class is useful when:

- The join entity needs its own navigations.
- The join table contains additional data (**payload**).
- You need direct control over the join entity.

Example payload:

```text
PostTag
├── PostId
├── TagId
└── CreatedAt
```

> 💡 **Payload** means additional data stored in the join table besides the foreign keys.

---

# 🔗 Navigations to the Join Entity

You can have both:

```text
Skip Navigations
Post.Tags
Tag.Posts

        +

Join Entity Navigations
Post.PostTags
Tag.PostTags
```

Example:

```csharp
public class Post
{
    public int Id { get; set; }

    public List<Tag> Tags { get; } = [];
    public List<PostTag> PostTags { get; } = [];
}

public class Tag
{
    public int Id { get; set; }

    public List<Post> Posts { get; } = [];
    public List<PostTag> PostTags { get; } = [];
}

public class PostTag
{
    public int PostId { get; set; }
    public int TagId { get; set; }
}
```

### 🧠 Best of Both Worlds

```text
Post
 │
 ├── Tags
 │     ↓
 │   Simple many-to-many access
 │
 └── PostTags
       ↓
   Direct join-entity access
```

> ⭐ Skip navigations provide natural many-to-many access, while join-entity navigations provide more control over the join records.

---

# 🏷️ Custom Join Table Name

The join table can be given an explicit name.

```csharp
modelBuilder.Entity<Post>()
    .HasMany(e => e.Tags)
    .WithMany(e => e.Posts)
    .UsingEntity("PostsToTagsJoinTable");
```

```text
Default
→ PostTag

Custom
→ PostsToTagsJoinTable
```

Only the join-table name changes; the relationship remains the same.

---

# 🏷️ Custom Join Foreign Key Names

You can change the names of the FK properties or their database columns.

### Option 1 — Change Join Entity Property Names

```csharp
modelBuilder.Entity<Post>()
    .HasMany(e => e.Tags)
    .WithMany(e => e.Posts)
    .UsingEntity(
        r => r.HasOne<Tag>()
            .WithMany()
            .HasForeignKey("TagForeignKey"),
        l => l.HasOne<Post>()
            .WithMany()
            .HasForeignKey("PostForeignKey"));
```

```text
PostForeignKey → Post
TagForeignKey  → Tag
```

### Option 2 — Keep EF Property Names, Change DB Columns

```csharp
modelBuilder.Entity<Post>()
    .HasMany(e => e.Tags)
    .WithMany(e => e.Posts)
    .UsingEntity(
        j =>
        {
            j.Property("PostsId")
                .HasColumnName("PostForeignKey");

            j.Property("TagsId")
                .HasColumnName("TagForeignKey");
        });
```

```text
EF Property     Database Column
────────────    ───────────────
PostsId    →    PostForeignKey
TagsId     →    TagForeignKey
```

---

# ⚙️ Fully Explicit Join Configuration

You can configure:

- Join entity name.
- Both foreign keys.
- Principal keys.
- Composite primary key.

Example:

```csharp
modelBuilder.Entity<Post>()
    .HasMany(e => e.Tags)
    .WithMany(e => e.Posts)
    .UsingEntity(
        "PostTag",
        r => r.HasOne(typeof(Tag))
            .WithMany()
            .HasForeignKey("TagsId")
            .HasPrincipalKey(nameof(Tag.Id)),
        l => l.HasOne(typeof(Post))
            .WithMany()
            .HasForeignKey("PostsId")
            .HasPrincipalKey(nameof(Post.Id)),
        j => j.HasKey("PostsId", "TagsId"));
```

> ⚠️ Do not fully configure everything unless necessary. EF Core conventions already handle much of the mapping, and excessive configuration makes the model harder to maintain and easier to get wrong.

---

# 🧠 Convention-Based Schema

A simple many-to-many:

```csharp
public class Post
{
    public int Id { get; set; }
    public List<Tag> Tags { get; } = [];
}

public class Tag
{
    public int Id { get; set; }
    public List<Post> Posts { get; } = [];
}
```

is conventionally mapped to:

```text
Posts
Tags
PostTag
```

with:

```text
PostTag
├── PostsId → Posts.Id
├── TagsId  → Tags.Id
└── PK(PostsId, TagsId)
```

and cascade delete relationships from the join table to both sides are part of the conventional relational mapping shown here.

---

# 🧠 Many-to-Many Mental Model

```text
              MANY-TO-MANY

      ┌──────────────┐
      │     Post     │
      └──────┬───────┘
             │
          MANY│
             ▼
       ┌───────────┐
       │  PostTag  │
       │ JOIN ENTITY
       └─────┬─────┘
             ▲
          MANY│
             │
      ┌──────┴───────┐
      │     Tag      │
      └──────────────┘
```

Conceptually:

```text
Post
  ↕
Skip Navigation
  ↕
Tag
```

Physically in the database:

```text
Post → PostTag ← Tag
```

---

# 🆚 Simple vs Explicit Join Entity

| Feature | Simple Many-to-Many | Explicit Join Entity |
|---|---|---|
| CLR join class | ❌ | ✅ |
| Join table | ✅ Managed by EF | ✅ Explicit |
| Skip navigations | ✅ | ✅ |
| Direct join access | ❌ | ✅ |
| Payload columns | Limited | ✅ |
| Configuration complexity | Low | Higher |

---

# ⭐ Key Rules to Remember

```text
1. Many-to-many means many entities on both sides
   can be associated with each other.

2. Relational databases represent this using a join table.

3. The join table contains foreign keys to both sides.

4. Each join-table row represents one association.

5. EF Core can hide the join entity for simple
   many-to-many mappings.

6. Direct collection navigations such as
   Post.Tags and Tag.Posts are called skip navigations.

7. HasMany().WithMany() is the basic Fluent API pattern.

8. UsingEntity() configures the join entity/table.

9. An explicit join class is useful when the join entity
   needs navigations or payload data.

10. PostId + TagId are commonly configured as a
    composite primary key for the join entity.

11. The join table name can be customized.

12. Join FK property/column names can be customized.

13. Skip navigations and navigations to the join entity
    can exist together.

14. Avoid unnecessary full explicit configuration;
    EF Core conventions already configure many details.

15. Many-to-many supports many additional configurations,
    including alternate keys, composite keys, payload,
    unidirectional relationships, and self-referencing
    relationships.
```

---

# ⚡ 30-Second Revision

```text
MANY-TO-MANY

Post
 ↕
Tag

        ↓

Database needs JOIN TABLE

Post
  ↓
PostTag
  ↓
Tag


SIMPLE EF CORE

Post.Tags
Tag.Posts

      ↓

EF manages Join Entity
      ↓
Skip Navigations


EXPLICIT JOIN

Post
 ├── Tags
 └── PostTags

Tag
 ├── Posts
 └── PostTags

PostTag
 ├── PostId
 └── TagId


FLUENT API

HasMany()
   ↓
WithMany()
   ↓
UsingEntity()
```

> 🎯 **Core takeaway:** A many-to-many relationship is represented in the database through a **join table** containing foreign keys to both sides. EF Core can manage that join entity transparently through **skip navigations**, or you can expose an explicit join entity when you need payload data or greater control.