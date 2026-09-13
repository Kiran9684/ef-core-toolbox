# 🧰 EF Core Interview Toolbox — Topic 28: Migrations in Team Environments

> **Category:** 🗂️ Managing Database Schemas  
> **File Name:** `28-migrations-team-environments.md`

---

# 👥 Migrations in Team Environments

## 🔎 Core Concept

When multiple developers work with EF Core Migrations, the **model snapshot** becomes especially important.

```text
Developer A
    ↓
Migration A
    ↓
Model Snapshot
    ↑
Migration B
    ↑
Developer B
```

The snapshot represents the model state that EF Core uses when creating the next migration.

> ⭐ When working in a team, pay close attention to the **model snapshot file**. It can reveal whether migrations from different branches merge cleanly or whether a migration needs to be recreated.

---

# 🌿 Why Team Migrations Can Conflict

Imagine:

```text
                 Main Branch
                     │
              ┌──────┴──────┐
              │             │
          Developer A   Developer B
              │             │
          Migration A   Migration B
              │             │
              └──────┬──────┘
                     ↓
                   Merge
                     ↓
              Model Snapshot
```

Both developers may have changed the model independently.

The migration files themselves may be fine, but the **snapshot can conflict** because both branches modified the model differently.

---

# 🔀 1. Merging Independent Changes

Suppose both developers changed **different properties**.

### Conflict

```text
<<<<<<< Mine
b.Property<bool>("Deactivated");
=======
b.Property<int>("LoyaltyPoints");
>>>>>>> Theirs
```

Both changes are valid and should exist in the final model.

### Resolve

```csharp
b.Property<bool>("Deactivated");
b.Property<int>("LoyaltyPoints");
```

```text
Mine
 └── Deactivated

Theirs
 └── LoyaltyPoints

      ↓ Merge

Final Model
 ├── Deactivated
 └── LoyaltyPoints
```

> ✅ When the changes are independent, the migrations are also independent. Either migration can generally be applied first, so no migration recreation is required.

---

# ⚔️ 2. Resolving a True Conflict

A real conflict occurs when both developers changed the **same model element in incompatible ways**.

Example:

```text
<<<<<<< Mine
b.Property<string>("Username");
=======
b.Property<string>("Alias");
>>>>>>> Theirs
```

Here, both migrations attempt to rename the same property differently.

```text
Developer A
Username

Developer B
Alias

       ↓
❌ True Conflict
```

Simply merging both lines is not correct.

---

# 🛠️ Correct Resolution Strategy

When the migration itself is affected by the conflict:

```text
1. Abort the merge
        ↓
2. Return to the working state before the merge
        ↓
3. Remove your migration
   (KEEP your model changes)
        ↓
4. Merge your teammate's changes
        ↓
5. Re-add your migration
        ↓
6. Commit / share the recreated migration
```

### 🧠 Flow

```text
Your Migration
      ↓
Abort Merge
      ↓
Remove Migration
      ↓
Keep Model Changes
      ↓
Merge Teammate Changes
      ↓
Recreate Migration
      ↓
Correct Migration Order
```

---

# 🔄 Why Recreate the Migration?

Suppose:

```text
Teammate Migration
Username → Alias

Your Model Change
Alias → Username
```

After recreating your migration:

```text
Migration 1
Username → Alias
       ↓
Migration 2
Alias → Username
```

```text
Database
   ↓
Apply teammate migration
   ↓
Alias
   ↓
Apply recreated migration
   ↓
Username
```

> ⭐ Recreating the migration ensures the migration chain represents the **actual order of model changes**.

---

# 🧠 Independent vs Conflicting Changes

| Situation | Example | Action |
|---|---|---|
| Independent changes | A adds `Deactivated`, B adds `LoyaltyPoints` | Merge snapshot changes |
| Same model element, conflicting changes | A → `Username`, B → `Alias` | Recreate affected migration |

```text
Independent
    ↓
Merge
    ↓
✅ Continue


True Conflict
    ↓
Recreate Migration
    ↓
✅ Correct Migration Chain
```

---

# 🆕 EF Core 11+ — Detecting Diverged Migration Trees

Starting with **EF Core 11 preview 3**, the model snapshot records the ID of the latest migration.

```text
Branch A
  ↓
Migration A
  ↓
Snapshot → Migration A

Branch B
  ↓
Migration B
  ↓
Snapshot → Migration B
```

When both branches are merged:

```text
Snapshot A
    +
Snapshot B
    ↓
Source Control Conflict
    ↓
🚨 Migration Trees Diverged
```

> ⭐ The conflict is intentional: it signals that migrations were created independently from different migration histories.

The required approach is to:

```text
Discard one migration
        ↓
Recreate it from the merged model
```

### Earlier EF Core Versions

With **EF Core 10 and earlier**:

```text
No automatic snapshot conflict signal
        ↓
But migration-tree divergence can still happen
        ↓
Manual care is required
```

---

# 🧠 Migration Branching Mental Model

```text
                 Last Shared Migration
                         │
                  ┌──────┴──────┐
                  │             │
                  ▼             ▼
              Branch A       Branch B
                  │             │
             Migration A    Migration B
                  │             │
                  └──────┬──────┘
                         ↓
                       Merge
                         │
              ┌──────────┴──────────┐
              │                     │
        Independent             Diverged
         Changes                 Tree
              │                     │
              ▼                     ▼
          Merge Normally       Recreate One
```

---

# ⭐ Key Rules to Remember

```text
1. The model snapshot is especially important
   when multiple developers work with migrations.

2. Migration files and the model snapshot should be
   tracked in source control.

3. Independent model changes can usually be merged directly.

4. If both developers changed unrelated properties,
   both changes should remain in the final snapshot.

5. Independent migrations can generally coexist and
   may be applied in either order.

6. A true conflict occurs when migrations represent
   incompatible changes to the same part of the model.

7. For a true conflict:
   → Abort merge
   → Remove your migration
   → Keep model changes
   → Merge teammate changes
   → Re-add migration

8. Recreating the migration ensures the migration history
   matches the final model-change sequence.

9. EF Core 11+ records the latest migration ID in the model snapshot.

10. A snapshot conflict in EF Core 11+ can signal that
    migration trees diverged across branches.

11. EF Core 10 and earlier may not automatically flag
    this divergence, but the underlying problem still exists.
```

---

# ⚡ 30-Second Revision

```text
TEAM MIGRATIONS
───────────────

Developer A ──→ Migration A ──┐
                              │
                              ▼
                           Snapshot
                              ▲
                              │
Developer B ──→ Migration B ──┘


UNRELATED CHANGES
─────────────────

A → Deactivated
B → LoyaltyPoints
      ↓
Merge Both
      ↓
✅ No Recreation


TRUE CONFLICT
─────────────

A → Username
B → Alias
      ↓
❌ Conflicting Model Changes
      ↓
Remove Your Migration
      ↓
Merge Teammate
      ↓
Re-add Migration
      ↓
✅ Correct Migration Chain


EF CORE 11+
───────────

Snapshot records latest Migration ID
      ↓
Diverged branches
      ↓
Snapshot conflict
      ↓
🚨 Recreate one migration
```

> 🎯 **Core takeaway:** In team environments, the **model snapshot is the key signal for migration consistency**. Unrelated changes can normally be merged directly, but when migration histories diverge or the same model change conflicts, **remove and recreate the affected migration from the merged model**.