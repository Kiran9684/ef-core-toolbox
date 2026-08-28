
---

### `01-introduction.md`

```markdown
# 🧰 EF Core Interview Toolbox — Topic 1: Introduction

EF CORE TOOLBOX
│
├── WHAT IS EF CORE?
│
│   • Lightweight, extensible, open-source, and cross-platform.
│   • Serves as an Object-Relational Mapper (O/RM).
│   • Enables working with a database using .NET objects.
│   • Eliminates the need for most manual data-access code.
│   • Supports many database engines (SQL Server, SQLite, PostgreSQL, etc.).
│
│
├── THE MODEL
│
│   • Data access in EF Core is performed using a model.
│   • A model is made up of two main components:
│       1. Entity Classes (e.g., Blog, Post)
│       2. Context Object (DbContext)
│
│   Context Object (DbContext)
│   • Represents a session with the database.
│   • Allows querying and saving data.
│
│   ⭐ Code Example:
│       public class BloggingContext : DbContext
│       {
│           public DbSet<Blog> Blogs { get; set; }
│           public DbSet<Post> Posts { get; set; }
│       }
│
│
├── DEVELOPMENT APPROACHES
│
│   • Generate a model from an existing database (Database-First).
│   • Hand-code a model to match the database (Code-First).
│
│   ⭐ EF Migrations:
│   • Once a model is created, use Migrations to create the database.
│   • Migrations allow evolving the database seamlessly as the C# model changes.
│
│
├── QUERYING DATA
│
│   • Instances of your entity classes are retrieved from the DB using LINQ.
│   • (Language Integrated Query).
│
│   ⭐ Code Example:
│       using (var db = new BloggingContext())
│       {
│           var blogs = await db.Blogs
│               .Where(b => b.Rating > 3)
│               .OrderBy(b => b.Url)
│               .ToListAsync();
│       }
│
│
├── SAVING DATA
│
│   • Data is created, deleted, and modified in the DB using instances
│     of your entity classes.
│
│   ⭐ Code Example:
│       using (var db = new BloggingContext())
│       {
│           var blog = new Blog { Url = "http://sample.com" };
│           db.Blogs.Add(blog);
│           await db.SaveChangesAsync();
│       }
│
│   ⭐ IMPORTANT LIFECYCLE:
│   • .Add() → Tells EF Core to track the new entity in memory.
│   • .SaveChangesAsync() → Translates tracked changes into SQL and pushes to DB.
│
│
└── 🧠 30-SECOND MENTAL MODEL

    DbContext
    → Your active session with the DB.

    DbSet<T>
    → Represents a specific Table in the DB.

    LINQ
    → How you ask for data (Generates SQL SELECT).

    SaveChanges()
    → How you push changes (Generates SQL INSERT/UPDATE/DELETE).