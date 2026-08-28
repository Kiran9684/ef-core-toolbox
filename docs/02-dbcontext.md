# 🧰 EF Core Interview Toolbox — Topic 2: DbContext Lifetime, Configuration, and Initialization

```text
EF CORE TOOLBOX
│
├── DBCONTEXT LIFETIME
│
│   • DbContext lifetime begins when the instance is created.
│   • It ends when the instance is disposed.
│   • DbContext is designed for a single Unit of Work.
│   • Therefore, a DbContext should usually be short-lived.
│
│   🧠 Mental Flow:
│
│       Create DbContext
│             ↓
│       Query / Track Entities
│             ↓
│       Apply Business Changes
│             ↓
│       SaveChanges()
│             ↓
│       Dispose DbContext
│
│   ⭐ IMPORTANT:
│   • Dispose DbContext after use.
│   • Releases resources and unregisters events/hooks.
│   • DbContext is NOT thread-safe.
│   • Do not share the same DbContext across concurrent threads.
│   • Always await EF Core async operations before using the context again.
│
│
├── UNIT OF WORK
│
│   • A Unit of Work represents one complete business operation.
│
│   Typical EF Core Unit of Work:
│
│       Create DbContext
│             ↓
│       Query Entities
│             ↓
│       Entities become Tracked
│             ↓
│       Modify / Add / Remove Entities
│             ↓
│       SaveChanges()
│             ↓
│       EF Core generates DB changes
│             ↓
│       Dispose DbContext
│
│   ⭐ Entities become tracked when:
│   • Returned from a tracking query.
│   • Added using Add().
│   • Attached using Attach().
│
│
├── DBCONTEXT + DEPENDENCY INJECTION
│
│   • In ASP.NET Core, one HTTP request usually represents one Unit of Work.
│   • AddDbContext() registers DbContext with the DI container.
│   • By default, DbContext is registered with a Scoped lifetime.
│
│   🧠 Mental Model:
│
│       HTTP Request
│            ↓
│       DI Scope
│            ↓
│       DbContext Created
│            ↓
│       Controller / Service
│            ↓
│       Database Operations
│            ↓
│       Request Ends
│            ↓
│       DbContext Disposed
│
│   ⭐ Configuration:
│
│       builder.Services.AddDbContext<ApplicationDbContext>(
│           options => options.UseSqlServer(connectionString));
│
│   DbContext constructor:
│
│       public class ApplicationDbContext : DbContext
│       {
│           public ApplicationDbContext(
│               DbContextOptions<ApplicationDbContext> options)
│               : base(options)
│           {
│           }
│       }
│
│
├── BASIC INITIALIZATION USING new
│
│   • DbContext can also be created manually using new.
│   • Configuration can be provided through:
│       1. OnConfiguring()
│       2. Constructor
│       3. DbContextOptionsBuilder
│
│   ⭐ OnConfiguring Example:
│
│       protected override void OnConfiguring(
│           DbContextOptionsBuilder optionsBuilder)
│       {
│           optionsBuilder.UseSqlServer(connectionString);
│       }
│
│   ⭐ Explicit Options Example:
│
│       var options =
│           new DbContextOptionsBuilder<ApplicationDbContext>()
│               .UseSqlServer(connectionString)
│               .Options;
│
│       using var context =
│           new ApplicationDbContext(options);
│
│
├── DBCONTEXT FACTORY
│
│   • Use AddDbContextFactory() when one DI scope does not match the
│     desired DbContext lifetime.
│   • Useful when multiple independent Units of Work are required.
│
│   🧠 Mental Model:
│
│       IDbContextFactory
│              │
│              ├── Create DbContext → Unit of Work 1
│              │
│              ├── Create DbContext → Unit of Work 2
│              │
│              └── Create DbContext → Unit of Work 3
│
│   ⭐ Registration:
│
│       services.AddDbContextFactory<ApplicationDbContext>(
│           options => options.UseSqlServer(connectionString));
│
│   ⭐ Usage:
│
│       public class MyService
│       {
│           private readonly
│               IDbContextFactory<ApplicationDbContext> _contextFactory;
│
│           public MyService(
│               IDbContextFactory<ApplicationDbContext> contextFactory)
│           {
│               _contextFactory = contextFactory;
│           }
│       }
│
│       using var context =
│           _contextFactory.CreateDbContext();
│
│   ⭐ IMPORTANT:
│   • Factory-created DbContext instances are NOT automatically disposed
│     by the application's DI scope.
│   • The application must dispose them.
│
│
├── DBCONTEXT OPTIONS & CONFIGURATION
│
│   • DbContextOptionsBuilder is the starting point for configuration.
│
│   It can be obtained from:
│
│       AddDbContext()
│             │
│             ├── OnConfiguring()
│             │
│             └── new DbContextOptionsBuilder()
│
│   • OnConfiguring() is called regardless of how the context is constructed.
│   • Additional configuration can be added through OnConfiguring().
│
│
├── DATABASE PROVIDER
│
│   • Every DbContext instance must use exactly ONE database provider.
│   • A provider is configured using a Use*() method.
│
│       DbContext
│           ↓
│       UseSqlServer()
│           ↓
│       SQL Server Provider
│
│   Common Providers:
│
│   • SQL Server      → UseSqlServer()
│   • SQLite          → UseSqlite()
│   • PostgreSQL      → UseNpgsql()
│   • MySQL/MariaDB   → UseMySql()
│   • Oracle          → UseOracle()
│   • Cosmos DB       → UseCosmos()
│   • In-Memory       → UseInMemoryDatabase()
│
│   ⭐ Provider packages provide the corresponding Use*() extension methods.
│
│
├── OTHER CONFIGURATION OPTIONS
│
│   • UseQueryTrackingBehavior()
│       → Sets default query tracking behavior.
│
│   • LogTo()
│       → Simple EF Core logging.
│
│   • UseLoggerFactory()
│       → Integrates Microsoft.Extensions.Logging.
│
│   • EnableSensitiveDataLogging()
│       → Includes application data in logs/errors.
│
│   • EnableDetailedErrors()
│       → Provides more detailed errors.
│
│   • ConfigureWarnings()
│       → Controls EF Core warnings.
│
│   • AddInterceptors()
│       → Registers EF Core interceptors.
│
│   • UseLazyLoadingProxies()
│       → Enables lazy loading proxies.
│
│
├── DBCONTEXTOPTIONS vs DBCONTEXTOPTIONS<TCONTEXT>
│
│   DbContextOptions<TContext>
│
│   • Preferred for most concrete DbContext classes.
│   • Ensures the correct options are resolved for the specific context type.
│
│       DbContextOptions<ApplicationDbContext>
│                    ↓
│            ApplicationDbContext
│
│   DbContextOptions
│
│   • Useful for base DbContext classes intended for inheritance.
│
│       DbContext
│           ↓
│       BaseContext
│        /       \
│       ↓         ↓
│   Context1   Context2
│
│   ⭐ Pattern:
│
│       public abstract class BaseContext : DbContext
│       {
│           protected BaseContext(
│               DbContextOptions options)
│               : base(options)
│           {
│           }
│       }
│
│
├── THREADING & ASYNC PITFALLS
│
│   ❌ Never do parallel operations on the same DbContext:
│
│           DbContext
│            /       \
│           ↓         ↓
│       Query A     Query B
│
│           ❌ Concurrent Access
│
│   ⭐ Correct:
│
│       await Query A
│             ↓
│       await Query B
│
│   OR:
│
│       Query A → DbContext A
│
│       Query B → DbContext B
│
│   ⭐ Golden Rule:
│
│       Parallel Operations
│               ↓
│       Separate DbContext Instances
│
│
└── 🧠 30-SECOND MENTAL MODEL

    DbContext
    → Represents a short-lived session / Unit of Work with the database.

    Typical ASP.NET Core Lifetime
    → One HTTP Request → One DI Scope → One DbContext.

    AddDbContext()
    → Registers DbContext as Scoped by default.

    DbContextOptionsBuilder
    → Used to configure the database provider and EF Core behavior.

    OnConfiguring()
    → Another place where DbContext configuration can be applied.

    IDbContextFactory
    → Creates independent DbContext instances when multiple Units of Work
      are needed.

    Thread Safety
    → DbContext is NOT thread-safe.

    Async Rule
    → Always await EF Core operations before reusing the same DbContext.

    Parallel Work
    → Use separate DbContext instances.
```
