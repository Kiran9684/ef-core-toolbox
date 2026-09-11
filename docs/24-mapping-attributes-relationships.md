# 🧰 EF Core Interview Toolbox — Topic 24: Mapping Attributes for Relationships

> **Category:** 🏗️ Creating Model  
> **File Name:** `24-mapping-attributes-relationships.md`

---

# 🏷️ Mapping Attributes

## 🔎 Core Concept

- **Mapping attributes** (also called **Data Annotations**) can modify or override EF Core relationship conventions.
- Fluent API configuration in `OnModelCreating()` has **higher precedence** and can override mapping attributes.

```text
EF Core Relationship Configuration

Conventions
     ↓
Mapping Attributes
     ↓
Fluent API
     ↓
✅ Final EF Core Model
```

> ⭐ **Fluent API is the final source of truth.** :contentReference[oaicite:0]{index=0}

---

# 📦 Where Do Mapping Attributes Come From?

Common attributes come from:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
```

EF Core-specific mapping attributes are also available through:

```text
Microsoft.EntityFrameworkCore.Abstractions
```

> 💡 Data Annotations are also used by technologies such as ASP.NET Core MVC and EF6, so semantics can differ between frameworks. EF Core-specific mapping attributes are provided to keep EF Core behavior clear. 

---

# 1️⃣ `[Required]`

`[Required]` indicates that a property cannot contain `null`.

For relationships, it is commonly used on:

- A foreign key property.
- The dependent navigation.

---

## 🔹 On the Foreign Key

```csharp
public class Post
{
    public int Id { get; set; }

    [Required]
    public string BlogId { get; set; }

    public Blog Blog { get; init; }
}
```

Conceptually:

```text
[Required]
    ↓
BlogId cannot be NULL
    ↓
Required Relationship
```

> ⚠️ With C# nullable reference types enabled, a non-nullable FK may already be configured as required, so `[Required]` may add no additional effect.

---

## 🔹 On the Dependent Navigation

```csharp
public class Post
{
    public int Id { get; set; }

    public string BlogId { get; set; }

    [Required]
    public Blog Blog { get; init; }
}
```

```text
[Required] on dependent navigation
            ↓
Foreign Key becomes required
            ↓
Required Relationship
```

---

## 🔹 Shadow Foreign Key

Even if there is no CLR FK property:

```csharp
public class Post
{
    public int Id { get; set; }

    [Required]
    public Blog Blog { get; init; }
}
```

EF Core can make the shadow FK non-nullable:

```text
Post.Blog
   ↓
[Required]
   ↓
Shadow BlogId
   ↓
NON-NULL
   ↓
Required Relationship
```

> ⚠️ `[Required]` on the **principal-side navigation** has no effect. :contentReference[oaicite:1]{index=1}

---

# 2️⃣ `[ForeignKey]`

`[ForeignKey]` connects a **foreign key property** with its navigation.

There are several ways to use it.

---

## 🔹 FK Attribute on the FK Property

```csharp
public class Post
{
    public int Id { get; set; }

    [ForeignKey(nameof(Blog))]
    public string BlogKey { get; set; }

    public Blog Blog { get; init; }
}
```

```text
BlogKey
   ↓
[ForeignKey(nameof(Blog))]
   ↓
Associated with Blog navigation
```

---

## 🔹 FK Attribute on the Navigation

```csharp
public class Post
{
    public int Id { get; set; }

    public string BlogKey { get; set; }

    [ForeignKey(nameof(BlogKey))]
    public Blog Blog { get; init; }
}
```

```text
Blog navigation
      ↓
[ForeignKey(nameof(BlogKey))]
      ↓
Uses BlogKey as FK
```

---

## 🔹 Creating a Shadow FK

If `[ForeignKey]` refers to a property that does not exist:

```csharp
public class Post
{
    public int Id { get; set; }

    [ForeignKey("BlogKey")]
    public Blog Blog { get; init; }
}
```

EF Core creates:

```text
"BlogKey"
   ↓
No CLR property found
   ↓
Shadow Foreign Key created
```

> ⭐ `[ForeignKey]` can therefore explicitly associate navigations with existing FK properties or cause a shadow FK to be created when the named property does not exist.

---

# 3️⃣ `[InverseProperty]`

`[InverseProperty]` tells EF Core which navigation is the **inverse** of another navigation.

This becomes important when **multiple relationships exist between the same two entity types**.

```text
Blog
  │
  ├── Posts
  │
  └── FeaturedPost
       │
       ↓
      Post
```

EF conventions may not know which navigation should pair with which.

---

## ✅ Example

```csharp
public class Blog
{
    public int Id { get; set; }

    [InverseProperty(nameof(Post.Blog))]
    public List<Post> Posts { get; } = new();

    public int FeaturedPostId { get; set; }

    public Post FeaturedPost { get; set; }
}

public class Post
{
    public int Id { get; set; }

    public int BlogId { get; set; }

    public Blog Blog { get; init; }
}
```

```text
Blog.Posts
    ↕
Post.Blog

✅ Explicitly paired
```

```text
Blog.FeaturedPost
    ↕
Another relationship with Post
```

> ⭐ `[InverseProperty]` is mainly used to resolve **navigation-pairing ambiguity** when multiple relationships exist between the same entity types.

---

# 4️⃣ `[DeleteBehavior]`

`[DeleteBehavior]` changes the delete behavior configured for a relationship.

Example:

```csharp
public class Post
{
    public int Id { get; set; }

    public int BlogId { get; set; }

    [DeleteBehavior(DeleteBehavior.Restrict)]
    public Blog Blog { get; init; }
}
```

```text
Delete Blog
    ↓
DeleteBehavior.Restrict
    ↓
Prevent dependent relationship from cascading
```

---

# 🧠 DeleteBehavior Quick Model

Common behaviors:

| DeleteBehavior | Basic Meaning |
|---|---|
| `Cascade` | Delete dependents |
| `ClientSetNull` | Set FK to null on tracked dependents when possible |
| `Restrict` | Prevent principal deletion when dependents exist |
| `NoAction` | Leave enforcement to the database |

Default conventions:

```text
Optional Relationship
        ↓
ClientSetNull

Required Relationship
        ↓
Cascade
```

> ⭐ The exact effects depend on whether dependents are tracked and on the database/provider behavior.

---

# 🆚 Attribute vs Fluent API

The same relationship configuration can often be expressed either way.

### Data Annotation

```csharp
[DeleteBehavior(DeleteBehavior.Restrict)]
public Blog Blog { get; init; }
```

### Fluent API

```csharp
modelBuilder.Entity<Post>()
    .HasOne(p => p.Blog)
    .WithMany(b => b.Posts)
    .OnDelete(DeleteBehavior.Restrict);
```

```text
Data Annotation
       ↓
Convention Override

Fluent API
       ↓
Highest Precedence
```

---

# 🧠 Relationship Attributes — Big Picture

```text
                 MAPPING ATTRIBUTES
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
      [Required]    [ForeignKey]   [InverseProperty]
          │              │              │
          │              │              └── Pair navigations
          │              └── Connect FK + navigation
          └── Required relationship

                    +
                    
              [DeleteBehavior]
                     │
                     ▼
              Delete behavior
```

---

# ⚖️ Attribute Selection Mental Model

```text
Problem
   │
   ├── Relationship must be required?
   │       ↓
   │    [Required]
   │
   ├── Which property is the FK?
   │       ↓
   │    [ForeignKey]
   │
   ├── Which navigation is the inverse?
   │       ↓
   │    [InverseProperty]
   │
   └── What happens on delete?
           ↓
      [DeleteBehavior]
```

---

# ⭐ Key Rules to Remember

```text
1. Mapping attributes override relationship conventions.

2. Fluent API configuration overrides mapping attributes.

3. [Required] on a dependent FK makes the relationship required.

4. [Required] on a dependent navigation also makes the
   relationship required.

5. [Required] can make a shadow FK non-nullable.

6. [Required] on the principal-side navigation has no effect.

7. [ForeignKey] connects an FK property with a navigation.

8. [ForeignKey] can be placed on either the FK property
   or a navigation.

9. If [ForeignKey] names a property that does not exist,
   EF Core creates a shadow FK.

10. [InverseProperty] resolves ambiguity when multiple
    relationships exist between the same entity types.

11. [DeleteBehavior] changes how EF handles deletes
    for a relationship.

12. Optional relationships use ClientSetNull by convention.

13. Required relationships use Cascade by convention.

14. Data Annotations are convenient for simple relationship
    configuration, but Fluent API provides greater control.

15. The Fluent API in OnModelCreating() is the final source
    of truth for the EF Core model.
```

---

# ⚡ 30-Second Revision

```text
RELATIONSHIP DATA ANNOTATIONS

[Required]
     ↓
Required Relationship


[ForeignKey]
     ↓
Connect FK ↔ Navigation


[InverseProperty]
     ↓
Resolve Navigation Pairing


[DeleteBehavior]
     ↓
Control Delete Behavior


PRECEDENCE

Convention
    ↓
Data Annotation
    ↓
Fluent API
    ↓
✅ Final Model
```

> 🎯 **Core takeaway:** Relationship mapping attributes are mainly useful for **requiredness, FK association, inverse-navigation pairing, and delete behavior**. Remember the four key attributes: **`[Required]` → required relationship, `[ForeignKey]` → FK mapping, `[InverseProperty]` → navigation pairing, `[DeleteBehavior]` → delete behavior**.