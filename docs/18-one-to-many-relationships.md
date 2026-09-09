# 🧰 EF Core Interview Toolbox — Topic 17: One-to-Many Relationships

> **Category:** 🏗️ Creating Model  
> **File Name:** `17-one-to-many-relationships.md`

---

# 🔗 One-to-Many Relationships

## 🔎 Core Concept

A **one-to-many relationship** means:

```text
1 Principal
     │
     ├── Many Dependents
     ├── Many Dependents
     └── Many Dependents
```

Example:

```text
Blog
 │
 ├── Post 1
 ├── Post 2
 └── Post 3
```

- One `Blog` can have many `Post` entities.
- Each `Post` belongs to one `Blog`.
- The relationship is represented using **navigation properties** in C# and **PK/FK** in the database.

> ⭐ `Blog.Posts` and `Post.Blog` are two navigations for **one relationship**, not two relationships.

---

# 🧠 Relationship Anatomy

A one-to-many relationship consists of:

```text
PRINCIPAL / PARENT
        │
        │ Primary / Alternate Key
        ▼
DEPENDENT / CHILD
        │
        │ Foreign Key
        ▼
    DATABASE
```

Example:

```text
Blog.Id
   ▲
   │
Post.BlogId
```

Optional navigations:

```text
Blog
 └── Posts       ← Collection Navigation

Post
 └── Blog        ← Reference Navigation
```

---

# ✅ Required One-to-Many

In a **required** relationship, every dependent must have a principal.

```csharp
public class Blog
{
    public int Id { get; set; }

    public ICollection<Post> Posts { get; }
        = new List<Post>();
}

public class Post
{
    public int Id { get; set; }

    public int BlogId { get; set; }

    public Blog Blog { get; set; } = null!;
}
```

### 🧠 Why Is It Required?

```text
Post.BlogId
     ↓
int
     ↓
Cannot be NULL
     ↓
Post MUST have a Blog
```

---

# ⚠️ Required Does NOT Mean the Principal Needs a Child

This distinction is important:

```text
Required Relationship

Blog
 │
 ├── Post
 └── Post
```

means:

```text
Every Post → MUST have a Blog
```

It does **not** mean:

```text
Every Blog → MUST have at least one Post
```

A `Blog` can exist without any `Post`.

> ⭐ EF Core cannot use the relationship itself to require a principal to have a certain number of dependents. Such rules belong in **application/business logic**.

---

# 🔄 Bidirectional Relationship

When both sides contain navigations:

```text
Blog
 │
 └── Posts
        ↕
       Post
        │
        └── Blog
```

this is called a **bidirectional relationship**.

> 💡 `Blog.Posts` → collection navigation  
> `Post.Blog` → reference navigation

---

# 🧭 Convention-Based Discovery

For a simple relationship, EF Core can discover the relationship automatically.

```text
Blog
 └── Posts
       ↓
     Post
       ↓
   BlogId
```

EF Core discovers:

```text
Blog
→ Principal

Post
→ Dependent

Post.BlogId
→ Foreign Key

Blog.Posts
→ Collection Navigation

Post.Blog
→ Reference Navigation
```

Because:

```text
BlogId = int
       ↓
Non-nullable
       ↓
Required Relationship
```

---

# 🛠️ Explicit Fluent API Configuration

When conventions are not enough, configure the relationship explicitly.

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasMany(e => e.Posts)
        .WithOne(e => e.Blog)
        .HasForeignKey(e => e.BlogId)
        .IsRequired();
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
Post.BlogId
 ↓
Foreign Key
 ↓
IsRequired()
```

### Method Meaning

| Method | Meaning |
|---|---|
| `HasMany(e => e.Posts)` | One Blog has many Posts |
| `WithOne(e => e.Blog)` | Each Post has one Blog |
| `HasForeignKey(e => e.BlogId)` | `BlogId` is the FK |
| `IsRequired()` | FK/relationship is required |

> ⭐ A relationship only needs to be configured **once**. Do not configure the principal and dependent halves separately.

---

# 🟢 Optional One-to-Many

An optional relationship means the dependent can exist without a principal.

```csharp
public class Blog
{
    public int Id { get; set; }

    public ICollection<Post> Posts { get; }
        = new List<Post>();
}

public class Post
{
    public int Id { get; set; }

    public int? BlogId { get; set; }

    public Blog? Blog { get; set; }
}
```

### 🧠 Why Optional?

```text
Post.BlogId
     ↓
int?
     ↓
Can be NULL
     ↓
Post can exist without Blog
```

Configure explicitly:

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasMany(e => e.Posts)
        .WithOne(e => e.Blog)
        .HasForeignKey(e => e.BlogId)
        .IsRequired(false);
}
```

---

# 🧩 Nullable Reference Types

When C# nullable reference types are enabled:

```csharp
public int? BlogId { get; set; }

public Blog? Blog { get; set; }
```

the navigation should also be nullable.

```text
Nullable FK
   ↓
BlogId = int?
   ↓
Optional Relationship
   ↓
Blog = Blog?
```

Whereas:

```csharp
public int BlogId { get; set; }

public Blog Blog { get; set; } = null!;
```

represents:

```text
Non-nullable FK
   ↓
Required Relationship
```

> ⭐ With NRT enabled, FK and reference-navigation nullability should agree with the relationship's required/optional nature.

---

# 🌫️ Required One-to-Many with Shadow Foreign Key

A dependent does not have to declare the FK property in its CLR class.

```csharp
public class Blog
{
    public int Id { get; set; }

    public ICollection<Post> Posts { get; }
        = new List<Post>();
}

public class Post
{
    public int Id { get; set; }

    public Blog Blog { get; set; } = null!;
}
```

EF Core can create:

```text
Post
 │
 └── BlogId
       ↓
   Shadow FK
```

### 🧠 Why Use It?

- Keeps database-specific FK details out of the domain class.
- Produces a cleaner domain model.

But there is a trade-off:

```text
No CLR BlogId
      ↓
Serialized Post does not contain BlogId
```

A private FK property can be used when the value needs to travel with the entity.

---

# 🔎 Shadow FK Nullability

With **Nullable Reference Types enabled**:

```text
public Blog Blog { get; set; } = null!;
        ↓
Shadow BlogId → NON-NULL
        ↓
Required
```

```text
public Blog? Blog { get; set; }
        ↓
Shadow BlogId → NULLABLE
        ↓
Optional
```

Without NRT, shadow FKs are nullable by default unless configured otherwise.

---

# 🛠️ Configuring a Shadow Foreign Key

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .HasForeignKey("BlogId")
    .IsRequired();
```

Or from the dependent side:

```csharp
modelBuilder.Entity<Post>()
    .HasOne(p => p.Blog)
    .WithMany(b => b.Posts)
    .HasForeignKey("BlogId")
    .IsRequired();
```

```text
Post
 ↓
Shadow Property "BlogId"
 ↓
Non-nullable
 ↓
Required Relationship
```

> 💡 `HasForeignKey("BlogId")` explicitly names the shadow FK property.

---

# ↔️ One-to-Many Without Navigation to Principal

A relationship can have only one navigation.

Example:

```csharp
public class Blog
{
    public int Id { get; set; }

    public ICollection<Post> Posts { get; }
        = new List<Post>();
}

public class Post
{
    public int Id { get; set; }

    public int BlogId { get; set; }
}
```

```text
Blog
 └── Posts
       ↓
     Post

Post
 └── BlogId
       ↓
      Blog
```

There is **no `Post.Blog` navigation**.

This is a **unidirectional relationship**.

---

# 🛠️ Configure It Explicitly

From the principal side:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne()
    .HasForeignKey(p => p.BlogId)
    .IsRequired();
```

`WithOne()` with no navigation means:

```text
Post has no Blog navigation
```

Or start from the dependent:

```csharp
modelBuilder.Entity<Post>()
    .HasOne<Blog>()
    .WithMany(b => b.Posts)
    .HasForeignKey(p => p.BlogId)
    .IsRequired();
```

### 🧠 Mental Model

```text
Blog
 └── Posts
       ↓
     Post
       │
       └── BlogId

Navigation exists only on one side.
```

---

# ↔️ Querying a Unidirectional Relationship

Because `Post` has no `Blog` navigation:

```csharp
context.Posts
    .Include(p => p.Blog); // ❌ Cannot do this
```

Instead, use the FK:

```csharp
var blog = await context.Blogs
    .FindAsync(post.BlogId);
```

Or use projection/join.

You can also query from the principal side:

```csharp
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();
```

> ⭐ Without a navigation property, EF Core cannot navigate that direction through `Include()`.

---

# 🔄 Change Tracking in a Unidirectional Relationship

Changing the FK still changes the relationship:

```csharp
post.BlogId = newBlogId;
```

There is no:

```csharp
post.Blog = newBlog;
```

because the navigation does not exist.

```text
Change BlogId
     ↓
Foreign Key Changes
     ↓
Relationship Changes
```

---

# 🌫️ One-to-Many Without Navigation to Principal + Shadow FK

The dependent can have neither a navigation nor an explicit FK property.

```csharp
public class Blog
{
    public int Id { get; set; }

    public ICollection<Post> Posts { get; }
        = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
}
```

EF Core can still discover a relationship from the collection navigation and create a shadow FK.

```text
Blog
 └── Posts
       ↓
     Post
       ↓
  Shadow BlogId
```

### Default Behavior

Because there is no explicit navigation/FK indicating requiredness:

```text
Shadow FK
   ↓
Nullable by default
   ↓
Optional Relationship
```

---

# ✅ Make the Shadow Relationship Required

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne()
    .IsRequired();
```

Or explicitly name the shadow FK:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(e => e.Posts)
    .WithOne()
    .HasForeignKey("BlogId")
    .IsRequired();
```

```text
WithOne()
→ No navigation on Post

IsRequired()
→ Shadow BlogId cannot be NULL
```

---

# 🔄 One-to-Many Without Navigation to Dependents

The opposite structure is also possible.

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
Blog
 └── No Posts navigation

Post
 ├── BlogId
 └── Blog
```

The relationship exists, but only the dependent can navigate to the principal.

---

# 🛠️ Fluent API — From Dependent

```csharp
modelBuilder.Entity<Post>()
    .HasOne(e => e.Blog)
    .WithMany()
    .HasForeignKey(e => e.BlogId)
    .IsRequired();
```

### Meaning

```text
HasOne(e => e.Blog)
→ Post has one Blog

WithMany()
→ Blog has no collection navigation

HasForeignKey(e => e.BlogId)
→ BlogId is the FK

IsRequired()
→ BlogId cannot be NULL
```

---

# 🛠️ Fluent API — From Principal

Because `Blog` has no navigation to `Post`, specify the dependent type:

```csharp
modelBuilder.Entity<Blog>()
    .HasMany<Post>()
    .WithOne(e => e.Blog)
    .HasForeignKey(e => e.BlogId)
    .IsRequired();
```

```text
Blog
  │
  └── HasMany<Post>()
          ↓
        Post
          ↓
     WithOne(Blog)
```

---

# 🌳 Self-Referencing One-to-Many

A **self-referencing relationship** occurs when an entity relates to other entities of the same type.

Example:

```text
Employee
    │
    ├── Manager
    │
    └── Reports
```

One employee can manage many employees.

```text
Manager
   │
   ├── Employee A
   ├── Employee B
   └── Employee C
```

---

# 👨‍💼 Self-Referencing Example

```csharp
public class Employee
{
    public int Id { get; set; }

    public int? ManagerId { get; set; }

    public Employee? Manager { get; set; }

    public ICollection<Employee> Reports { get; }
        = new List<Employee>();
}
```

### Relationship

```text
Employee
 │
 ├── ManagerId → FK
 │
 ├── Manager   → Principal Navigation
 │
 └── Reports   → Dependent Collection
```

Because:

```text
ManagerId = int?
        ↓
Nullable
        ↓
Optional Relationship
```

This allows a top-level employee, such as a CEO, to have no manager.

---

# 🔄 Self-Referencing Mental Model

```text
Employee
   │
   └── Manager
          ↑
          │
     ManagerId
          │
          ↓
       Employee
          │
          └── Reports
```

The principal and dependent are both:

```text
Employee
```

but they play different roles in the relationship.

---

# 🛠️ Configure Self-Referencing Relationship

```csharp
modelBuilder.Entity<Employee>()
    .HasOne(e => e.Manager)
    .WithMany(e => e.Reports)
    .HasForeignKey(e => e.ManagerId)
    .IsRequired(false);
```

### Read It

```text
Employee
   ↓
HasOne(Manager)
   ↓
Manager has Many Reports
   ↓
ManagerId = FK
   ↓
Optional Relationship
```

---

# ⏳ Additional One-to-Many Variations

The following are separate configurations and can be studied later:

```text
One-to-Many
│
├── ✅ Required
├── ✅ Optional
├── ✅ Shadow Foreign Key
├── ✅ Unidirectional
├── ✅ Self-Referencing
│
├── ⏳ Without Navigations
├── ⏳ Alternate Key
├── ⏳ Composite Foreign Key
└── ⏳ Required Without Cascade Delete
```

---

# ⭐ Required vs Optional — Quick Comparison

| Feature | Required | Optional |
|---|---|---|
| FK example | `int BlogId` | `int? BlogId` |
| Can FK be NULL? | ❌ | ✅ |
| Can dependent exist without principal? | ❌ | ✅ |
| Navigation with NRT | `Blog` | `Blog?` |
| Typical delete behavior | Cascade by convention | ClientSetNull by convention |

---

# ⭐ Navigation Shapes

| Navigation Shape | Example | Name |
|---|---|---|
| Both sides | `Blog.Posts` + `Post.Blog` | Bidirectional |
| Principal only | `Blog.Posts` | Unidirectional |
| Dependent only | `Post.Blog` | Unidirectional |
| Neither | No navigations | No-navigation relationship |

---

# 🧠 Complete Mental Model

```text
                 ONE-TO-MANY
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
      PRINCIPAL               DEPENDENT
       / PARENT                / CHILD
          │                       │
       PK / AK                    FK
          │                       │
          └──────────┬────────────┘
                     │
                     ▼
                RELATIONSHIP
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
      Required    Optional   Self-Reference
          │          │          │
          ▼          ▼          ▼
       FK NOT     FK NULL     Same Entity
       NULLABLE   allowed        Type
```

---

# ⚡ 30-Second Revision

```text
ONE-TO-MANY

One Blog
   ↓
Many Posts

Blog
 ├── Id       → Principal Key
 └── Posts    → Collection Navigation

Post
 ├── BlogId   → Foreign Key
 └── Blog     → Reference Navigation


REQUIRED
→ FK is non-nullable
→ Every dependent must have a principal


OPTIONAL
→ FK is nullable
→ Dependent can exist without principal


SHADOW FK
→ FK exists in EF model
→ No CLR FK property


UNIDIRECTIONAL
→ Only one navigation exists


SELF-REFERENCING
→ Same entity type plays both principal
  and dependent roles


FLUENT API
→ HasMany()
→ WithOne()
→ HasForeignKey()
→ IsRequired()


IMPORTANT
→ Configure a relationship only once.
→ Principal does NOT have to contain a dependent.
→ Required means dependent requires principal,
  not that principal requires a dependent.
```

> 🎯 **Core takeaway:** A one-to-many relationship connects **one principal to many dependents** using a principal key, dependent foreign key, and optional navigation properties. The most important variations are **required, optional, shadow-FK, unidirectional, and self-referencing relationships**. :contentReference[oaicite:0]{index=0}