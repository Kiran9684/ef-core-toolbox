# 🧰 EF Core Interview Toolbox — Topic 25: Managing Database Schemas

> **Category:** 🗂️ Managing Database Schemas  
> **File Name:** `25-managing-database-schemas.md`

---

# 🗄️ Managing Database Schemas

EF Core provides two main approaches to keep the **EF Core model** and **database schema** in sync.

```text
           SOURCE OF TRUTH?
                  │
         ┌────────┴────────┐
         │                 │
         ▼                 ▼
    EF Core Model      Database Schema
         │                 │
         ▼                 ▼
     Migrations       Reverse Engineering
```

| Source of Truth | Approach | Purpose |
|---|---|---|
| EF Core Model | **Migrations** | Evolve database schema from model changes |
| Database Schema | **Reverse Engineering** | Scaffold model from existing database |

> ⭐ **Model is the source of truth → Migrations**  
> ⭐ **Database is the source of truth → Reverse Engineering**

---

# 🔄 Migrations — Core Idea

In real applications, the model changes over time:

```text
Initial Model
     ↓
Add Property
     ↓
Add Migration
     ↓
Update Database
     ↓
Add Another Entity
     ↓
Add Migration
     ↓
Update Database
```

EF Core Migrations allow the database schema to evolve **incrementally while preserving existing data**.

---

# 🧠 How Migrations Work

```text
Developer changes EF Core Model
             ↓
      Add Migration
             ↓
EF compares:
Current Model
      vs
Previous Model Snapshot
             ↓
     Detects Differences
             ↓
   Generates Migration
             ↓
   Store in Source Control
             ↓
    Apply to Database
             ↓
Migration History Table
tracks what was applied
```

### Important Components

- **Migration files** describe schema changes.
- **Model snapshot** represents the previous model state used for comparison.
- **Migration history table** records migrations already applied to a database.

> ⭐ Migration files can be committed to **source control** just like normal source files.

---

# 🛠️ Getting Started with Migrations

Suppose the model starts as:

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

Create the first migration.

### .NET CLI

```bash
dotnet ef migrations add InitialCreate
```

### Visual Studio Package Manager Console

```powershell
Add-Migration InitialCreate
```

EF Core creates a `Migrations` directory containing the generated migration files.

```text
Project
│
├── DbContext
├── Entities
└── Migrations
    ├── InitialCreate
    └── ModelSnapshot
```

> 💡 It is good practice to inspect generated migrations and amend them when necessary.

---

# 🏗️ Create / Update Database

Apply migrations using:

### .NET CLI

```bash
dotnet ef database update
```

### Visual Studio

```powershell
Update-Database
```

```text
Migration
   ↓
Database Update
   ↓
Schema matches EF Model
```

> ⚠️ Applying migrations this way is convenient for **local development**, but is generally less suitable as the production deployment strategy.

---

# ✏️ Evolving the Model

Suppose the model changes:

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime CreatedTimestamp { get; set; }
}
```

Now:

```text
EF Model
   ≠
Database Schema
```

Create another migration:

### .NET CLI

```bash
dotnet ef migrations add AddBlogCreatedTimestamp
```

### Visual Studio

```powershell
Add-Migration AddBlogCreatedTimestamp
```

Then apply it:

```bash
dotnet ef database update
```

or:

```powershell
Update-Database
```

---

# 🧠 What Happens During the Second Migration?

```text
Updated EF Model
      +
Previous Model Snapshot
      ↓
Compare
      ↓
Detect New Property
      ↓
Generate Migration
      ↓
Add CreatedTimestamp Column
```

Because the database already exists:

```text
Database
   ↓
Migration History Table
   ↓
Which migrations already applied?
   ↓
Apply only pending migrations
```

> ⭐ Give migrations **descriptive names** such as `AddBlogCreatedTimestamp` so the project history remains understandable.

---

# 🗂️ Migration History

EF Core records applied migrations in a special **migration history table**.

```text
Database
   │
   └── Migration History
          │
          ├── InitialCreate ✅
          ├── AddBlogCreatedTimestamp ✅
          └── FutureMigration ⬜
```

When updating:

```text
Applied migrations
       ↓
Skip

Pending migrations
       ↓
Apply
```

This prevents EF Core from unnecessarily reapplying migrations.

---

# 🔀 Migrations vs Reverse Engineering

```text
                 DATABASE / MODEL SYNC
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
       Model is Truth         DB is Truth
              │                     │
              ▼                     ▼
         Migrations          Reverse Engineering
              │                     │
              ▼                     ▼
       Model → Database       Database → Model
```

### Simple Mental Model

```text
Migrations
→ "My C# model changed, update the database."


Reverse Engineering
→ "My database already exists, create the EF model from it."
```

---

# 🚫 Excluding Parts of the Model from Migrations

Sometimes an application references an entity type managed by another `DbContext`.

This can cause migration conflicts.

Example:

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<IdentityUser>()
        .ToTable(
            "AspNetUsers",
            t => t.ExcludeFromMigrations());
}
```

```text
Entity exists in EF Model
        ↓
Can still be used by this DbContext
        ↓
But migrations do NOT manage its table
```

> ⭐ This is useful when multiple contexts share or reference a database table that should be managed by only one context.

---

# 🧠 Migration Lifecycle

```text
1. Change Entity Model
        ↓
2. Add Migration
        ↓
3. EF compares Model + Snapshot
        ↓
4. Migration generated
        ↓
5. Review / Source Control
        ↓
6. Apply Migration
        ↓
7. Database Schema Updated
        ↓
8. Migration History Recorded
```

---

# ⭐ Key Rules to Remember

```text
1. EF Core provides Migrations and Reverse Engineering
   for keeping model and database schema synchronized.

2. Migrations are used when the EF Core model is the
   source of truth.

3. Reverse Engineering is used when the database schema
   is the source of truth.

4. Migrations incrementally update the database while
   preserving existing data.

5. EF Core compares the current model with the previous
   model snapshot to generate migration changes.

6. Migration files should normally be stored in source control.

7. A migration history table records which migrations
   have already been applied.

8. dotnet ef migrations add <Name>
   creates a migration.

9. dotnet ef database update
   applies pending migrations.

10. Migration names should be descriptive.

11. Local development can use database update directly.

12. Production environments generally require a more
    controlled migration deployment approach.

13. ExcludeFromMigrations() keeps an entity in the EF model
    while preventing its table from being managed by migrations.

14. Multiple DbContexts may need migration exclusions when
    they reference tables managed by another context.
```

---

# ⚡ 30-Second Revision

```text
MODEL → DATABASE
────────────────

EF Model is Truth
       ↓
   Migration
       ↓
Compare Snapshot
       ↓
Generate Changes
       ↓
Update Database
       ↓
History Table


DATABASE → MODEL
────────────────

Database is Truth
       ↓
Reverse Engineering
       ↓
Scaffold DbContext
       +
Entity Classes


MIGRATION FLOW
──────────────

Model Change
    ↓
Add Migration
    ↓
Migration Files
    ↓
Database Update
    ↓
Migration History
```

> 🎯 **Core takeaway:** Decide the **source of truth** first. **Migrations** move schema changes from the EF Core model to the database, while **Reverse Engineering** moves an existing database schema into an EF Core model. Migrations work incrementally by comparing the current model with the previous snapshot and recording applied migrations in the database.