````markdown
# 🧰 EF Core Interview Toolbox — Topic 14: Shadow and Indexer Properties

> **Category:** 🏗️ Creating Model  
> **File Name:** `14-shadow-indexer-properties.md`

---

# 🌫️ Shadow Properties

## 🔎 What Are Shadow Properties?

- A **shadow property** exists in the EF Core model but is **not defined in the .NET entity class**.
- Its value and state are maintained by EF Core's **Change Tracker**.
- They are useful when database data should exist without exposing a corresponding CLR property.

```text
C# Entity
   │
   ├── BlogId
   ├── Url
   └── ❌ LastUpdated
             │
             ▼
        EF Core Model
             │
             └── LastUpdated
                    ↓
              Shadow Property
```

---

# 🔗 Shadow Properties as Foreign Keys

A common use is a foreign key that is not explicitly defined in the entity.

```csharp
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }

    public List<Post> Posts { get; set; }
}

public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }

    public Blog Blog { get; set; }
}
```

There is no:

```csharp
public int BlogId { get; set; }
```

in `Post`.

EF Core can create:

```text
Post Entity
   │
   ├── Blog navigation
   │
   └── BlogId ← Shadow Property
                  │
                  ▼
              FK → Blog.BlogId
```

### 🧠 Key Idea

```text
Navigation Property
→ Used for object relationship in C#

Shadow FK
→ Used by EF Core for database relationship
```

---

# 🏷️ Shadow Property Naming Convention

EF Core derives shadow FK names from navigation/principal key names.

Typical patterns:

```text
Navigation + Principal Key
        ↓
Blog + BlogId
        ↓
BlogId
```

If there is no navigation property:

```text
Principal Type + Principal Key
        ↓
Blog + BlogId
        ↓
BlogBlogId
```

> ⭐ The exact shadow-property name depends on the relationship and key naming conventions.

---

# 🛠️ Configuring Shadow Properties

Use the string overload of `Property<T>()`.

```csharp
modelBuilder.Entity<Blog>()
    .Property<DateTime>("LastUpdated");
```

EF Core now has:

```text
Blog CLR Class
     ↓
No LastUpdated property
     ↓
EF Model
     ↓
"LastUpdated" Shadow Property
```

### ⭐ Important Rule

```csharp
.Property<DateTime>("LastUpdated")
```

behaves according to what already exists:

```text
Property name exists?
        │
   ┌────┼──────────────┐
   │    │              │
 CLR   Shadow         None
   │    │              │
   ▼    ▼              ▼
Config Config      Create Shadow
existing existing   property
property property
```

---

# 🔍 Accessing Shadow Properties

## 1️⃣ Through Change Tracker

Shadow properties can be accessed through `Entry()`.

```csharp
context.Entry(myBlog)
    .Property("LastUpdated")
    .CurrentValue = DateTime.Now;
```

Even though:

```text
Blog class
   ↓
No LastUpdated property
```

EF Core can still maintain:

```text
Change Tracker
   ↓
LastUpdated value
```

---

## 2️⃣ In LINQ Queries

Use:

```csharp
EF.Property<T>()
```

Example:

```csharp
var blogs = context.Blogs
    .OrderBy(b =>
        EF.Property<DateTime>(
            b,
            "LastUpdated"))
    .ToList();
```

`EF.Property<T>()` can be used for:

- Filtering
- Sorting
- Projection

```text
LINQ
 ↓
EF.Property(...)
 ↓
EF Core translates property
 ↓
Database Query
```

---

## ⚠️ No-Tracking Limitation

Shadow-property values depend on EF Core's tracking infrastructure.

```csharp
context.Blogs.AsNoTracking()
```

After a no-tracking query:

```text
No Change Tracker
       ↓
Shadow values are not available
through the tracked entity
```

> ⭐ **Shadow property values are maintained by the Change Tracker, so they cannot be accessed from an entity after a no-tracking query in the normal way.**

---

# 🧩 Indexer Properties

## 🔎 What Are Indexer Properties?

An **indexer property** is accessed using an indexer such as:

```csharp
entity["PropertyName"]
```

instead of a normal CLR property.

EF Core can map these indexer-backed properties to database columns.

```text
CLR Entity
   │
   └── this[string key]
             ↓
       Dictionary / Storage
             ↓
        EF Core Property
             ↓
       Database Column
```

### Why Use Them?

- Add properties without changing the CLR class.
- Support flexible/dynamic schemas.
- Store additional values through an indexer.

---

# 🛠️ Defining an Indexer Property

Example:

```csharp
public class Blog
{
    private readonly Dictionary<string, object> _data
        = new();

    public int BlogId { get; set; }

    public object this[string key]
    {
        get => _data[key];
        set => _data[key] = value;
    }
}
```

Configure the indexer property:

```csharp
modelBuilder.Entity<Blog>()
    .IndexerProperty<DateTime>("LastUpdated");
```

```text
"LastUpdated"
      ↓
IndexerProperty<DateTime>()
      ↓
EF Core Model
      ↓
Database Column
```

---

# 🔄 Using an Indexer Property

Set the value through the CLR indexer:

```csharp
var blog = new Blog();

blog["LastUpdated"] = DateTime.Now;
```

And query it using `EF.Property<T>()`:

```csharp
var blogs = context.Blogs
    .OrderBy(b =>
        EF.Property<DateTime>(
            b,
            "LastUpdated"))
    .ToList();
```

---

# 🆚 Shadow vs Indexer Properties

| Feature | Shadow Property | Indexer Property |
|---|---|---|
| CLR property declared? | ❌ No | ❌ Normal property not required |
| CLR indexer required? | ❌ No | ✅ Yes |
| Stored by | EF Core Change Tracker | CLR indexer/backing storage |
| Accessible in entity code? | Not directly | ✅ `entity["Name"]` |
| Flexible additional properties | ✅ | ✅ |
| Typical use | Hidden model/database data | Dynamic property storage |

### 🧠 Simple Mental Model

```text
Shadow Property

Entity
  ↓
EF Core Model
  ↓
Change Tracker
  ↓
Property Value
```

```text
Indexer Property

Entity
  ↓
Indexer
  ↓
Dictionary / Backing Storage
  ↓
Property Value
```

---

# 🗃️ Property Bag Entity Types

## 🔎 What Are Property Bags?

A **property bag entity type** contains only indexer properties.

The common CLR type used by EF Core is:

```text
Dictionary<string, object>
```

```text
Property Bag
     ↓
Dictionary<string, object>
     ↓
Dynamic Entity Shape
```

This is useful when the complete entity shape is not known at compile time.

---

# 🛠️ Property Bag Example

```csharp
public class MyContext : DbContext
{
    public DbSet<Dictionary<string, object>> Blogs
        => Set<Dictionary<string, object>>("Blog");

    protected override void OnModelCreating(
        ModelBuilder modelBuilder)
    {
        modelBuilder.SharedTypeEntity<
            Dictionary<string, object>>(
                "Blog",
                bb =>
                {
                    bb.Property<int>("BlogId");
                    bb.Property<string>("Url");
                    bb.Property<DateTime>("LastUpdated");
                });
    }
}
```

Create and populate one:

```csharp
var blog = new Dictionary<string, object>
{
    ["BlogId"] = 1,
    ["Url"] = "https://example.com",
    ["LastUpdated"] = DateTime.Now
};

context.Add(blog);
```

### 🧠 Mental Model

```text
Dictionary<string, object>
          ↓
      "Blog" Entity
          ↓
 ┌────────┼────────────┐
 ↓        ↓            ↓
BlogId    Url      LastUpdated
```

---

# 🎯 Why Use Property Bags?

Useful for:

- Dynamic schemas.
- Metadata-driven systems.
- Extensible applications.
- Data whose structure is not known at compile time.
- Multiple logical entities sharing the same CLR type.

---

# ⚠️ Property Bag Limitations

Property bags have several important restrictions:

```text
Property Bag
    │
    ├── ❌ No Shadow Properties
    ├── ❌ No Navigation Properties via Indexers
    ├── ❌ No Inheritance
    ├── ❌ Some relationship APIs unavailable
    └── ❌ Only Dictionary<string, object> is supported
```

### ⭐ Important

> Property bag entities already consist entirely of **indexer properties**, so they do not use shadow properties.

---

# 🧠 Big Picture

```text
                    EF CORE EXTRA PROPERTIES
                             │
                 ┌───────────┴───────────┐
                 │                       │
                 ▼                       ▼
          SHADOW PROPERTY         INDEXER PROPERTY
                 │                       │
          Not in CLR class          CLR indexer
                 │                       │
          Change Tracker            Entity indexer
                 │                       │
                 ▼                       ▼
          Hidden EF data            Dynamic data
```

---

# ⭐ Key Rules to Remember

```text
1. Shadow properties exist in the EF Core model
   but not in the CLR entity class.

2. Shadow-property values are maintained by the
   EF Core Change Tracker.

3. Shadow properties are commonly used as
   implicit foreign keys.

4. A missing DbSet is not related to whether a
   property is shadow; shadow properties are
   model properties without CLR properties.

5. Use Property<T>("Name") to configure a shadow property.

6. Use context.Entry(entity).Property("Name")
   to access a shadow property's value.

7. Use EF.Property<T>(entity, "Name") in LINQ
   to query shadow/indexer properties.

8. No-tracking queries do not provide the normal
   Change Tracker state required for shadow-property values.

9. Indexer properties are accessed through
   an entity indexer such as entity["Name"].

10. Use IndexerProperty<T>("Name") to configure
    an indexer-backed property.

11. Property bag entities use Dictionary<string, object>.

12. Property bags are useful for dynamic schemas.

13. Property bags do not support shadow properties,
    inheritance, or navigation properties via indexers.

14. Shadow property:
    EF stores the value.

15. Indexer property:
    The CLR indexer provides access to the value.
```

---

# ⚡ 30-Second Revision

```text
SHADOW PROPERTY
────────────────

Not in C# class
       ↓
Exists in EF Model
       ↓
Stored by Change Tracker
       ↓
Useful for hidden data / FK


INDEXER PROPERTY
────────────────

Accessed through:

entity["PropertyName"]

       ↓
EF maps indexer-backed property
       ↓
Useful for flexible/dynamic properties


PROPERTY BAG
────────────

Dictionary<string, object>
       ↓
All properties are indexer-based
       ↓
Dynamic entity shape


QUICK DIFFERENCE
────────────────

Shadow
→ Hidden from CLR class
→ EF Change Tracker stores value

Indexer
→ Accessible through entity["Name"]
→ CLR indexer provides storage/access

Property Bag
→ Dictionary<string, object>
→ Dynamic entity with indexer properties
```

> 🎯 **Core takeaway:** **Shadow properties** exist only in the EF Core model and are maintained by the Change Tracker, while **indexer properties** are exposed through a CLR indexer. **Property bag entities** take the idea further by using `Dictionary<string, object>` for fully dynamic entity shapes.
````
