# 🧰 EF Core Interview Toolbox — Topic 4: Saving Related Data

```text
EF CORE TOOLBOX
│
├── SAVING RELATED DATA — CORE IDEA
│
│   • EF Core can save not only isolated entities,
│     but also entities connected through relationships.
│
│   • EF Core uses:
│
│       Navigation Properties
│               +
│       Change Tracking
│               +
│       Entity Relationships
│
│   → To detect and persist changes across related entities.
│
│
├── ADDING A GRAPH OF NEW ENTITIES
│
│   • A graph = multiple related entities connected together.
│
│   • Adding one root entity can cause EF Core to discover
│     and add reachable new related entities.
│
│   🧠 Mental Model:
│
│       Blog
│        │
│        ├── Post 1
│        ├── Post 2
│        └── Post 3
│
│             ↓
│
│       context.Blogs.Add(blog)
│
│             ↓
│
│       EF Core discovers the graph
│
│             ↓
│
│       Blog + Posts marked as Added
│
│             ↓
│
│       SaveChanges()
│
│             ↓
│
│       INSERT Blog
│       INSERT Posts
│
│   ⭐ Example:
│
│       var blog = new Blog
│       {
│           Url = "http://blogs.msdn.com/dotnet",
│           Posts = new List<Post>
│           {
│               new Post { Title = "Intro to C#" },
│               new Post { Title = "Intro to VB.NET" },
│               new Post { Title = "Intro to F#" }
│           }
│       };
│
│       context.Blogs.Add(blog);
│       await context.SaveChangesAsync();
│
│
├── ADDING A RELATED ENTITY
│
│   • A new related entity can be added through the navigation
│     property of an entity already tracked by DbContext.
│
│   🧠 Flow:
│
│       Existing Blog (Tracked)
│                ↓
│       Add New Post
│       to Blog.Posts
│                ↓
│       EF discovers new Post
│                ↓
│       Post marked as Added
│                ↓
│       SaveChanges()
│                ↓
│       INSERT Post
│
│   ⭐ Example:
│
│       var blog = await context.Blogs
│           .Include(b => b.Posts)
│           .FirstAsync();
│
│       var post = new Post
│       {
│           Title = "Intro to EF Core"
│       };
│
│       blog.Posts.Add(post);
│
│       await context.SaveChangesAsync();
│
│
├── CHANGING RELATIONSHIPS
│
│   • Changing a navigation property changes the relationship.
│
│   • EF Core synchronizes the corresponding foreign key.
│
│   🧠 Example:
│
│       Post
│        │
│        └── Blog = Old Blog
│
│             ↓
│
│       post.Blog = New Blog
│
│             ↓
│
│       EF Core updates:
│
│       post.BlogId
│
│
│   ⭐ Example:
│
│       var blog = new Blog
│       {
│           Url = "http://blogs.msdn.com/visualstudio"
│       };
│
│       var post = await context.Posts.FirstAsync();
│
│       post.Blog = blog;
│
│       await context.SaveChangesAsync();
│
│   🧠 WHAT EF CORE DOES:
│
│       New Blog
│           ↓
│       INSERT Blog
│           ↓
│       Database generates BlogId
│           ↓
│       Existing Post relationship changes
│           ↓
│       UPDATE Post.BlogId
│
│
├── REMOVING RELATIONSHIPS
│
│   A relationship can be removed by:
│
│   • Setting a reference navigation to null.
│
│       post.Blog = null;
│
│   • Removing an entity from a collection navigation.
│
│       blog.Posts.Remove(post);
│
│
├── REQUIRED vs OPTIONAL RELATIONSHIPS
│
│   ⭐ REQUIRED RELATIONSHIP
│
│       Blog
│         │
│         └── Post
│
│   • Dependent entity requires a principal.
│   • By default, cascade delete behavior is configured.
│
│   Remove Relationship
│          ↓
│   Dependent cannot exist without Principal
│          ↓
│   Dependent may be Deleted
│
│
│   ⭐ OPTIONAL RELATIONSHIP
│
│   • Dependent entity can exist without the principal.
│
│   Remove Relationship
│          ↓
│   Foreign Key
│          ↓
│   Set to NULL
│
│
├── REMOVING FROM A COLLECTION
│
│   Example:
│
│       var blog = await context.Blogs
│           .Include(b => b.Posts)
│           .FirstAsync();
│
│       var post = blog.Posts.First();
│
│       blog.Posts.Remove(post);
│
│       await context.SaveChangesAsync();
│
│   Result depends on relationship configuration:
│
│       Required Relationship
│               ↓
│       Cascade Delete
│               ↓
│       DELETE Dependent
│
│   OR
│
│       Optional Relationship
│               ↓
│       Relationship Removed
│               ↓
│       Foreign Key = NULL
│
│
├── ENTITY GRAPH + SAVECHANGES()
│
│       Entity Graph
│
│            Blog
│           /    \
│       Post    Post
│
│             ↓
│
│       Change Tracker
│
│             ↓
│
│       Detect Entity States
│
│       Added
│       Modified
│       Deleted
│
│             ↓
│
│       SaveChanges()
│
│             ↓
│
│       Correct SQL Operations
│
│
├── IMPORTANT RULE
│
│   ⭐ Navigation properties are not just convenient object references.
│
│   When entities are tracked, changing navigation properties can cause:
│
│       • Foreign Key Updates
│       • INSERT Operations
│       • UPDATE Operations
│       • DELETE Operations
│
│   → EF Core keeps relationships and foreign keys synchronized.
│
│
└── 🧠 30-SECOND MENTAL MODEL

    Entity Graph
    → A group of related entities connected through relationships.

    Add Root Entity
    → EF Core can discover and add reachable new entities.

    Add Related Entity
    → Add it through a tracked entity's navigation property.

    Change Navigation
    → EF Core updates the corresponding foreign key relationship.

    Remove Relationship
    → Required relationship may delete the dependent.
    → Optional relationship usually sets the foreign key to NULL.

    SaveChanges()
    → EF Core analyzes the entire tracked entity graph
      and generates the required INSERT / UPDATE / DELETE operations.

    Golden Rule
    → Change relationships through tracked entities,
      and EF Core keeps the database relationship in sync.
```