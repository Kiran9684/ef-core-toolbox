# 🧰 EF Core Interview Toolbox — Topic 23: Relationship Discovery Conventions

> **Category:** 🏗️ Creating Model  
> **File Name:** `23-relationship-discovery-conventions.md`

---

# 🔎 Relationship Discovery by Convention

EF Core uses **conventions** to automatically discover and configure relationships from entity classes.

```text
Entity Classes
      ↓
EF Core Conventions
      ↓
Discover Navigations
      ↓
Pair Navigations
      ↓
Discover Foreign Keys
      ↓
Determine Cardinality
      ↓
Build Relationship
```

> ⭐ Conventions can always be overridden with **mapping attributes** or the **Fluent API / ModelBuilder**. :contentReference[oaicite:0]{index=0}

---

# 🧭 1. Discovering Reference Navigations

A property can be discovered as a **reference navigation** when:

- It is `public`.
- It has a getter and setter.
- The setter can be `private` or another accessibility.
- An `init` setter is also valid.
- Its type is, or could be, an entity type.
- It is not static.
- It is not an indexer.

```csharp
public class Blog
{
    public int Id { get; set; }

    public Author? Author { get; private set; }  // ✅ Reference navigation
}
```

These are not discovered as reference navigations:

```csharp
public int Id { get; set; }                      // Primitive
public string Title { get; set; } = null!;       // Primitive
public Uri? Uri { get; set; }                    // Automatically converted
public ConsoleKeyInfo Info { get; set; }         // Value type
public Author DefaultAuthor => new();            // No setter
```

```text
Reference Navigation Candidate
            │
     ┌──────┼──────┐
     │      │      │
   Public  Setter  Entity-like Type
     │      │      │
     └──────┴──────┘
            ↓
     Reference Navigation
```

> 💡 A private setter or `init` setter is enough; the setter does not have to be public. :contentReference[oaicite:1]{index=1} :contentReference[oaicite:2]{index=2}

---

# 📦 2. Discovering Collection Navigations

A property can be discovered as a **collection navigation** when:

- It is `public`.
- It has a getter.
- The property type implements `IEnumerable<TEntity>`.
- `TEntity` is, or could be, an entity type.
- It is not static.
- It is not an indexer.

A setter is **not required**.

```csharp
public class Blog
{
    public int Id { get; set; }

    public List<Tag> Tags { get; set; } = [];
}

public class Tag
{
    public int Id { get; set; }

    public IEnumerable<Blog> Blogs { get; } = new List<Blog>();
}
```

Both are collection navigations:

```text
Blog.Tags
   ↓
Collection Navigation

Tag.Blogs
   ↓
Collection Navigation
```

> ⭐ Collection navigations represent the **many** side of relationships. :contentReference[oaicite:3]{index=3}

---

# 🔗 3. Pairing Navigations

After EF Core discovers a navigation from `A → B`, it checks whether an inverse navigation exists from `B → A`.

```text
A ───── Navigation ─────→ B
A ←──── Inverse ────────── B
            ↓
      Paired Relationship
```

If the inverse is found, both navigations form **one bidirectional relationship**.

### Relationship Type

```text
Navigation Types
       │
       ├── Collection + Reference
       │       ↓
       │   One-to-Many
       │
       ├── Reference + Reference
       │       ↓
       │   One-to-One
       │
       └── Collection + Collection
               ↓
           Many-to-Many
```

:contentReference[oaicite:4]{index=4}

### Examples

```text
Blog.Posts  +  Post.Blog
      ↓
One-to-Many


Blog.Author + Author.Blog
      ↓
One-to-One


Post.Tags + Tag.Posts
      ↓
Many-to-Many
```

---

# ⚠️ Multiple Relationships Between the Same Types

Navigation pairing by convention works only when there is **one relationship between two types**.

```text
Order ──→ User
  │
  └──────→ User
```

If multiple relationships exist between the same two entity types:

> ❌ EF Core cannot safely infer which navigation belongs to which relationship.

They must be configured explicitly.

```text
CreatedById → CreatedBy
ApprovedById → ApprovedBy
```

```csharp
modelBuilder.Entity<Order>()
    .HasOne(o => o.CreatedBy)
    .WithMany(u => u.CreatedOrders)
    .HasForeignKey(o => o.CreatedById);
```

```csharp
modelBuilder.Entity<Order>()
    .HasOne(o => o.ApprovedBy)
    .WithMany(u => u.ApprovedOrders)
    .HasForeignKey(o => o.ApprovedById);
```

> ⭐ **Multiple relationships between the same entity types require explicit configuration.** :contentReference[oaicite:5]{index=5}

---

# 🔑 4. Discovering Foreign Key Properties

Once the relationship/navigation is known, EF Core looks for a suitable foreign key.

A property is considered a matching FK when:

- Its type is compatible with the principal primary/alternate key.
- Its name follows an FK naming convention.

Compatible types mean:

```text
Same Type
OR
Nullable Version of Principal Key Type
```

Example:

```text
Principal Key
int

Compatible FK
int
int?
```

:contentReference[oaicite:6]{index=6}

---

# 🏷️ Foreign Key Naming Conventions

EF Core looks for these patterns:

```text
1. <NavigationName><PrincipalKeyName>

2. <NavigationName>Id

3. <PrincipalTypeName><PrincipalKeyName>

4. <PrincipalTypeName>Id
```

Example:

```text
Principal:
Blog.Key

Navigation:
TheBlog
```

Possible FK names:

```text
TheBlogKey
TheBlogId
BlogKey
BlogId
```

> 💡 The `Id` suffix is case-insensitive: `Id`, `ID`, `id`, etc. are recognized. :contentReference[oaicite:7]{index=7}

---

# 🧠 FK Discovery Example

```csharp
public class Blog
{
    public int Key { get; set; }

    public ICollection<Post> Posts { get; }
        = new List<Post>();
}

public class Post
{
    public int Id { get; set; }

    public int? TheBlogKey { get; set; }

    public Blog? TheBlog { get; set; }
}
```

```text
Post.TheBlog
     ↓
Navigation Name = TheBlog

Blog.Key
     ↓
Principal Key Name = Key

TheBlog + Key
     ↓
TheBlogKey
     ↓
✅ FK discovered
```

Other valid convention examples:

```text
TheBlogID
BlogKey
Blogid
```

can also match the supported naming patterns. :contentReference[oaicite:8]{index=8}

---

# 🧩 Dependent Primary Key as Foreign Key

If the dependent end has been explicitly configured and its primary key is compatible with the principal key, EF Core can also use the **dependent primary key as the foreign key**.

```text
Dependent PK
     ↓
Compatible with Principal Key
     ↓
Can also act as FK
```

This pattern is especially relevant to one-to-one relationships. :contentReference[oaicite:9]{index=9}

---

# 🧱 Composite Foreign Keys

The same rules apply to composite foreign keys.

```text
Principal Key
(A, B)
   ↑
   │
Dependent FK
(A, B)
```

Each FK property must:

- Have a compatible type with the corresponding principal-key property.
- Match one of the supported naming conventions.

> ⭐ Composite FK properties correspond **position-by-position** with the principal key properties. :contentReference[oaicite:10]{index=10}

---

# 🔢 5. Determining Relationship Cardinality

**Cardinality** describes how many entities can participate in the relationship.

```text
1 : 1  → One-to-One
1 : N  → One-to-Many
M : N  → Many-to-Many
```

EF Core uses:

```text
Navigations
   +
Foreign Keys
   +
Principal / Dependent Ends
   ↓
Cardinality
```

---

# 1️⃣ One Unpaired Reference Navigation

Example:

```csharp
public class Post
{
    public int Id { get; set; }

    public Blog Blog { get; set; } = null!;
}

public class Blog
{
    public int Id { get; set; }
}
```

EF interprets:

```text
Post.Blog
   ↓
Unpaired Reference Navigation
   ↓
Post = Dependent
Blog = Principal
   ↓
Unidirectional One-to-Many
```

Even though `Blog.Posts` does not exist:

```text
Blog
  ↓
Many Posts

Post
  ↓
One Blog
```

EF can infer a `BlogId` FK. :contentReference[oaicite:11]{index=11}

---

# 2️⃣ One Unpaired Collection Navigation

Example:

```csharp
public class Blog
{
    public int Id { get; set; }

    public ICollection<Post> Posts { get; set; }
        = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
}
```

EF interprets:

```text
Blog.Posts
   ↓
Unpaired Collection Navigation
   ↓
Blog = Principal
Post = Dependent
   ↓
Unidirectional One-to-Many
```

```text
Blog
  └── Posts
        ↓
      Post
```

:contentReference[oaicite:12]{index=12}

---

# 3️⃣ Paired Collection + Reference

Example:

```text
Blog.Posts
    +
Post.Blog
    ↓
Bidirectional One-to-Many
```

```text
Blog
 └── Posts
       ↕
      Post
       └── Blog
```

The collection side is the **principal end** and the reference-navigation side is the **dependent end**. :contentReference[oaicite:13]{index=13}

---

# 4️⃣ Paired Reference + Reference

If both sides have reference navigations:

```text
Blog.BlogImage
      +
BlogImage.Blog
      ↓
Potential One-to-One
```

EF must determine which side is the dependent.

### FK Found on One Side

```text
BlogImage.BlogId
      ↓
FK discovered
      ↓
BlogImage = Dependent
Blog      = Principal
```

```csharp
public class BlogImage
{
    public int BlogImageId { get; set; }

    public int BlogId { get; set; }

    public Blog Blog { get; set; } = null!;
}
```

```text
Blog
  ↕
BlogImage
  ↑
  └── BlogId = FK
```

EF can therefore establish the one-to-one relationship. :contentReference[oaicite:14]{index=14}

### No FK Found

```text
Blog.BlogImage
      ↕
BlogImage.Blog

No FK on either side
      ↓
🤯 Principal/Dependent cannot be determined
      ↓
❌ Exception
```

Configure it explicitly:

```csharp
modelBuilder.Entity<Blog>()
    .HasOne(b => b.BlogImage)
    .WithOne(i => i.Blog)
    .HasForeignKey<BlogImage>(i => i.BlogId);
```

> ⭐ In a one-to-one relationship, if EF cannot determine the dependent end from an FK, explicitly specify it with `HasForeignKey<T>()`. :contentReference[oaicite:15]{index=15}

---

# 5️⃣ Paired Collection + Collection

```text
Post.Tags
    +
Tag.Posts
    ↓
Bidirectional Many-to-Many
```

```text
Post
  ↕
Join Entity
  ↕
Tag
```

EF conventionally creates a join entity for the relationship. :contentReference[oaicite:16]{index=16}

---

# 🔀 Many-to-Many Convention

For a simple many-to-many:

```csharp
public class Post
{
    public int Id { get; set; }

    public ICollection<Tag> Tags { get; }
        = new List<Tag>();
}

public class Tag
{
    public int Id { get; set; }

    public ICollection<Post> Posts { get; }
        = new List<Post>();
}
```

EF conventionally discovers:

```text
Post + Tag
    ↓
Join Entity = PostTag
    ↓
Join Table = PostTag
```

The join entity gets:

```text
PostsId
TagsId
```

as foreign keys and:

```text
(PostsId, TagsId)
       ↓
Composite Primary Key
```

The FK properties are non-nullable, so the relationships to the join entity are required and cascade delete is configured by convention. :contentReference[oaicite:17]{index=17}

---

# 🧠 Relationship Discovery Flow

```text
               ENTITY TYPES
                    │
                    ▼
           DISCOVER NAVIGATIONS
                    │
          ┌─────────┴─────────┐
          │                   │
      Reference           Collection
          │                   │
          └─────────┬─────────┘
                    ▼
             PAIR NAVIGATIONS
                    │
          ┌─────────┼─────────┐
          │         │         │
          ▼         ▼         ▼
       1 : N       1 : 1      M : N
                    │
                    ▼
          DISCOVER FOREIGN KEY
                    │
                    ▼
       DETERMINE PRINCIPAL / DEPENDENT
                    │
                    ▼
           BUILD RELATIONSHIP
```

---

# 🎯 Convention Cheat Sheet

| Discovered Navigations | Relationship |
|---|---|
| Reference + Collection | One-to-Many |
| Reference + Reference | One-to-One |
| Collection + Collection | Many-to-Many |
| One unpaired Reference | Unidirectional One-to-Many |
| One unpaired Collection | Unidirectional One-to-Many |

---

# 🔑 FK Convention Cheat Sheet

```text
<navigation><principal key>
<navigation>Id
<principal type><principal key>
<principal type>Id
```

Example:

```text
Navigation = TheBlog
Principal Key = Key

→ TheBlogKey
→ TheBlogId
→ BlogKey
→ BlogId
```

---

# ⭐ Key Rules to Remember

```text
1. EF Core begins relationship discovery by finding navigations.

2. Reference navigation:
   → Public
   → Getter + Setter
   → Setter can be private
   → Init-only setter is valid
   → Not static / not indexer

3. Collection navigation:
   → Public
   → Getter required
   → Setter optional
   → Must expose IEnumerable<TEntity>-compatible data

4. Paired navigations form one bidirectional relationship.

5. Reference + Collection → One-to-Many.

6. Reference + Reference → One-to-One.

7. Collection + Collection → Many-to-Many.

8. Multiple relationships between the same entity types
   must be configured explicitly.

9. EF discovers FKs after discovering/configuring navigations.

10. FK type must be compatible with the principal key type.

11. EF recognizes four main FK naming patterns:
    → Navigation + Principal Key
    → Navigation + Id
    → Principal Type + Principal Key
    → Principal Type + Id

12. In one-to-one relationships, an FK found on one side
    helps EF determine the dependent.

13. If neither side of a one-to-one has a discoverable FK,
    explicitly configure the dependent.

14. Composite FK properties follow the same compatibility
    and naming rules.

15. A single unpaired reference navigation is treated as
    a unidirectional one-to-many with the navigation on the dependent.

16. A single unpaired collection navigation is treated as
    a unidirectional one-to-many with the collection on the principal.

17. Many-to-many uses a join entity containing two FKs.

18. Many-to-many conventionally uses the two FK properties
    as a composite primary key on the join entity.

19. Required relationships cascade delete by convention;
    optional relationships do not cascade delete by convention.

20. Conventions can be overridden with Data Annotations
    or Fluent API.
```

---

# ⚡ 30-Second Revision

```text
RELATIONSHIP DISCOVERY

1. Find Navigations
       ↓
2. Pair Inverses
       ↓
3. Determine Cardinality
       ↓
4. Find FK
       ↓
5. Determine Principal / Dependent


NAVIGATION PAIRING

Reference + Collection
        ↓
      1 : N

Reference + Reference
        ↓
      1 : 1

Collection + Collection
        ↓
      M : N


FK DISCOVERY

Navigation + Key
Navigation + Id
Principal + Key
Principal + Id


IMPORTANT

One-to-One + no FK discovered
        ↓
EF cannot know dependent
        ↓
Use HasForeignKey<T>()


Many-to-Many
        ↓
Join Entity
        ↓
Two FKs
        ↓
Composite PK
```

> 🎯 **Core takeaway:** EF Core relationship discovery follows a predictable chain: **discover navigations → pair them → determine cardinality → discover foreign keys → determine principal/dependent ends**. Understanding these conventions makes it much easier to predict when EF Core will configure a relationship automatically and when explicit Fluent API configuration is required.