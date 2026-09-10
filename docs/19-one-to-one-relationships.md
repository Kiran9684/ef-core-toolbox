# 🧰 EF Core Interview Toolbox — Topic 18: One-to-One Relationships

> **Category:** 🏗️ Creating Model  
> **File Name:** `18-one-to-one-relationships.md`

---

# 🔗 One-to-One Relationships

## 🔎 Core Concept

A **one-to-one relationship** is used when one entity is associated with **at most one** other entity.

Example:

```text
Blog
  │
  └── BlogHeader

BlogHeader
  └── Blog
```

- One `Blog` can have at most one `BlogHeader`.
- One `BlogHeader` belongs to one `Blog`.
- The relationship uses a **principal key** and **foreign key**.

---

# 🧠 Relationship Anatomy

```text
PRINCIPAL / PARENT
        │
        │ Primary or Alternate Key
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
BlogHeader.BlogId
```

Optional navigations:

```text
Blog
 └── Header       ← Reference Navigation

BlogHeader
 └── Blog         ← Reference Navigation
```

> ⭐ Unlike one-to-many relationships, both sides of a one-to-one relationship use **reference navigations**.

---

# 🎯 Choosing Principal vs Dependent

Sometimes it is not immediately obvious which entity is the principal.

Useful rules:

```text
Existing database tables?
        ↓
Table containing FK
        ↓
Dependent
```

```text
Logical parent/child?
        ↓
Child usually → Dependent
```

```text
Can one entity logically exist
without the other?
        ↓
Entity that cannot exist alone
        ↓
Usually Dependent
```

Example:

```text
Blog
  ↓
BlogHeader

A BlogHeader without a Blog
does not make logical sense.

Therefore:

Blog       → Principal
BlogHeader → Dependent
```

---

# ✅ Required One-to-One

```csharp
public class Blog
{
    public int Id { get; set; }

    public BlogHeader Header { get; set; } = null!;
}

public class BlogHeader
{
    public int Id { get; set; }

    public int BlogId { get; set; }

    public Blog Blog { get; set; } = null!;
}
```

### Why Required?

```text
BlogHeader.BlogId
       ↓
      int
       ↓
Cannot be NULL
       ↓
BlogHeader MUST have a Blog
```

---

# 🧠 Required Does NOT Mean Principal Must Have a Dependent

This is an important distinction.

```text
Required One-to-One

Blog
 │
 └── BlogHeader
```

means:

```text
Every BlogHeader
     ↓
Must have a Blog
```

It does **not** guarantee:

```text
Every Blog
     ↓
Must have a BlogHeader
```

A principal can exist without a dependent.

> ⭐ If the business rule requires every `Blog` to have a `BlogHeader`, that rule must be enforced through application/business logic.

---

# ↔️ Bidirectional One-to-One

If both entities contain navigations:

```text
Blog
 │
 └── Header
       ↕
   BlogHeader
       │
       └── Blog
```

this is a **bidirectional relationship**.

EF Core can discover this simple relationship by convention.

It identifies:

```text
Blog
→ Principal

BlogHeader
→ Dependent

BlogHeader.BlogId
→ Foreign Key

Blog.Header
→ Navigation

BlogHeader.Blog
→ Navigation
```

---

# 🛠️ Fluent API — Required One-to-One

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasOne(e => e.Header)
        .WithOne(e => e.Blog)
        .HasForeignKey<BlogHeader>(e => e.BlogId)
        .IsRequired();
}
```

### 🧠 Read It Left to Right

```text
Blog
 ↓
HasOne(Header)
 ↓
BlogHeader
 ↓
WithOne(Blog)
 ↓
BlogHeader.BlogId
 ↓
Foreign Key
 ↓
IsRequired()
```

### Method Meaning

| Method | Meaning |
|---|---|
| `HasOne(e => e.Header)` | Blog has one BlogHeader |
| `WithOne(e => e.Blog)` | BlogHeader has one Blog |
| `HasForeignKey<BlogHeader>(...)` | BlogHeader contains the FK |
| `IsRequired()` | Relationship is required |

> ⭐ The relationship only needs to be configured **once**.

---

# 🟡 Optional One-to-One

An optional relationship allows the dependent to exist without a principal.

```csharp
public class Blog
{
    public int Id { get; set; }

    public BlogHeader? Header { get; set; }
}

public class BlogHeader
{
    public int Id { get; set; }

    public int? BlogId { get; set; }

    public Blog? Blog { get; set; }
}
```

### Mental Model

```text
BlogHeader.BlogId
       ↓
      int?
       ↓
   Can be NULL
       ↓
BlogHeader can exist
without a Blog
```

---

# 🧩 Nullable Reference Types

When C# nullable reference types are enabled:

```text
Nullable FK
     ↓
BlogId = int?
     ↓
Dependent navigation
     ↓
Blog? 
```

For a non-nullable FK:

```text
BlogId = int
     ↓
Required relationship
```

The navigation can be nullable or non-nullable from the CLR perspective, depending on how the application uses it.

Example:

```csharp
public int BlogId { get; set; }

public Blog Blog { get; set; } = null!;
```

`= null!` tells the compiler that the property is intentionally initialized later, typically by EF Core.

---

# 🛠️ Fluent API — Optional One-to-One

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasOne(e => e.Header)
        .WithOne(e => e.Blog)
        .HasForeignKey<BlogHeader>(e => e.BlogId)
        .IsRequired(false);
}
```

```text
IsRequired(false)
        ↓
Optional Relationship
        ↓
BlogHeader.BlogId → NULL allowed
```

---

# 🌫️ One-to-One with Shadow Foreign Key

A one-to-one relationship can also use a foreign key that does not exist as a CLR property.

```text
BlogHeader
   │
   ├── Id
   ├── Blog navigation
   └── ❌ No BlogId CLR property
```

EF Core can maintain:

```text
BlogId
  ↓
Shadow FK
```

This variation will be studied separately.

---

# 🧭 Navigation Shapes

```text
ONE-TO-ONE
│
├── Both navigations
│     → Bidirectional
│
├── Only principal navigation
│     → Unidirectional
│
├── Only dependent navigation
│     → Unidirectional
│
└── No navigations
      → Relationship without navigations
```

These variations require different configuration patterns.

---

# 🔄 Relationship Mapping Mental Model

```text
             C# OBJECT MODEL
                    │
          ┌─────────┴─────────┐
          │                   │
      Blog.Header        Header.Blog
          │                   │
          └─────────┬─────────┘
                    ↓
               EF CORE
                    ↓
             Relationship
                    ↓
              PK ↔ FK
                    ↓
           RELATIONAL DATABASE
```

---

# 🧠 Required vs Optional

```text
REQUIRED

BlogHeader.BlogId
       ↓
   Non-nullable
       ↓
Every dependent needs principal


OPTIONAL

BlogHeader.BlogId
       ↓
      Nullable
       ↓
Dependent can exist without principal
```

> ⭐ In both cases, the principal itself can exist without a dependent.

---

# 👤 Self-Referencing One-to-One

A self-referencing relationship occurs when the **principal and dependent are the same entity type**.

Example:

```text
Person
 │
 ├── Husband
 │
 └── Wife
```

Example class:

```csharp
public class Person
{
    public int Id { get; set; }

    public int? HusbandId { get; set; }

    public Person? Husband { get; set; }

    public Person? Wife { get; set; }
}
```

Here:

```text
Person
  │
  ├── HusbandId → FK
  ├── Husband   → Dependent → Principal navigation
  └── Wife      → Principal → Dependent navigation
```

The relationship is optional because:

```csharp
public int? HusbandId { get; set; }
```

allows a person not to have a husband.

---

# 🛠️ Configure Self-Referencing One-to-One

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Person>()
        .HasOne(e => e.Husband)
        .WithOne(e => e.Wife)
        .HasForeignKey<Person>(e => e.HusbandId)
        .IsRequired(false);
}
```

### 🧠 Important Detail

With a self-referencing one-to-one relationship:

```text
Principal Type = Person
Dependent Type = Person
```

Therefore, simply specifying the CLR type does **not** identify which role the type is playing.

EF Core determines the roles from the navigation directions:

```text
HasOne(...)
   ↓
Dependent → Principal

WithOne(...)
   ↓
Principal → Dependent
```

> ⭐ In self-referencing relationships, the **navigation configuration helps establish the dependent and principal ends**.

---

# ⏳ Additional One-to-One Variations

These are separate configurations and can be studied later:

```text
One-to-One
│
├── ✅ Required
├── ✅ Optional
├── ✅ Self-Referencing
│
├── ⏳ Primary Key → Primary Key
├── ⏳ Shadow Foreign Key
├── ⏳ Optional Shadow Foreign Key
├── ⏳ Without navigation to principal
├── ⏳ Without navigation to principal + shadow FK
├── ⏳ Without navigation to dependent
├── ⏳ No navigations
├── ⏳ Alternate Key
├── ⏳ Composite Foreign Key
└── ⏳ Required without cascade delete
```

---

# ⭐ Key Rules to Remember

```text
1. One-to-one means one entity is associated
   with at most one other entity.

2. One-to-one uses a principal key and foreign key.

3. The dependent usually contains the FK.

4. Existing database:
   table containing FK → usually dependent.

5. Logical parent/child:
   child → usually dependent.

6. Required FK → dependent must have a principal.

7. Required does NOT mean the principal must have a dependent.

8. Optional FK → dependent can exist without a principal.

9. Both sides can have reference navigations.

10. Two navigations still represent one relationship.

11. Configure the relationship only once.

12. HasOne() + WithOne() is the common Fluent API pattern.

13. HasForeignKey<T>() identifies which entity contains the FK.

14. Self-referencing one-to-one uses the same CLR type
    for principal and dependent.

15. With self-referencing relationships, navigation direction
    helps define which end is principal and which is dependent.

16. Several one-to-one variations such as shadow FK,
    alternate-key, composite-FK, and no-navigation cases
    require separate configuration.
```

---

# ⚡ 30-Second Revision

```text
                 ONE-TO-ONE

              PRINCIPAL
                 Blog
                   │
              PK: Id
                   │
                   ▼
              DEPENDENT
            BlogHeader
                   │
             FK: BlogId
                   │
                   ▼
                Blog


REQUIRED
→ BlogId is non-nullable
→ Dependent must have Principal


OPTIONAL
→ BlogId is nullable
→ Dependent may exist without Principal


NAVIGATIONS
→ Blog.Header
→ BlogHeader.Blog


FLUENT API
→ HasOne()
→ WithOne()
→ HasForeignKey<T>()
→ IsRequired()


SELF-REFERENCING
→ Same entity type on both ends
→ Navigation direction identifies roles
```

> 🎯 **Core takeaway:** A one-to-one relationship connects a **principal and dependent with at most one related entity**. The key distinction is that the **dependent owns the foreign key**, while required/optional behavior is determined primarily by FK nullability. Self-referencing one-to-one relationships use the same entity type on both sides and require careful navigation-based configuration.