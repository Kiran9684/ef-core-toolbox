# 🧰 EF Core Interview Toolbox — Topic 3: Basic SaveChanges

```text
EF CORE TOOLBOX
│
├── SAVING DATA — OVERVIEW
│
│   Saving data means:
│
│   • Adding new entities.
│   • Updating existing entities.
│   • Removing existing entities.
│
│   EF Core provides two main approaches:
│
│       1. Change Tracking + SaveChanges()
│
│       2. ExecuteUpdate() / ExecuteDelete()
│          → Performs operations without normal change tracking.
│
│   ⭐ THIS TOPIC:
│   → Focuses on Change Tracking + SaveChanges().
│
│
├── CORE MENTAL MODEL
│
│       Query / Create Entity
│              ↓
│       DbContext Tracks Changes
│              ↓
│       Modify / Add / Remove
│              ↓
│       SaveChanges()
│              ↓
│       EF Detects Changes
│              ↓
│       SQL Generated
│              ↓
│       Database Updated
│
│
├── HOW SAVECHANGES() WORKS
│
│   Typical Unit of Work:
│
│       1. Query Entity
│              ↓
│       2. Entity becomes Tracked
│              ↓
│       3. Modify .NET Properties
│              ↓
│       4. Call SaveChanges()
│              ↓
│       5. EF Detects Changes
│              ↓
│       6. Changes Persisted to DB
│
│   ⭐ IMPORTANT:
│
│   • EF Core queries are tracking by default.
│   • EF stores information about tracked entities internally.
│   • You modify entity properties normally using C#.
│   • EF compares the current entity state with its tracked state.
│   • SaveChanges() generates the required database operations.
│
│
├── ADDING DATA
│
│   Use:
│
│       DbSet<TEntity>.Add()
│
│   Flow:
│
│       Create Entity
│             ↓
│       Add()
│             ↓
│       Entity State = Added
│             ↓
│       SaveChanges()
│             ↓
│       SQL INSERT
│
│   ⭐ Example:
│
│       var blog = new Blog
│       {
│           Url = "http://example.com"
│       };
│
│       context.Blogs.Add(blog);
│
│       await context.SaveChangesAsync();
│
│
├── UPDATING DATA
│
│   Load a tracked entity → Modify its properties → SaveChanges().
│
│   Flow:
│
│       Query Entity
│             ↓
│       Entity State = Tracked
│             ↓
│       Modify Property
│             ↓
│       SaveChanges()
│             ↓
│       SQL UPDATE
│
│   ⭐ Example:
│
│       var blog = await context.Blogs
│           .SingleAsync(b => b.Url == "http://example.com");
│
│       blog.Url = "http://example.com/blog";
│
│       await context.SaveChangesAsync();
│
│   ⭐ KEY POINT:
│
│   • You do NOT need to manually tell EF:
│       "This property has changed."
│
│   • EF Core detects changes automatically.
│
│
├── DELETING DATA
│
│   Use:
│
│       DbSet<TEntity>.Remove()
│
│   Flow:
│
│       Query Entity
│             ↓
│       Remove()
│             ↓
│       Entity State = Deleted
│             ↓
│       SaveChanges()
│             ↓
│       SQL DELETE
│
│   ⭐ Example:
│
│       var blog = await context.Blogs
│           .SingleAsync(b => b.Url == "http://example.com/blog");
│
│       context.Blogs.Remove(blog);
│
│       await context.SaveChangesAsync();
│
│   ⭐ IMPORTANT:
│
│   • Existing database entity
│       → Deleted from the database.
│
│   • Entity that was Added but not yet saved
│       → Removed from the DbContext.
│       → INSERT will not occur.
│
│
├── MULTIPLE OPERATIONS — ONE SAVECHANGES()
│
│   You can combine multiple operations:
│
│       Add
│        +
│       Update
│        +
│       Remove
│
│             ↓
│
│       SaveChanges()
│
│             ↓
│
│       All tracked changes are applied.
│
│   ⭐ Example:
│
│       // ADD
│       context.Blogs.Add(
│           new Blog { Url = "http://example.com/new" });
│
│       // UPDATE
│       var firstBlog = await context.Blogs.FirstAsync();
│       firstBlog.Url = "http://updated.com";
│
│       // DELETE
│       var lastBlog = await context.Blogs.LastAsync();
│       context.Blogs.Remove(lastBlog);
│
│       // SAVE EVERYTHING
│       await context.SaveChangesAsync();
│
│
├── WHY SAVECHANGES() IS POWERFUL
│
│   ⭐ Automatic Change Detection
│
│   • EF determines what entities/properties changed.
│   • Reduces manual change-tracking code.
│
│   ⭐ Relationship Handling
│
│   • EF can manage complex insert/update ordering.
│   • Example:
│
│       Blog
│         ↓
│       Generated BlogId
│         ↓
│       Insert Related Posts
│
│
│   ⭐ Concurrency Detection
│
│   • Can detect conflicts when database data changes
│     between querying and saving.
│
│
│   ⭐ Transactions
│
│   • For most database providers, SaveChanges() is transactional.
│
│       All Operations
│              ↓
│       SaveChanges()
│              ↓
│       ┌─────────────────┐
│       │ All Succeed  ✅ │
│       │       OR        │
│       │ All Fail     ❌ │
│       └─────────────────┘
│
│   → Operations are not left partially applied.
│
│
│   ⭐ Batching
│
│   • EF Core can batch multiple database commands.
│   • Reduces database round trips.
│   • Improves performance.
│
│
├── ENTITY STATE CONNECTION
│
│       Add()
│         ↓
│       Added
│         ↓
│       INSERT
│
│       Modify Tracked Entity
│         ↓
│       Modified
│         ↓
│       UPDATE
│
│       Remove()
│         ↓
│       Deleted
│         ↓
│       DELETE
│
│   ⭐ SaveChanges() converts tracked entity state changes
│     into database operations.
│
│
└── 🧠 30-SECOND MENTAL MODEL

    Change Tracking
    → EF remembers entities and monitors their changes.

    Add()
    → Marks a new entity for INSERT.

    Modify a Tracked Entity
    → EF detects changes and performs UPDATE.

    Remove()
    → Marks an entity for DELETE.

    SaveChanges()
    → Detects tracked changes and pushes them to the database.

    Multiple Operations
    → Add + Update + Remove can be saved together.

    Transaction
    → For most providers, SaveChanges() ensures operations
      succeed together or fail together.

    Golden Flow
    → Track → Change → Save → EF Generates SQL.
```
