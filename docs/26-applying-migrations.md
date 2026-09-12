# 🧰 EF Core Interview Toolbox — Topic 26: Applying Migrations

> **Category:** 🗂️ Managing Database Schemas  
> **File Name:** `26-applying-migrations.md`

---

# 🚀 Applying Migrations

## 🔎 Core Concept

Once migrations are created, they must be **deployed and applied** to the target database.

```text
EF Core Model
     ↓
Migration Created
     ↓
Migration Deployment
     ↓
Database Schema Updated
```

> ⚠️ **Always inspect generated migrations and test them before applying them to production.** A migration may accidentally drop a column instead of renaming it, or fail when executed. :contentReference[oaicite:0]{index=0}

---

# 🏆 Production Recommendation — SQL Scripts

The recommended strategy for production is to **generate SQL scripts from migrations**.

```text
Migrations
    ↓
Generate SQL Script
    ↓
Review / Test
    ↓
Deploy
    ↓
Production Database
```

### Why SQL Scripts?

- ✅ Can be reviewed before execution.
- ✅ Can be modified or tuned for production requirements.
- ✅ Can be integrated into CI/CD pipelines.
- ✅ Can be given to DBAs and archived separately.

> ⭐ SQL scripts provide more control and visibility than directly applying migrations from the EF CLI. :contentReference[oaicite:1]{index=1}

---

# 🛠️ Generate Migration Scripts

## Generate All Migrations

### .NET CLI

```bash
dotnet ef migrations script
```

Generates SQL from an empty database to the latest migration.

### Visual Studio

```powershell
Script-Migration
```

---

# 🎯 Generate From a Specific Migration

```bash
dotnet ef migrations script AddNewTables
```

```powershell
Script-Migration AddNewTables
```

This generates SQL from:

```text
AddNewTables
      ↓
Latest Migration
```

---

# 🎯 Generate Between Two Migrations

```bash
dotnet ef migrations script AddNewTables AddAuditTable
```

```powershell
Script-Migration AddNewTables AddAuditTable
```

```text
AddNewTables
      ↓
AddAuditTable
```

> 💡 You can specify a **newer migration as `from` and an older migration as `to`** to generate a rollback script. :contentReference[oaicite:2]{index=2}

---

# 🧭 `From` and `To` Mental Model

```text
FROM
 ↓
Last migration already applied

TO
 ↓
Last migration that should be applied
```

If no migrations have been applied:

```text
FROM = 0
```

By default:

```text
TO = Latest migration
```

:contentReference[oaicite:3]{index=3}

---

# 🛡️ Idempotent Scripts

A normal migration script assumes the database is already at the expected migration state.

An **idempotent script** checks the migration history and applies only missing migrations.

```text
Database
   ↓
Check __EFMigrationsHistory
   ↓
Migration already applied?
   ┌─────────────┴─────────────┐
  YES                          NO
   ↓                            ↓
 Skip                       Apply
```

### .NET CLI

```bash
dotnet ef migrations script --idempotent
```

### Visual Studio

```powershell
Script-Migration -Idempotent
```

### Why Useful?

Especially useful when:

- You do not know the exact migration currently applied.
- Multiple databases may be at different migration levels.

> ⭐ Idempotent scripts are safer for environments where databases may not all be at the same migration. :contentReference[oaicite:4]{index=4}

---

# 🧰 Applying Migrations Directly with EF Tools

The EF command-line tools can also directly update the database.

```bash
dotnet ef database update
```

Or:

```powershell
Update-Database
```

To migrate to a specific migration:

```bash
dotnet ef database update AddNewTables
```

```powershell
Update-Database AddNewTables
```

You can also use this to roll back to an earlier migration.

```text
Current Migration
      ↓
Target Older Migration
      ↓
Rollback
```

> ⚠️ Be aware of potential **data-loss scenarios** when moving backward or changing schemas. :contentReference[oaicite:5]{index=5}

---

# ⚠️ Why Direct CLI Application Is Not Ideal for Production

Although:

```bash
dotnet ef database update
```

is convenient for local development and testing, it is less suitable for production because:

```text
EF Tool
   ↓
Directly executes SQL
   ↓
No review/modification before execution
```

Additional concerns:

- The EF tool and .NET SDK may need to be installed on the production server.
- The project's source code is required.
- SQL is executed directly instead of going through a controlled review process.

:contentReference[oaicite:6]{index=6}

---

# 📦 Migration Bundles

A **migration bundle** is a single executable that contains the migrations and can apply them to a database.

```text
Migrations
    ↓
Bundle
    ↓
Single Executable
    ↓
Deployment Environment
    ↓
Database
```

### Why Bundles?

They address some limitations of scripts and direct EF tooling:

- ✅ Single executable.
- ✅ Can be generated during CI.
- ✅ Easy to execute during deployment.
- ✅ Does not require project source code.
- ✅ Does not require the .NET SDK or EF Tool on the target machine.
- ✅ Can be self-contained, without requiring the .NET runtime.

:contentReference[oaicite:7]{index=7}

---

# 🛠️ Creating a Migration Bundle

### Basic

```bash
dotnet ef migrations bundle
```

### Self-Contained Linux Bundle

```bash
dotnet ef migrations bundle --self-contained -r linux-x64
```

The default executable name is:

```text
efbundle
```

On Windows:

```text
efbundle.exe
```

---

# ▶️ Running a Bundle

The generated bundle can be executed directly:

```bash
./efbundle
```

or:

```powershell
.\efbundle.exe
```

It applies migrations that have not already been applied.

```text
efbundle
   ↓
Check Migration History
   ↓
Pending migrations?
   ┌──────────┴──────────┐
  YES                    NO
   ↓                      ↓
Apply                  Do Nothing
```

Running the same bundle again when the database is already up to date does nothing.

:contentReference[oaicite:8]{index=8} :contentReference[oaicite:9]{index=9}

---

# 🔄 Updating the Bundle

When more migrations are added:

```text
New Migration
     ↓
dotnet ef migrations add
     ↓
Rebuild Bundle
     ↓
Deploy New Bundle
```

To overwrite an existing bundle:

```bash
dotnet ef migrations bundle --force
```

> ⭐ `--force` overwrites the existing bundle. :contentReference[oaicite:10]{index=10}

---

# 🔌 Using a Different Database

By default, the bundle uses the configured application connection string.

You can override it:

```bash
./efbundle --connection "Data Source=...;Database=Production"
```

```text
Bundle
  ↓
--connection
  ↓
Target Database
```

:contentReference[oaicite:11]{index=11}

---

# 🎯 Bundle Migration Target

A bundle can target a specific migration:

```text
efbundle <MIGRATION>
```

Special case:

```text
efbundle 0
```

means:

```text
Revert all migrations
```

The default target is the latest migration.

:contentReference[oaicite:12]{index=12}

---

# 🧩 Applying Migrations at Runtime

EF Core also allows the application itself to run migrations programmatically.

```csharp
await db.Database.MigrateAsync();
```

Typical flow:

```text
Application Starts
       ↓
Create DI Scope
       ↓
Resolve DbContext
       ↓
MigrateAsync()
       ↓
Database Updated
       ↓
Application Runs
```

Example:

```csharp
public static async Task Main(string[] args)
{
    var host = CreateHostBuilder(args).Build();

    using (var scope = host.Services.CreateScope())
    {
        var db = scope.ServiceProvider
            .GetRequiredService<ApplicationDbContext>();

        await db.Database.MigrateAsync();
    }

    host.Run();
}
```

`MigrateAsync()` is built on the `IMigrator` service for more advanced scenarios.

:contentReference[oaicite:13]{index=13}

---

# ⚠️ Why Runtime Migration Is Usually Not Recommended for Production

```text
Application
     ↓
Startup
     ↓
MigrateAsync()
     ↓
Production Schema Change
```

Potential problems include:

- Multiple application instances may try to migrate simultaneously.
- The application may access the database while another migration is running.
- The application requires elevated database permissions.
- Rollback/control is less convenient.
- SQL cannot be reviewed or modified before execution.

> ⭐ For production, generating SQL scripts or using controlled migration deployment is generally preferred. :contentReference[oaicite:14]{index=14}

---

# 🚫 `EnsureCreated()` + `MigrateAsync()`

Do **not** call:

```csharp
await context.Database.EnsureCreatedAsync();
```

before:

```csharp
await context.Database.MigrateAsync();
```

Why?

```text
EnsureCreated()
      ↓
Creates schema without migrations
      ↓
Migration History not established correctly
      ↓
MigrateAsync()
      ↓
❌ Problem
```

> ⚠️ `EnsureCreatedAsync()` bypasses migrations, so combining it with `MigrateAsync()` can cause migration failures. :contentReference[oaicite:15]{index=15}

---

# 🏆 Deployment Strategy Comparison

| Strategy | Best Fit | Key Characteristic |
|---|---|---|
| `database update` | Local development | Directly applies migrations |
| SQL Script | ⭐ Production | Reviewable and controllable |
| Idempotent SQL Script | Multiple/unknown DB states | Applies only missing migrations |
| Migration Bundle | Controlled deployment | Single executable |
| Runtime `MigrateAsync()` | Development/testing | Application performs migration |

---

# 🧠 Migration Deployment Decision Tree

```text
Need to apply EF Core migrations?
              │
              ▼
        Production?
        ┌─────┴─────┐
       NO          YES
       │            │
       ▼            ▼
database update   Controlled
                  deployment
                      │
             ┌────────┴────────┐
             ▼                 ▼
          SQL Script        Bundle
             │
       Review / Test
             │
             ▼
         Production
```

---

# ⭐ Key Rules to Remember

```text
1. Always inspect and test migrations before production.

2. SQL scripts are the recommended production deployment approach.

3. `dotnet ef migrations script`
   generates SQL from migrations.

4. `from` defines the migration the database is currently at.

5. `to` defines the migration the database should reach.

6. `from = 0` means no migrations have been applied.

7. A newer `from` and older `to` can generate a rollback script.

8. Idempotent scripts check migration history and apply
   only migrations that are missing.

9. `dotnet ef database update` is convenient for local
   development and testing.

10. Direct CLI migration application is less suitable for
    production because SQL is executed without review.

11. Migration bundles package migrations into a single executable.

12. Bundles can be generated during CI/CD.

13. Bundles can run without the project source code or EF Tool.

14. `--force` overwrites an existing migration bundle.

15. A bundle can target a specific migration.

16. Migration target `0` reverts all migrations.

17. `--connection` can override the configured database connection.

18. `MigrateAsync()` can apply migrations programmatically.

19. Runtime migration is generally discouraged for production
    because of concurrency, permissions, access, and control concerns.

20. Never use `EnsureCreatedAsync()` before `MigrateAsync()`.
```

---

# ⚡ 30-Second Revision

```text
             APPLYING MIGRATIONS
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
       SQL Script  Bundle   CLI Update
          │          │          │
          │          │          └── Local / Testing
          │          │
          │          └── Deployment Executable
          │
          └── ⭐ Preferred Production Strategy


SQL SCRIPT
──────────
Migration
   ↓
Generate SQL
   ↓
Review
   ↓
Test
   ↓
Deploy


IDEMPOTENT
──────────
History Table
    ↓
Already Applied?
    ↓
YES → Skip
NO  → Apply


BUNDLE
──────
Migrations
   ↓
efbundle
   ↓
Target Database
   ↓
Apply Pending Migrations


RUNTIME
───────
App Startup
   ↓
MigrateAsync()
   ↓
Database

⚠️ Usually avoid for Production
```

> 🎯 **Core takeaway:** Creating migrations and **applying migrations are separate steps**. For production, prefer a controlled deployment using **reviewable SQL scripts or migration bundles**. Use `database update` mainly for development/testing, and be cautious with runtime `MigrateAsync()` because the application itself then becomes responsible for modifying the production schema.