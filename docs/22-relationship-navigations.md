# 🧰 EF Core Interview Toolbox — Topic 22: Relationship Navigations

> **Category:** 🏗️ Creating Model  
> **File Name:** `22-relationship-navigations.md`

---

# 🧭 Relationship Navigations

## 🔎 Core Concept

- EF Core relationships are fundamentally defined using **foreign keys**.
- **Navigations** are layered over those foreign keys to provide a natural, object-oriented way to work with relationships.
- This allows application code to work with **entity graphs** without directly managing FK values. :contentReference[oaicite:0]{index=0}

```text
DATABASE RELATIONSHIP

Principal Key
     ↑
     │
Foreign Key
     │
Dependent


OBJECT MODEL

Principal
   ↕
Navigation
   ↕
Dependent
```

---

# 🌳 Entity Graph

A **graph** is a set of objects connected through references/navigation properties.

Example:

```csharp
var post = new Post
{
    Title = "EF Core",
    Tags = new List<Tag>
    {
        new Tag { Name = "ORM" },
        new Tag { Name = "DotNet" }
    }
};
```

```text
Post
 ├── Tag: ORM
 └── Tag: DotNet
```

> 🧠 Navigations let you work with the object graph naturally instead of manually managing every FK value.

---

# 🚫 Navigations Cannot Be Shared Between Relationships

A single FK can be associated with:

- At most one navigation from **dependent → principal**.
- At most one navigation from **principal → dependent**.

### ✅ Valid

```text
Order
 ├── CreatedById ──→ User
 │       ↓
 │   CreatedBy
 │
 └── ApprovedById ─→ User
         ↓
      ApprovedBy
```

```csharp
public class Order
{
    public int Id { get; set; }

    public int CreatedById { get; set; }
    public int ApprovedById { get; set; }

    public User CreatedBy { get; set; } = null!;
    public User ApprovedBy { get; set; } = null!;
}

public class User
{
    public int Id { get; set; }

    public List<Order> CreatedOrders { get; set; } = [];
    public List<Order> ApprovedOrders { get; set; } = [];
}
```

Configure each relationship separately:

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>()
        .HasOne(o => o.CreatedBy)
        .WithMany(u => u.CreatedOrders)
        .HasForeignKey(o => o.CreatedById)
        .OnDelete(DeleteBehavior.Restrict);

    modelBuilder.Entity<Order>()
        .HasOne(o => o.ApprovedBy)
        .WithMany(u => u.ApprovedOrders)
        .HasForeignKey(o => o.ApprovedById)
        .OnDelete(DeleteBehavior.Restrict);
}
```

### ❌ Invalid Pattern

```csharp
public class Order
{
    public int CreatedById { get; set; }
    public int ApprovedById { get; set; }

    public User User { get; set; }
}
```

```text
CreatedById ──┐
              ├──→ User
ApprovedById ─┘

       ❌ One navigation cannot represent
          both relationships
```

> ⭐ When multiple FKs connect the same two entity types, each relationship needs its **own navigation pair** when navigations are used.

---

# 💤 `virtual` Navigations

Navigations do **not** need to be `virtual` by default.

```csharp
public Blog Blog { get; set; } = null!;
```

`virtual` is needed when using EF Core mechanisms such as:

- Lazy-loading proxies.
- Change-tracking proxies.

```text
Normal Navigation
→ virtual not required

Proxy-based Lazy Loading / Change Tracking
→ virtual may be required
```

> 💡 Don't add `virtual` automatically unless a proxy-based feature requires it. :contentReference[oaicite:1]{index=1}

---

# 🔹 Reference Navigations

A **reference navigation** is a normal object reference to another entity.

```csharp
public Blog Blog { get; set; }
```

Reference navigations represent the **one side** of:

```text
One-to-Many
One-to-One
```

### 🧠 Mental Model

```text
Post
  │
  └── Blog
        ↓
   Reference Navigation
```

---

# 🔐 Reference Navigation Rules

- A reference navigation must have a **setter**.
- The setter does not need to be public.
- Avoid automatically initializing a reference navigation to a non-null object when no related entity is guaranteed to exist.

### ❌ Avoid

```csharp
public Blog Blog { get; set; } = new Blog();
```

This effectively says:

```text
Blog navigation
      ↓
"There is always a Blog"
```

when that may not be true.

---

# ❓ Required Reference Navigations

For the **dependent → principal** navigation:

```text
Required FK
   ↓
Non-nullable FK
   ↓
Required relationship
   ↓
Navigation should represent a required principal
```

For an optional relationship:

```csharp
public int? BlogId { get; set; }

public Blog? Blog { get; set; }
```

```text
Nullable FK
    ↓
Optional Relationship
    ↓
Nullable Reference Navigation
```

> ⭐ With C# nullable reference types enabled, a dependent-to-principal navigation should be nullable when its FK is nullable. :contentReference[oaicite:2]{index=2} :contentReference[oaicite:3]{index=3}

---

# 🔸 Collection Navigations

A **collection navigation** represents multiple related entities.

```csharp
public ICollection<Post> Posts { get; } = new List<Post>();
```

Collection navigations represent the **many side** of:

```text
One-to-Many
Many-to-Many
```

```text
Blog
 │
 ├── Post 1
 ├── Post 2
 └── Post 3
```

### Important

- A collection navigation does **not** need a setter.
- It is common to initialize it inline.

---

# ⚠️ Expression-Bodied Collection Trap

### ❌ Incorrect

```csharp
public ICollection<Post> Posts
    => new List<Post>();
```

Every access creates a **new collection**.

```text
Access Posts
   ↓
New List

Access Posts again
   ↓
Another New List

❌ Navigation becomes useless
```

### ✅ Correct

```csharp
public ICollection<Post> Posts { get; }
    = new List<Post>();
```

```text
One Entity Instance
      ↓
One Collection Instance
      ↓
EF + Application share the same navigation
```

---

# 📦 Collection Types

The underlying collection must:

- Implement `ICollection<T>`.
- Have a working `Add()` method.

Common choices:

| Collection | Typical Characteristic |
|---|---|
| `List<T>` | Efficient for smaller collections; stable ordering |
| `HashSet<T>` | Faster lookups for larger collections; no stable ordering |
| Custom Collection | Supported when it satisfies EF Core requirements |
| Array | ❌ Not suitable because `Add()` throws |

> ⭐ Even though the underlying collection must implement `ICollection<T>`, the property does not have to be exposed as `ICollection<T>`.

---

# 👀 Exposing Collections as `IEnumerable<T>`

A collection can be exposed as a read-only view:

```csharp
public class Blog
{
    public int Id { get; set; }

    public IEnumerable<Post> Posts { get; }
        = new List<Post>();
}
```

```text
Internal Collection
       ↓
IEnumerable<Post>
       ↓
Application sees read-only view
```

This reduces direct modification by application code.

---

# 🛡️ Encapsulated Collection Pattern

A common domain-model approach:

```csharp
public class Blog
{
    private readonly List<Post> _posts = new();

    public int Id { get; set; }

    public IEnumerable<Post> Posts => _posts;

    public void AddPost(Post post)
        => _posts.Add(post);
}
```

```text
Application
     ↓
AddPost()
     ↓
_private List
```

This keeps collection manipulation behind a domain method.

---

# 🛡️ Defensive Copy

`IEnumerable<T>` alone does not completely prevent someone from casting the underlying collection.

A defensive copy can be returned:

```csharp
public class Blog
{
    private readonly List<Post> _posts = new();

    public int Id { get; set; }

    public IEnumerable<Post> Posts
        => _posts.ToList();

    public void AddPost(Post post)
        => _posts.Add(post);
}
```

```text
Internal Collection
       ↓
ToList()
       ↓
Separate Copy
       ↓
Application
```

> 💡 Use a defensive copy when preventing external mutation is important.

---

# 🏗️ Initializing Collection Navigations

Collection navigations can be initialized eagerly or lazily.

## Eager Initialization

```csharp
public ICollection<Post> Posts { get; }
    = new List<Post>();
```

The collection exists as soon as the entity is created.

## Lazy Initialization

```csharp
private ICollection<Post>? _posts;

public ICollection<Post> Posts
    => _posts ??= new List<Post>();
```

```text
First Access
    ↓
_posts == null
    ↓
Create Collection
    ↓
Store Collection

Later Access
    ↓
Reuse Same Collection
```

---

# 🤖 EF Core Can Initialize Collections

If EF Core needs to add related entities to a collection and it is currently `null`, EF Core can create an appropriate collection instance.

The type depends on how the navigation is exposed.

```text
Navigation Type
      ↓
EF chooses collection implementation
```

Typical behavior:

```text
HashSet<T>
   ↓
HashSet<T> + ReferenceEqualityComparer

Concrete collection with parameterless constructor
   ↓
That concrete type

IEnumerable<T> / ICollection<T> / ISet<T>
   ↓
HashSet<T> + ReferenceEqualityComparer

IList<T>
   ↓
List<T>

Unsupported type
   ↓
Exception
```

> ⭐ EF Core uses **reference equality** for the automatically created `HashSet<T>` collections. :contentReference[oaicite:4]{index=4}

---

# ⚙️ Configuring Navigations

Navigations normally become part of the model when the relationship is configured:

```text
Convention
   OR
Fluent API
   ↓
Relationship
   ↓
Navigation
```

Most navigation configuration is actually **relationship configuration**:

```csharp
HasOne()
HasMany()
WithOne()
WithMany()
```

Some settings are specific to the navigation itself and use:

```csharp
Navigation(...)
```

---

# 🛠️ `Navigation()` Configuration

Example:

```csharp
modelBuilder.Entity<Blog>()
    .Navigation(e => e.Posts)
    .UsePropertyAccessMode(
        PropertyAccessMode.Property);

modelBuilder.Entity<Post>()
    .Navigation(e => e.Blog)
    .UsePropertyAccessMode(
        PropertyAccessMode.Property);
```

This forces EF Core to access the navigation through its **property** rather than its backing field.

```text
Default / Configured Access
       ↓
Property or Backing Field

UsePropertyAccessMode(Property)
       ↓
Force Property Access
```

> ⚠️ `Navigation()` does **not create** a navigation. The relationship/navigation must already exist. :contentReference[oaicite:5]{index=5}

---

# 🧠 Required Navigation ≠ Required Dependent

This distinction is extremely important.

For a dependent:

```text
Required Relationship
      ↓
Dependent must have Principal
```

But from the principal side:

```text
Principal
      ↓
Can usually exist
without any dependents
```

Example:

```text
Blog
 │
 └── 0 Posts
```

is normally valid even when every `Post` is required to have a `Blog`.

> ⭐ EF Core does not generally provide a way to enforce "every principal must have at least one dependent" through the relationship model or standard relational database constraints. That rule belongs in **business/application logic**. :contentReference[oaicite:6]{index=6}

### Exception

When principal and dependent share the same table or are contained in a document, a principal-to-dependent navigation can be marked required to indicate that the dependent must exist.

Examples include:

- Owned types.
- Table splitting.

:contentReference[oaicite:7]{index=7}

---

# 🔄 Navigation + Foreign Key Mental Model

```text
              OBJECT MODEL
                   │
          ┌────────┴────────┐
          │                 │
          ▼                 ▼
 Reference Navigation   Collection Navigation
          │                 │
          └────────┬────────┘
                   ▼
                EF Core
                   │
                   ▼
             Foreign Key
                   │
                   ▼
              Database
```

```text
Reference Navigation
→ "Which related entity?"

Collection Navigation
→ "Which related entities?"

Foreign Key
→ "How is the relationship stored?"
```

---

# ⭐ Key Rules to Remember

```text
1. Navigations provide an object-oriented view over FK-based relationships.

2. A graph is a set of entities connected through navigations.

3. One FK cannot be associated with multiple navigations
   on the same relationship end.

4. When two relationships use the same entity types,
   use distinct navigations for each relationship.

5. Navigations do not need virtual unless proxy-based
   lazy loading or change tracking requires it.

6. Reference navigation = object reference.
   It represents the "one" side.

7. Reference navigations require a setter,
   though the setter can be non-public.

8. Avoid initializing reference navigations to a fake/default
   entity when the relationship is not guaranteed.

9. Collection navigation = collection of related entities.
   It represents the "many" side.

10. Collection navigations do not require a setter.

11. The underlying collection must implement ICollection<T>
    and have a working Add() method.

12. List<T> and HashSet<T> are common collection choices.

13. Arrays should not be used as collection navigations.

14. Expression-bodied collections that create a new collection
    on every access are incorrect navigation implementations.

15. Collections can be exposed as IEnumerable<T> to reduce
    direct modification.

16. A defensive copy can further protect an internal collection.

17. Collection navigations can be eagerly or lazily initialized.

18. EF Core can initialize null collection navigations when
    materializing related entities.

19. Navigation-specific configuration uses Navigation().

20. Navigation() configures an existing navigation;
    it does not create one.

21. Required dependent navigation generally follows
    required FK / relationship semantics.

22. A required relationship does NOT mean the principal
    must have a dependent.

23. Business rules such as "every Blog must have a Post"
    should normally be enforced in application logic.

24. With nullable reference types, optional dependent-to-principal
    navigations should be nullable.
```

---

# ⚡ 30-Second Revision

```text
RELATIONSHIP
     ↓
Foreign Key
     ↓
Navigation
     ↓
Object-Oriented Access


NAVIGATION TYPES

Reference
→ One related entity
→ One-to-One / One-to-Many

Collection
→ Many related entities
→ One-to-Many / Many-to-Many


REFERENCE

Post
  └── Blog


COLLECTION

Blog
  ├── Post
  ├── Post
  └── Post


IMPORTANT

Multiple FKs
    ↓
Multiple Relationships
    ↓
Distinct Navigations


COLLECTION

ICollection<T>
   ↓
List<T> / HashSet<T>

❌ Array
❌ new collection every property access


REQUIRED

Required FK
    ↓
Dependent requires Principal

BUT

Principal
    ↓
Can usually exist without Dependent


CONFIGURATION

HasOne / HasMany / WithOne / WithMany
    ↓
Relationship

Navigation()
    ↓
Navigation-specific settings
```

> 🎯 **Core takeaway:** Navigations are the **object-oriented layer over EF Core foreign-key relationships**. Remember the two types — **reference** and **collection** — and understand their rules around setters, nullability, initialization, collection implementations, multiple relationships, and relationship-specific configuration.