# 🧰 EF Core Interview Toolbox — Topic 27: Script Generation

> **Category:** 🗂️ Managing Database Schemas  
> **File Name:** `27-script-generation.md`

---

# 📜 Script Generation

## 🔎 Core Concept

EF Core can generate **SQL scripts from migrations** instead of applying migrations directly.

```text
EF Core Migrations
        ↓
   Script Generation
        ↓
     SQL Script
        ↓
 Review / Test / Deploy
        ↓
    Database
```

> ⭐ For production deployments, generating a SQL script gives you a chance to **inspect, test, modify, and control** the database changes before execution.

---

# 1️⃣ Basic Script Generation

Generates SQL for **all migrations**, from an empty database to the latest migration.

## 📦 Package Manager Console

```powershell
Script-Migration `
    -Project MyPortfolio.DataAccess `
    -StartupProject MyPortfolio.Web
```

## 💻 .NET CLI

```bash
dotnet ef migrations script \
    --project MyPortfolio.DataAccess \
    --startup-project MyPortfolio.Web
```

```text
MyPortfolio.DataAccess
        ↓
Migrations + DbContext
        ↓
MyPortfolio.Web
        ↓
Startup / Configuration
        ↓
Generated SQL
```

---

# 2️⃣ Save the Script to a File

By default, the generated SQL is printed to the console.

For deployment, save it as a `.sql` file.

## 📦 Package Manager Console

```powershell
Script-Migration `
    -Project MyPortfolio.DataAccess `
    -StartupProject MyPortfolio.Web `
    -Output "MyDeploymentScript.sql"
```

## 💻 .NET CLI

```bash
dotnet ef migrations script \
    --project MyPortfolio.DataAccess \
    --startup-project MyPortfolio.Web \
    --output "MyDeploymentScript.sql"
```

```text
Migrations
    ↓
SQL Generation
    ↓
MyDeploymentScript.sql
    ↓
Review / Test
    ↓
Deploy
```

> 💡 Saving the script makes it easier to pass through deployment pipelines, DB review processes, or DBA workflows.

---

# 3️⃣ Production-Ready — Idempotent Script

An **idempotent script** checks which migrations have already been applied and executes only the missing migrations.

## 📦 Package Manager Console

```powershell
Script-Migration `
    -Project MyPortfolio.DataAccess `
    -StartupProject MyPortfolio.Web `
    -Output "MyDeploymentScript.sql" `
    -Idempotent
```

## 💻 .NET CLI

```bash
dotnet ef migrations script \
    --project MyPortfolio.DataAccess \
    --startup-project MyPortfolio.Web \
    --output "MyDeploymentScript.sql" \
    --idempotent
```

### 🧠 Mental Model

```text
MyDeploymentScript.sql
        ↓
Check __EFMigrationsHistory
        ↓
Migration already applied?
       / \
     YES  NO
      ↓    ↓
    Skip  Apply
```

> ⭐ Idempotent scripts are useful when different databases may already be at **different migration levels**.

---

# 🔑 Why `-Idempotent` Matters

Without idempotency:

```text
Database at Migration 2
        ↓
Script expects Migration 0
        ↓
May try to replay existing migrations
        ↓
❌ Problem
```

With idempotency:

```text
Database at Migration 2
        ↓
Check Migration History
        ↓
Migrations 1 & 2 → Skip
Migration 3 → Apply
```

---

# 🧩 N-Tier Project Flags

For a project such as:

```text
MyPortfolio
│
├── MyPortfolio.Web
│
└── MyPortfolio.DataAccess
```

the important flags are:

| Flag | Value | Purpose |
|---|---|---|
| `-Project` | `MyPortfolio.DataAccess` | Locates the migrations and `DbContext` |
| `-StartupProject` | `MyPortfolio.Web` | Provides the application's startup/configuration context |
| `-Output` | `MyDeploymentScript.sql` | Saves the generated SQL to a file |
| `-Idempotent` | Switch | Applies only migrations missing from the target database |

---

# 🧠 `Project` vs `StartupProject`

```text
-Project
    ↓
Where EF Core artifacts live
    ↓
DbContext + Migrations


-StartupProject
    ↓
Which application starts/configures EF
    ↓
Configuration / services / connection setup
```

> ⭐ In an N-tier application, the **migration project** and the **startup project** can be different projects.

---

# ⚡ Complete Production Flow

```text
Model Changes
      ↓
Create Migration
      ↓
dotnet ef migrations add ...
      ↓
Migration Files
      ↓
Generate SQL Script
      ↓
--idempotent
      ↓
MyDeploymentScript.sql
      ↓
Review / Test
      ↓
Deployment Pipeline / DBA
      ↓
Production Database
```

---

# ⭐ Key Rules to Remember

```text
1. Script generation converts EF Core migrations into SQL.

2. `Script-Migration` is the Package Manager Console command.

3. `dotnet ef migrations script` is the .NET CLI command.

4. Without an output option, the script is printed to the console.

5. `-Output` / `--output` saves the generated SQL to a file.

6. `-Project` identifies where the DbContext and migrations live.

7. `-StartupProject` identifies the application used for
   startup/configuration.

8. `-Idempotent` creates a script that checks migration history
   and applies only migrations that are missing.

9. Idempotent scripts are useful when target databases may be
   at different migration levels.

10. SQL scripts provide a controlled deployment mechanism because
    the generated SQL can be reviewed and tested before execution.
```

---

# 🧠 30-Second Mental Model

```text
                EF CORE MIGRATIONS
                       │
                       ▼
                Script Generation
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
         Normal Script      Idempotent
              │                 │
              │            Check History
              │                 │
              │            Skip Applied
              │                 │
              └────────┬────────┘
                       ▼
                .sql Deployment File
                       ↓
                 Review / Test
                       ↓
                   Production
```

> 🎯 **Core takeaway:** **Script Generation = Migrations → SQL file.** In an N-tier application, use `-Project` for the project containing the **DbContext/migrations**, `-StartupProject` for the application providing **startup/configuration**, and `-Idempotent` when the deployment script must safely handle databases that may already be at different migration states.