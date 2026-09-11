# 🧰 EF Core Interview Toolbox — Topic 21: Foreign and Principal Keys in Relationships

> **Category:** 🏗️ Creating Model  
> **File Name:** `21-foreign-and-principal-keys.md`

---

# 🔑 Foreign & Principal Keys

## 🔎 Core Concept

EF Core relationships connect:

```text
PRINCIPAL ENTITY
      │
      │ Principal Key
      ▼
DEPENDENT ENTITY
      │
      │ Foreign Key
      ▼
   Relationship
```

- In **one-to-many** and **one-to-one** relationships, the dependent contains a **foreign key (FK)** that references a **principal key**.
- The principal key is usually the **primary key**, but it can also be an **alternate key**.
- A many-to-many relationship is internally composed of **two one-to-many relationships** through its join entity. :contentReference[oaicite:0]{index=0}

---

# 🧠 Principal vs Foreign Key

```text
Blog
 └── Id
      ↓
   Principal Key
      ↑
      │ referenced by
      │
Post
 └── BlogId
      ↓
   Foreign Key
```

### Example

```csharp
public class Blog
{
    public int Id { get; set; }
}

public class Post
{
    public int Id { get; set; }

    public int BlogId { get; set; }

    public Blog Blog { get; set; } = null!;
}
```

```text
Blog.Id
  ↓
Principal Key

Post.BlogId
  ↓
Foreign Key
```

---

# 🛠️ Configuring Foreign Keys

EF Core can discover foreign keys by convention.

When conventions are not enough, use:

```csharp
.HasForeignKey(...)
```

### Single FK — Lambda Expression

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasForeignKey(e => e.BlogId);
```

```text
Blog
 ↓
HasMany(Posts)
 ↓
WithOne(Blog)
 ↓
Post.BlogId = FK
```

---

# 🔗 Composite Foreign Key

A foreign key can consist of multiple properties.

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasForeignKey(e => new
    {
        e.ContainingBlogId1,
        e.ContainingBlogId2
    });
```

```text
Post
 ├── ContainingBlogId1
 └── ContainingBlogId2
          ↓
   Composite Foreign Key
          ↓
        Blog
```

> ⭐ Composite FK properties correspond to a composite principal key.

---

# 📝 String-Based Foreign Keys

`HasForeignKey()` can also receive property names as strings.

### Single Property

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasForeignKey("ContainingBlogId");
```

### Composite FK

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasForeignKey(
        "ContainingBlogId1",
        "ContainingBlogId2");
```

### Why Use Strings?

Useful when:

```text
FK Property
   │
   ├── Private
   ├── Shadow Property
   └── Name determined dynamically
```

> ⭐ String-based configuration is especially useful for **shadow foreign keys**. :contentReference[oaicite:1]{index=1} :contentReference[oaicite:2]{index=2}

---

# ✅ Required vs Optional Foreign Keys

By convention, FK nullability determines whether a relationship is required or optional.

```text
Foreign Key
     │
 ┌───┴────┐
 │        │
int      int?
 │        │
 ▼        ▼
Required Optional
```

### Required

```csharp
public int BlogId { get; set; }
```

```text
BlogId
  ↓
Cannot be NULL
  ↓
Required relationship
```

### Optional

```csharp
public int? BlogId { get; set; }
```

```text
BlogId
  ↓
Can be NULL
  ↓
Optional relationship
```

---

# ⚠️ Nullable FK Can Still Be Required

A nullable FK property can be configured as **required**.

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasForeignKey(e => e.BlogId)
    .IsRequired();
```

Or, if EF already discovered the FK:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .IsRequired();
```

The result:

```text
CLR Property
BlogId = int?
     ↓
EF Configuration
IsRequired()
     ↓
Database FK Column
NON-NULLABLE
```

> ⭐ **Important:** CLR property nullability and database relationship requiredness do not always have to be identical; explicit EF configuration can make a nullable FK required. :contentReference[oaicite:3]{index=3}

You can also configure the FK property itself:

```csharp
modelBuilder.Entity<Post>()
    .Property(e => e.BlogId)
    .IsRequired();
```

---

# 🌫️ Shadow Foreign Keys

A **shadow FK** exists in the EF Core model but not in the CLR class.

```csharp
public class Post
{
    public int Id { get; set; }

    public Blog Blog { get; set; } = null!;
}
```

There is no:

```csharp
public int BlogId { get; set; }
```

Yet EF Core can create:

```text
Post
 ├── Id
 ├── Blog
 └── BlogId ← Shadow FK
```

### Database

```text
Posts
─────────────
Id
BlogId  ← FK
```

---

# 💡 Why Use Shadow Foreign Keys?

- Keeps relational details out of the domain model.
- Lets the entity work through navigation properties.
- EF Core maintains the FK internally.

```text
Post.Blog
   ↓
Navigation
   ↓
EF Core
   ↓
Shadow BlogId
   ↓
Database FK
```

### Trade-off

```text
Explicit FK
   ↓
BlogId available for serialization / DTOs

Shadow FK
   ↓
BlogId hidden from CLR entity
```

> 💡 A private FK can be a useful compromise when the FK should not be publicly exposed but still needs to travel with the entity. :contentReference[oaicite:4]{index=4}

---

# 🛠️ Creating a Shadow FK Explicitly

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasForeignKey("MyBlogId");
```

If `"MyBlogId"` is not an existing CLR property:

```text
MyBlogId
   ↓
Shadow FK created
```

You can configure it separately:

```csharp
modelBuilder.Entity<Post>()
    .Property<string>("MyBlogId")
    .IsRequired();
```

```text
Shadow FK
   ↓
Configure type / nullability / facets
```

---

# 🧠 Shadow FK Defaults

- The shadow FK type is inferred from the **principal key type**.
- It is nullable unless the relationship is configured as required.
- It can inherit facets such as maximum length and Unicode settings from the principal key.

```text
Principal Key
      ↓
   Key Type
      ↓
Shadow FK Type
```

---

# 🚨 Detecting Accidental Shadow Properties

A typo in an FK name can accidentally create a shadow property.

EF Core can be configured to throw when a shadow property is created:

```csharp
protected override void OnConfiguring(
    DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.ConfigureWarnings(
        b => b.Throw(
            CoreEventId.ShadowPropertyCreated));
}
```

```text
Typo in Property Name
        ↓
Shadow Property Created
        ↓
Exception
```

> ⭐ Useful when you want accidental shadow properties to fail fast. :contentReference[oaicite:5]{index=5}

---

# 🏷️ Foreign Key Constraint Names

By convention, relational FK constraints are named:

```text
FK_<DependentType>_<PrincipalType>_<ForeignKeyProperty>
```

Example:

```text
FK_Post_Blog_BlogId
```

For composite FKs:

```text
FK_<Dependent>_<Principal>_<FK1>_<FK2>
```

You can customize the constraint name:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasForeignKey(e => e.BlogId)
    .HasConstraintName("My_BlogId_Constraint");
```

> 💡 The constraint name is mainly relevant when EF Core creates the database schema through **Migrations**; the name itself is not used by the EF runtime for relationship behavior. :contentReference[oaicite:6]{index=6}

---

# 🚫 Excluding FK Constraints From Migrations

For scenarios such as legacy databases or specialized synchronization/import workflows, the relationship can remain in the EF model while its FK constraint is excluded from schema creation.

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasForeignKey(e => e.BlogId)
    .ExcludeForeignKeyFromMigrations();
```

```text
EF Model
   │
   ├── Relationship ✅
   ├── Navigation ✅
   ├── Querying ✅
   └── Database FK Constraint ❌
```

This means:

- The relationship remains part of the EF model.
- Navigations and change tracking still work.
- Queries can still use the FK.
- The FK constraint is not generated by migrations / `EnsureCreated`.
- The FK index can still be created.

> ⚠️ This API is available in **EF Core 11**. :contentReference[oaicite:7]{index=7}

---

# 🌐 Excluding All FK Constraints

FK constraints can also be excluded globally:

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    foreach (var foreignKey in modelBuilder.Model
        .GetEntityTypes()
        .SelectMany(e => e.GetForeignKeys()))
    {
        foreignKey.SetIsExcludedFromMigrations(true);
    }
}
```

```text
All Entity Types
      ↓
All Foreign Keys
      ↓
Exclude From Migrations
```

---

# 📈 FK Indexes

EF Core creates an index for FK properties by convention.

```text
Post.BlogId
     ↓
Foreign Key
     +
Index
```

Why?

```text
Query using FK
     ↓
Index Lookup
     ↓
Faster access
```

> ⭐ FK indexing is a convention and can be customized as part of model configuration. :contentReference[oaicite:8]{index=8}

---

# 🔑 Principal Keys

By convention:

```text
Foreign Key
     ↓
Principal Primary Key
```

Example:

```text
Post.BlogId
     ↓
Blog.Id
```

But EF Core can use an **alternate key** instead.

```text
Post.AlternateId
        ↓
Blog.AlternateId
        ↑
 Alternate Key
```

---

# 🔄 Using an Alternate Principal Key

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasPrincipalKey(e => e.AlternateId);
```

Composite principal key:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasPrincipalKey(e => new
    {
        e.AlternateId1,
        e.AlternateId2
    });
```

String form:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasPrincipalKey("AlternateId");
```

Composite string form:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne(e => e.Blog)
    .HasPrincipalKey(
        "AlternateId1",
        "AlternateId2");
```

---

# ⭐ Key Ordering Rule

For composite keys:

```text
Principal Key Order
        ↕
Foreign Key Order
```

must match.

Example:

```text
Principal:
(AlternateId1, AlternateId2)

Foreign:
(AlternateId1, AlternateId2)
```

> ⚠️ The key-property order used by the relationship must match between the principal and foreign key definitions. It does not have to match the order of properties in the CLR class or table columns. :contentReference[oaicite:9]{index=9}

---

# 🧩 `HasPrincipalKey()` Automatically Creates an Alternate Key

If the specified principal properties are not the primary key:

```csharp
.HasPrincipalKey(e => e.AlternateId)
```

EF Core automatically introduces the alternate key.

You do **not** need to separately call:

```csharp
.HasAlternateKey(...)
```

unless you need additional configuration, such as customizing the constraint name.

```text
HasPrincipalKey(AlternateId)
          ↓
AlternateId becomes Principal Key
          ↓
EF creates required alternate-key metadata
```

:contentReference[oaicite:10]{index=10}

---

# 🗝️ Foreign Keys in Many-to-Many Relationships

In many-to-many relationships, the FK properties live on the **join entity**.

```text
Post
  │
  │ PK
  ▼
PostTag
 ├── PostId → Post.Id
 └── TagId  → Tag.Id
  ▲
  │
  │ PK
  ▼
Tag
```

```text
Many-to-Many

Post
  ↕
PostTag
  ↕
Tag
```

The same FK concepts apply:

- FK configuration.
- Principal key configuration.
- FK constraint names.
- Composite keys.
- Cascade behavior.

Example:

```csharp
modelBuilder.Entity<Post>()
    .HasMany(e => e.Tags)
    .WithMany(e => e.Posts)
    .UsingEntity(
        l => l.HasOne<Tag>()
            .WithMany()
            .HasConstraintName(
                "TagForeignKey_Constraint"),
        r => r.HasOne<Post>()
            .WithMany()
            .HasConstraintName(
                "PostForeignKey_Constraint"));
```

:contentReference[oaicite:11]{index=11}

---

# 🚫 Keyless Entities as Principals

A relationship needs:

```text
Foreign Key
      ↓
Principal Key
```

Therefore:

> ❌ A **keyless entity cannot be the principal** because it has no primary or alternate key to reference.

A keyless entity can still be a **dependent** if it has an FK pointing to a keyed entity.

```text
Post
 └── Id
      ↑
      │
Tag
 ├── PostId
 └── No Primary Key
```

```text
Post → Principal ✅
Tag  → Dependent ✅
```

:contentReference[oaicite:12]{index=12}

---

# ⚠️ No "Alternate Key Only"

A type cannot simply be:

```text
No Primary Key
      +
Alternate Key
      ↓
❌ Not valid as a principal
```

The entity must still have a primary key when it needs to act as a principal in an EF Core relationship.

```text
Entity
 ├── Primary Key
 └── Alternate Key(s)
```

The alternate key can be selected as the **principal key** of a relationship.

---

# 🧠 Complete Mental Model

```text
                 RELATIONSHIP
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
      PRINCIPAL               DEPENDENT
          │                       │
      PK / AK                     FK
          │                       │
          └──────────┬────────────┘
                     ▼
                FK Constraint
                     │
                     ▼
                  Database


CONFIGURATION

HasForeignKey()
      ↓
Which property is the FK?

HasPrincipalKey()
      ↓
Which key does the FK reference?

IsRequired()
      ↓
Can FK be NULL?

HasConstraintName()
      ↓
What is the DB FK constraint called?


SHADOW FK

No CLR FK property
      ↓
EF creates shadow FK
      ↓
Change Tracker / Model
      ↓
Database FK Column


MANY-TO-MANY

Post
  ↕
Join Entity
  ↕
Tag

Join Entity owns both FKs.
```

---

# ⭐ Key Rules to Remember

```text
1. One-to-one and one-to-many relationships are based on
   FK → Principal Key.

2. The principal key is usually the primary key.

3. An alternate key can also be the principal key.

4. HasForeignKey() identifies the dependent FK.

5. HasForeignKey() supports single and composite FKs.

6. String-based HasForeignKey() is useful for private,
   shadow, or dynamically named properties.

7. Non-nullable FK → required relationship by convention.

8. Nullable FK → optional relationship by convention.

9. IsRequired() can make a nullable FK required.

10. Shadow FK exists in the EF model but not in the CLR class.

11. Shadow FK type is inferred from the principal key type.

12. EF creates FK indexes by convention.

13. HasConstraintName() customizes the relational FK
    constraint name.

14. FK constraint names mainly matter during schema creation /
    migrations.

15. ExcludeForeignKeyFromMigrations() keeps the relationship
    in EF while excluding the database FK constraint.

16. HasPrincipalKey() can make an alternate key the
    principal key of a relationship.

17. Composite FK and principal-key property order must match.

18. HasPrincipalKey() automatically introduces an alternate key
    when the selected property is not the primary key.

19. Many-to-many relationships use two FKs on the join entity.

20. Keyless entities cannot be principals because they have
    no key to reference.

21. Keyless entities can still act as dependents.

22. A keyless entity cannot be made into an "alternate-key-only"
    principal.
```

---

# ⚡ 30-Second Revision

```text
PRINCIPAL
   │
   └── PK / Alternate Key
             ▲
             │
             │ referenced by
             │
DEPENDENT
   │
   └── FK


FLUENT API

HasForeignKey()
→ Defines FK

HasPrincipalKey()
→ Defines principal key

IsRequired()
→ Controls requiredness

HasConstraintName()
→ Names FK constraint


SHADOW FK

No FK property in C#
        ↓
EF creates shadow property
        ↓
FK still exists in DB


COMPOSITE KEYS

Principal:
(A, B)

Foreign:
(A, B)

→ Order must match


MANY-TO-MANY

Post
 ↓
Join Entity
 ↓
Tag

Join Entity
├── Post FK
└── Tag FK
```

> 🎯 **Core takeaway:** EF Core relationships are fundamentally a **Foreign Key → Principal Key** mapping. Once you understand **who is principal, who is dependent, which properties form the FK, which key is referenced, and whether the FK is required or optional**, the rest of relationship configuration becomes much easier.