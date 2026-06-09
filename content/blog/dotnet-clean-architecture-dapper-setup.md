+++
title = "Wiring Up Dapper and a Clean Program.cs in .NET Clean Architecture"
date = "2026-06-10"
description = "How to set up Dapper with PostgreSQL, a UnitOfWork pattern, DbUp migrations, and a clean Program.cs using extension methods in a .NET Clean Architecture solution."
tags = ["dotnet", "clean-architecture", "dapper", "backend"]
+++

In the [previous post](/blog/dotnet-clean-architecture-scaffolding/), we scaffolded a Clean Architecture solution from scratch. Now we wire it up: Dapper for data access, DbUp for migrations, a Unit of Work for transactions, and a `Program.cs` that stays readable no matter how large the app grows.

---

## The Problem with a Bloated Program.cs

The default `Program.cs` template is fine for small apps. As the app grows, it becomes a dumping ground — EF context registration, JWT config, Swagger, CORS, Coravel schedulers, custom middleware all piled into one file.

The fix: each layer owns its own registration, and `Program.cs` just calls them.

---

## Program.cs — Keep It to Three Lines

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.ConfigureServices().ConfigurePipeline();
app.Run();
```

With Serilog bootstrapped before the builder (so startup exceptions get logged too):

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
            .MinimumLevel.Information()
            .CreateLogger();
        try
        {
            var builder = WebApplication.CreateBuilder(args);
            var app = builder.ConfigureServices().ConfigurePipeline();
            app.Run();
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "Application start-up failed");
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }
}
```

`ConfigureServices` and `ConfigurePipeline` are extension methods on `WebApplicationBuilder` and `WebApplication` respectively, defined in a `StartupExtensions.cs` file in the API project.

---

## StartupExtensions.cs — Two Extension Methods

```csharp
public static class StartupExtensions
{
    public static WebApplication ConfigureServices(this WebApplicationBuilder builder)
    {
        // Run DB migrations before anything else
        DbUpMigrator.MigrateDatabase(builder.Configuration);

        // Each layer registers its own services
        builder.Services.AddDomainServices(builder.Configuration);
        builder.Services.AddApplicationServices();
        builder.Services.AddPersistenceServices(builder.Configuration);
        builder.Services.AddInfrastructureServices(builder.Configuration);

        builder.Services.AddHttpContextAccessor();
        builder.Services.AddControllers();
        builder.Services.AddMemoryCache();
        builder.Services.AddResponseCaching();
        builder.Services.AddDistributedMemoryCache();

        builder.Services.AddCors(options =>
        {
            options.AddPolicy("Open",
                b => b.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod());
        });

        builder.Services.AddEndpointsApiExplorer();
        builder.Services.AddSwaggerGen();

        return builder.Build();
    }

    public static WebApplication ConfigurePipeline(this WebApplication app)
    {
        app.UseSwagger();
        app.UseSwaggerUI();
        app.UseHttpsRedirection();
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseCustomExceptionHandler();
        app.UseCors("Open");
        app.UseResponseCaching();
        app.MapControllers();
        return app;
    }
}
```

`Program.cs` never changes. All growth happens inside the layer-specific `AddXxxServices` methods.

---

## Self-Registering Layers

Each layer exposes one extension method on `IServiceCollection`. The API just calls it — it doesn't know what's inside.

**Domain layer** (`DomainServiceRegistration.cs`):
```csharp
public static IServiceCollection AddDomainServices(
    this IServiceCollection services, IConfiguration configuration)
{
    // domain-level settings, value object configs, etc.
    return services;
}
```

**Application layer** (`ApplicationServiceRegistration.cs`):
```csharp
public static IServiceCollection AddApplicationServices(
    this IServiceCollection services)
{
    services.AddMediatR(cfg =>
        cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
    services.AddAutoMapper(Assembly.GetExecutingAssembly());
    services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
    return services;
}
```

---

## Dapper Setup in the Persistence Layer

SampleProject uses pure Dapper with PostgreSQL — no EF Core. The entire setup lives in `PersistenceServiceRegistration.cs`.

### Register IDbConnection as Scoped

```csharp
public static IServiceCollection AddPersistenceServices(
    this IServiceCollection services, IConfiguration configuration)
{
    var connectionString = configuration["ConnectionString:Postgres"];

    services.AddScoped<IDbConnection>(db =>
    {
        var connection = new NpgsqlConnection(connectionString);
        connection.Open();
        return connection;
    });

    SqlMapper.AddTypeHandler(new DapperDateOnlyTypeMapper());

    // Register repositories
    services.AddScoped<IAuthRepository, AuthRepository>();
    services.AddScoped<IUnitOfWork, UnitOfWork>();
    // ... more repositories

    return services;
}
```

`IDbConnection` is registered as `Scoped` — one connection per HTTP request. All repositories in the same request share the same connection, which makes transactions work correctly.

### Repositories Inject IDbConnection Directly

```csharp
public class AuthRepository : IAuthRepository
{
    private readonly IDbConnection _dbConnection;

    public AuthRepository(IDbConnection dbConnection)
    {
        _dbConnection = dbConnection;
        Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true;
    }

    public async Task<User?> GetByEmail(string email)
    {
        var sql = "SELECT * FROM users WHERE email = @Email";
        return await _dbConnection.QueryFirstOrDefaultAsync<User>(sql, new { Email = email });
    }

    public async Task<IReadOnlyList<TradeflowFeature>> GetFeaturesByUser(Guid userId)
    {
        var sql = @"
            SELECT tf.*, sf.*
            FROM feature_access fa
            INNER JOIN app_sub_features sf ON fa.sub_feature_id = sf.Id
            INNER JOIN app_features tf ON sf.feature_id = tf.Id
            WHERE fa.user_id = @UserId";

        var lookup = new Dictionary<Guid, TradeflowFeature>();
        await _dbConnection.QueryAsync<TradeflowFeature, TradeflowSubFeature, TradeflowFeature>(
            sql,
            (feature, subFeature) =>
            {
                if (!lookup.TryGetValue(feature.Id, out var current))
                {
                    current = feature;
                    current.SubFeatures = new List<TradeflowSubFeature>();
                    lookup.Add(current.Id, current);
                }
                current.SubFeatures.Add(subFeature);
                return current;
            },
            new { UserId = userId },
            splitOn: "Id"
        );
        return lookup.Values.ToList();
    }
}
```

`MatchNamesWithUnderscores = true` maps PostgreSQL snake_case columns (`user_id`, `created_at`) to C# PascalCase properties automatically.

---

## Custom Type Handler for DateOnly

Dapper doesn't know how to map PostgreSQL `date` to C#'s `DateOnly` out of the box. Fix with a type handler:

```csharp
public class DapperDateOnlyTypeMapper : SqlMapper.TypeHandler<DateOnly>
{
    public override DateOnly Parse(object value)
        => DateOnly.FromDateTime((DateTime)value);

    public override void SetValue(IDbDataParameter parameter, DateOnly value)
        => parameter.Value = value.ToDateTime(new TimeOnly());
}
```

Register it once in `PersistenceServiceRegistration`:

```csharp
SqlMapper.AddTypeHandler(new DapperDateOnlyTypeMapper());
```

---

## Unit of Work for Transactions

Because `IDbConnection` is scoped, all repositories in a request share the same connection. The Unit of Work wraps a transaction around that shared connection:

```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly IDbConnection _dbConnection;
    private IDbTransaction? _transaction;

    public UnitOfWork(IDbConnection dbConnection)
    {
        _dbConnection = dbConnection;
    }

    public void Begin()
    {
        if (_dbConnection.State != ConnectionState.Open)
            _dbConnection.Open();
        _transaction = _dbConnection.BeginTransaction();
    }

    public void Commit()
    {
        try { _transaction?.Commit(); }
        catch { _transaction?.Rollback(); throw; }
        finally { _transaction?.Dispose(); _transaction = null; }
    }

    public void Rollback()
    {
        _transaction?.Rollback();
        _transaction?.Dispose();
        _transaction = null;
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _dbConnection.Dispose();
    }
}
```

Usage in a handler:

```csharp
_unitOfWork.Begin();
await _billRepository.Insert(bill);
await _journalRepository.Insert(journal);
_unitOfWork.Commit();
```

If either insert fails, `Commit()` catches and rolls back. No partial writes.

---

## DbUp Migrations

Instead of EF Core migrations, SampleProject uses DbUp — a library that runs plain SQL scripts from a folder in order.

```csharp
public static class DbUpMigrator
{
    public static void MigrateDatabase(IConfiguration configuration)
    {
        var connectionString = configuration["ConnectionString:Postgres"];

        EnsureDatabase.For.PostgresqlDatabase(connectionString);

        var migrationsPath = Path.Combine(AppContext.BaseDirectory, "Migrations");
        var upgrader = DeployChanges.To
            .PostgresqlDatabase(connectionString)
            .LogToConsole()
            .WithScriptsFromFileSystem(migrationsPath)
            .Build();

        using var scope = new TransactionScope();
        var result = upgrader.PerformUpgrade();
        if (result.Successful)
        {
            scope.Complete();
        }
        else
        {
            Console.WriteLine("Migration failed: " + result.Error);
            Environment.Exit(1);
        }
    }
}
```

Called at the very start of `ConfigureServices`, before any DI registration runs. DbUp tracks which scripts have already run in a `schemaversions` table — safe to call on every startup.

SQL scripts go in `Persistence/Migrations/` as numbered files:

```
001_create_users.sql
002_create_accounts.sql
003_add_refresh_token_column.sql
```

---

## appsettings.json Connection String

```json
{
  "ConnectionString": {
    "Postgres": "Host=localhost;Database=sampledb;Username=postgres;Password=secret"
  }
}
```

---

## Summary

| Concern | Where it lives | How |
|---|---|---|
| DB connection | `PersistenceServiceRegistration` | `IDbConnection` scoped, opened per request |
| Queries | Repository classes | Dapper `QueryAsync` / `ExecuteAsync` |
| Transactions | `UnitOfWork` | Wraps the shared scoped `IDbConnection` |
| Type mapping | `DapperDateOnlyTypeMapper` | `SqlMapper.TypeHandler<T>` |
| Migrations | `DbUpMigrator` | DbUp, plain SQL files, runs at startup |
| Service registration | Each layer's `AddXxxServices` | Called once from `StartupExtensions` |
| `Program.cs` | Three lines | Never changes |

This pattern keeps every concern in the layer that owns it. Adding a new repository means touching only `PersistenceServiceRegistration` — `Program.cs` and `StartupExtensions` stay untouched.

---

**Next:** [Bulk Inserts in PostgreSQL with Npgsql Binary Import](/blog/dotnet-postgres-bulk-insert-binary-import/) — how to use `BeginBinaryImportAsync` for high-volume writes, JSONB columns, and the temp table pattern for bulk updates.
