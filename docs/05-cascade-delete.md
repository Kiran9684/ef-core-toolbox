# 🧰 EF Core Interview Toolbox — Topic 5: Cascade Delete

```text
EF CORE TOOLBOX
│
├── 🔎 CORE CONCEPT
│
│   Relationships in EF Core are enforced using Foreign Keys.
│
│       Principal / Parent
│              │
│              │ Primary Key
│              ↓
│       Dependent / Child
│              │
│              └── Foreign Key
│
│   Problem:
│
│       Blog Deleted
│             ↓
│       Posts still reference BlogId
│             ↓
│       ❌ Referential Constraint Violation
│
│   EF Core avoids this using:
│
│       1. Cascade Delete
│       2. Set Foreign Key to NULL
│
│
├── 1️⃣ CASCADE DELETE
│
│   • Default behavior for required relationships.
│   • Required relationship = non-nullable Foreign Key.
│
│   🧠 Flow:
│
│       Delete Blog
│            ↓
│       Related Posts Found
│            ↓
│       Delete Posts
│            ↓
│       Delete Blog
│
│   ⭐ Example:
│
│       var blog = await context.Blogs
│           .Include(b => b.Posts)
│           .FirstAsync();
│
│       context.Remove(blog);
│
│       await context.SaveChangesAsync();
│
│   Result:
│
│       DELETE Posts
│       DELETE Blog
│
│
├── 2️⃣ ORPHAN DELETION
│
│   • Happens when the parent still exists,
│     but the relationship is removed.
│
│   ⭐ Examples:
│
│       post.Blog = null;
│
│   OR:
│
│       blog.Posts.Remove(post);
│
│   OR:
│
│       blog.Posts.Clear();
│
│   🧠 Flow:
│
│       Blog Exists
│            ↓
│       Relationship Removed
│            ↓
│       Post becomes Orphan
│            ↓
│
│       Required Relationship?
│
│            ↓ YES
│
│       DELETE Post
│
│   ⭐ KEY DIFFERENCE:
│
│       Cascade Delete
│       → Parent is deleted.
│
│       Orphan Delete
│       → Parent exists, relationship is removed.
│
│
├── REQUIRED vs OPTIONAL RELATIONSHIP
│
│   REQUIRED
│
│       public int BlogId { get; set; }
│
│   • FK cannot be NULL.
│   • Default → Cascade Delete.
│
│       Relationship Removed
│              ↓
│       Dependent cannot exist without Parent
│              ↓
│       DELETE Dependent
│
│
│   OPTIONAL
│
│       public int? BlogId { get; set; }
│
│   • FK can be NULL.
│   • Default → ClientSetNull.
│
│       Relationship Removed
│              ↓
│       BlogId = NULL
│              ↓
│       Dependent Survives
│
│
├── 🧠 CASCADE DELETE vs CASCADE NULL
│
│   REQUIRED FK
│
│       Blog Deleted
│            ↓
│       Post Deleted
│
│
│   OPTIONAL FK
│
│       Blog Deleted
│            ↓
│       Post.BlogId = NULL
│            ↓
│       Post Survives
│
│
├── WHERE DOES CASCADING HAPPEN?
│
│   Cascade behavior can happen in TWO places:
│
│       1. EF Core / Change Tracker
│       2. Database / Foreign Key Constraint
│
│
├── 1️⃣ CLIENT-SIDE CASCADING — EF CORE
│
│   EF Core handles cascading for tracked entities.
│
│       DbContext
│           │
│           ├── Blog (Tracked)
│           └── Posts (Tracked)
│
│                 ↓
│
│           Delete Blog
│
│                 ↓
│
│           EF Core processes dependents
│
│                 ↓
│
│           DELETE Posts
│           DELETE Blog
│
│   ⭐ Cascade timing can be controlled using:
│
│       ChangeTracker.CascadeDeleteTiming
│
│       ChangeTracker.DeleteOrphansTiming
│
│   ⚠ IMPORTANT:
│
│   Client-side cascading requires EF Core to know about
│   the dependent entities (typically they must be tracked).
│
│
├── 2️⃣ DATABASE-SIDE CASCADING
│
│   Database FK can be configured as:
│
│       ON DELETE CASCADE
│
│   🧠 Flow:
│
│       DELETE Blog
│            ↓
│       Database detects FK constraint
│            ↓
│       Database deletes Posts automatically
│
│   ⭐ Advantage:
│
│   Dependents do NOT need to be loaded into DbContext.
│
│
├── TRACKED vs NOT LOADED
│
│   ⭐ DEPENDENTS TRACKED
│
│       EF Core
│          ↓
│       Can apply configured client behavior
│
│
│   ⭐ DEPENDENTS NOT LOADED
│
│       EF Core
│          ↓
│       Database must handle the relationship
│
│
│   If DB has:
│
│       ON DELETE CASCADE
│
│       → Database deletes dependents.
│
│   Otherwise:
│
│       → ❌ Foreign Key violation
│       → DbUpdateException
│
│
├── DATABASE vs EF CORE
│
│   Database knows:
│
│       • Foreign Keys
│       • ON DELETE CASCADE
│       • ON DELETE SET NULL
│
│   Database DOES NOT know:
│
│       • Navigation properties
│       • blog.Posts.Remove(post)
│       • post.Blog = null
│
│   ⭐ Therefore:
│
│   Orphan deletion is primarily handled by EF Core's
│   Change Tracker, not by the database.
│
│
├── SOFT DELETE WARNING ⚠
│
│   Soft Delete:
│
│       IsDeleted = true
│
│   Database Cascade Delete:
│
│       ❌ Physically deletes dependents.
│
│   ⭐ Rule:
│
│   Be careful with ON DELETE CASCADE when implementing
│   soft-delete behavior.
│
│
├── 3️⃣ CASCADE CYCLES / MULTIPLE CASCADE PATHS
│
│   SQL Server restricts cascade configurations that create
│   multiple paths to the same dependent table.
│
│   Example:
│
│       Person
│        │   \
│        │    \
│        ↓     ↓
│      Blog   Posts
│        │
│        ↓
│      Posts
│
│   Delete Person:
│
│       Path 1:
│       Person → Blog → Posts
│
│       Path 2:
│       Person → Posts
│
│   ❌ Two cascade paths reach Posts.
│
│   SQL Server may reject this schema configuration.
│
│
├── FIX 1️⃣ — MAKE A RELATIONSHIP OPTIONAL
│
│   Example:
│
│       public int? BlogId { get; set; }
│
│   Before:
│
│       Person → Blog → Posts
│       Person ─────────→ Posts
│
│       ❌ Multiple cascade paths
│
│   After Blog relationship becomes optional:
│
│       Person → Blog
│                  │
│                  └── BlogId = NULL
│
│       Person ───────→ Posts
│                     (Cascade)
│
│   ⭐ Result:
│
│   Only one cascade path deletes Posts.
│
│
├── FIX 2️⃣ — CLIENT-SIDE CASCADE
│
│   Configure:
│
│       modelBuilder
│           .Entity<Blog>()
│           .HasOne(e => e.Owner)
│           .WithOne(e => e.OwnedBlog)
│           .OnDelete(DeleteBehavior.ClientCascade);
│
│   🧠 Mental Model:
│
│       Database
│       → Does NOT cascade this relationship.
│
│       EF Core
│       → Handles cascading for tracked entities.
│
│   Flow:
│
│       Parent + Dependents Loaded
│                  ↓
│           Delete Parent
│                  ↓
│       EF Core deletes dependents
│                  ↓
│       SQL DELETE statements generated
│
│   ⚠ IMPORTANT:
│
│       Parent deleted
│       BUT dependents not loaded/tracked
│                  ↓
│       Database cannot ClientCascade
│                  ↓
│       ❌ FK Constraint Violation
│
│
├── CASCADING NULLS
│
│   Used mainly with optional relationships.
│
│       public int? BlogId { get; set; }
│
│   🧠 Flow:
│
│       Delete Blog
│            ↓
│       Post remains
│            ↓
│       Post.BlogId = NULL
│
│   ⭐ EF Core default for optional relationships:
│
│       DeleteBehavior.ClientSetNull
│
│
├── CONFIGURING DELETE BEHAVIOR
│
│   Configure per relationship:
│
│       modelBuilder
│           .Entity<Blog>()
│           .HasOne(...)
│           .WithMany(...)
│           .OnDelete(DeleteBehavior.Cascade);
│
│
├── DELETEBEHAVIOR — QUICK REFERENCE
│
│   DeleteBehavior
│
│   ├── Cascade
│   │     → EF Core cascades tracked entities
│   │     → DB uses ON DELETE CASCADE
│   │
│   ├── SetNull
│   │     → FK set to NULL
│   │     → DB uses ON DELETE SET NULL
│   │
│   ├── Restrict / NoAction
│   │     → Database prevents invalid delete
│   │
│   ├── ClientSetNull
│   │     → EF Core sets tracked FK to NULL
│   │     → DB does not cascade
│   │
│   ├── ClientCascade
│   │     → EF Core deletes tracked dependents
│   │     → DB does not cascade
│   │
│   └── ClientNoAction
│         → EF Core does not automatically fix relationship
│
│
├── DATABASE SCHEMA IMPACT
│
│   ┌─────────────────────┬─────────────────────────┐
│   │ DeleteBehavior      │ Database FK Behavior    │
│   ├─────────────────────┼─────────────────────────┤
│   │ Cascade             │ ON DELETE CASCADE       │
│   │ SetNull             │ ON DELETE SET NULL      │
│   │ Restrict            │ ON DELETE RESTRICT*     │
│   │ NoAction            │ NO ACTION / Default     │
│   │ ClientSetNull       │ No DB cascade           │
│   │ ClientCascade       │ No DB cascade           │
│   │ ClientNoAction      │ No DB cascade           │
│   └─────────────────────┴─────────────────────────┘
│
│   * Provider-specific behavior may vary.
│
│
├── REQUIRED RELATIONSHIP — GOLDEN RULE
│
│       FK = NOT NULL
│
│              ↓
│
│       Dependent requires Principal
│
│              ↓
│
│       Cannot simply set FK to NULL
│
│              ↓
│
│       Usually:
│
│       • Cascade Delete
│       OR
│       • Database throws FK violation
│
│
├── OPTIONAL RELATIONSHIP — GOLDEN RULE
│
│       FK = NULLABLE
│
│              ↓
│
│       Dependent can survive without Principal
│
│              ↓
│
│       Default:
│
│       ClientSetNull
│
│              ↓
│
│       FK becomes NULL
│
│
├── ⚠ MOST IMPORTANT PITFALL
│
│   Client-side cascading is NOT the same as database cascading.
│
│   ClientCascade:
│
│       Dependents must be tracked
│
│   Database Cascade:
│
│       Database can delete dependents
│       even when they were never loaded.
│
│
└── 🧠 30-SECOND MENTAL MODEL

    Foreign Key
    → Connects Principal and Dependent entities.

    Required Relationship
    → FK cannot be NULL.
    → Default = Cascade.

    Optional Relationship
    → FK can be NULL.
    → Default = ClientSetNull.

    Cascade Delete
    → Delete Parent → Delete Dependents.

    Orphan Delete
    → Parent exists, but relationship is severed
      → Required dependent may be deleted.

    Client-Side Cascade
    → EF Core handles tracked dependents.

    Database Cascade
    → Database handles dependents using ON DELETE CASCADE.

    ClientCascade
    → Useful for avoiding database cascade cycles,
      but dependents must be loaded/tracked.

    Multiple Cascade Paths
    → SQL Server may reject the schema.

    Golden Rule
    → Required relationship = child cannot survive without parent.
    → Optional relationship = child can survive with FK = NULL.
```