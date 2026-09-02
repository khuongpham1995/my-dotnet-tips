# Clean Architecture + CQRS .NET Template Guide

A genericized reference distilled from a production multi-tenant .NET 10 Web API on **SQL Server**. Use this as an example to bootstrap a **brand-new** project with the same architecture — it contains structure, conventions, and file templates, not business logic. Every example below uses placeholder names (`YourApp`, `Product`) — replace with your real solution/company/domain names.

Hand this file to an agent (or a new teammate) with the instruction: *"Set up a new solution following this guide."*

---

## 1. Architectural style

**Clean Architecture + CQRS + Repository/Unit-of-Work**, strict inward dependency direction:

```
Domain  ←  Application  ←  Infrastructure[.Tenant]  ←  WebAPI
```

- `Domain` has **zero** project references — pure entities/enums/base types.
- `Application` references `Domain` only — commands/queries (MediatR), DTOs, validators, service *interfaces*. No EF Core, no HTTP.
- `Infrastructure` / `Infrastructure.Tenant` implement the interfaces declared in `Application` — EF Core `DbContext`, repositories, external HTTP clients, migrations.
- `WebAPI` is the composition root — controllers, middleware, DI wiring, Swagger.

Do not let a handler in `Application` new-up a `DbContext` or an `HttpClient` directly — everything crosses layers through an interface.

## 2. Solution layout

```
YourApp.sln
YourApp.Domain/
  Common/                      # BaseEntity<T>, AuditableEntity<T>, FullAuditableEntity<T>, marker interfaces
  Entities/                    # POCO domain entities
  GlobalUsings.cs
YourApp.Application/
  {Feature}/                   # one folder per bounded feature, e.g. Products/
    Commands/{CommandName}/
    Queries/{QueryName}/
    {Feature}Dto.cs            # DTOs shared across a feature's commands/queries
  Common/
    Interfaces/                # I{Entity}Repository.cs, IUnitOfWork.cs, service interfaces
    Services/                  # cross-cutting service interfaces (e.g. IDateTimeService)
    Exceptions/                # NotFoundException, ConflictDataException, ...
    Models/                    # ResponseModel<T>, PagedListResponse<T>, shared request/response shapes
    Abstractions/
      Behaviors/                # ValidationBehavior<,> (MediatR pipeline)
      Messaging/                 # ICommand<T>/IQuery<T> — see §8
    Mappings/                  # IMapFrom<T> + AutoMapper profile auto-registration
  IoCExtension.cs              # AddApplicationServices(): MediatR, AutoMapper, FluentValidation, pipeline behaviors
  GlobalUsings.cs
YourApp.Infrastructure/            # master / non-tenant DB (only needed if multi-tenant)
  Data/                       # master DbContext + initialiser/seed
  Migrations/
  IoCExtension.cs
YourApp.Infrastructure.Tenant/     # (or just "Infrastructure" in a single-tenant app)
  Data/
    ApplicationDbContext.cs
    Configurations/            # IEntityTypeConfiguration<T> per entity
    Repositories/
      BaseRepository.cs
      UnitOfWork.cs
      {Entity}Repository.cs
    Interceptors/               # SaveChanges interceptors (audit stamping, soft delete)
  Services/                    # implementations of Application service interfaces, external HTTP clients
                                # (DatabaseManager.cs / BulkManager.cs here too, if using §6b/§6c)
  Migrations/                  # EF Core migrations
  IoCExtension.cs              # AddInfrastructureServices(): DbContext, UoW, reflection-based DI registration
  GlobalUsings.cs
YourApp.WebAPI/
  Controllers/
    BaseController.cs
    {Feature}Controller.cs
  Middlewares/
    ExceptionHandlerMiddleware.cs
    TenantIdentifierMiddleware.cs   # only if multi-tenant
  Models/                     # ResponseModel<T>, ResponsePagedListModel<T> (if not in Application)
  Migrations/Scripts/          # raw .sql for stored procedures / views, copied to output via csproj
  Program.cs
  appsettings.json
  GlobalUsings.cs
YourApp.Tests/
  Unit/
  Integration/
```

A new end-to-end feature touches **Domain → Application → Infrastructure(.Tenant) → WebAPI**, in that order.

## 3. Naming & file conventions

- **File-scoped namespaces** (`namespace YourApp.Application.Products;`).
- **One public type per file**, filename matches the type name.
- **Folder structure mirrors namespace** — a command lives at `Application/{Feature}/Commands/{CommandName}/{CommandName}Command.cs`.
- `PascalCase` for public members/types, `camelCase` for locals/parameters.
- **`Async` suffix mandatory** on anything returning `Task`/`ValueTask`.
- DTOs end in `Dto` (`{Name}RequestDto`, `{Name}ResponseDto`).
- **Repository classes end in `Repository`, service classes end in `Service`** — keep this regardless of which DI registration approach (§7) you pick, for readability. Under §7a (reflection by name suffix) it's also functionally required — a class with no interface, or a name that doesn't match, silently doesn't get registered. Under §7b (Scrutor + marker interfaces) it's cosmetic; the marker interface is what actually drives registration.
- Async all the way: never `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` outside `Program.cs`.
- Propagate `CancellationToken` from controller → MediatR handler → repository call.

## 4. Domain layer — entity base classes

Three tiers, in `Domain/Common/`:

```csharp
public interface ISoftDelete
{
    bool IsDeleted { get; set; }
}

public abstract class BaseEntity<TKey>
{
    public TKey Id { get; set; }
}

public abstract class AuditableEntity<TKey> : BaseEntity<TKey>
{
    public string? CreatedBy { get; set; }
    public DateTime CreatedDate { get; set; }
    public string? LastModifiedBy { get; set; }
    public DateTime? LastModifiedDate { get; set; }
}

public abstract class FullAuditableEntity<TKey> : AuditableEntity<TKey>, ISoftDelete
{
    public bool IsDeleted { get; set; }
    public string? DeletedBy { get; set; }
    public DateTime? DeletedDate { get; set; }
}
```

Pick the tier by need: `BaseEntity<int>` for lookup/reference tables, `AuditableEntity<int>` for most business entities, `FullAuditableEntity<int>` where soft-delete matters. `TKey` is `int` unless you have a specific reason for `Guid`.

A `SaveChanges` interceptor (registered scoped in `Infrastructure(.Tenant)/IoCExtension.cs`) stamps `CreatedBy`/`CreatedDate`/`LastModifiedBy`/`LastModifiedDate` automatically — handlers never set these fields manually.

Example entity:

```csharp
public class Product : FullAuditableEntity<int>
{
    public string Code { get; set; }
    public string Name { get; set; }

    public int? CategoryId { get; set; }
    public virtual Category? Category { get; set; }

    public virtual ICollection<ProductVariant> Variants { get; set; } = new List<ProductVariant>();
}
```

Rules: collection navigations are `virtual ICollection<T>` initialised to `new List<T>()`; reference navigation nullability matches its FK's nullability.

## 5. EF Core configuration

One `IEntityTypeConfiguration<T>` file per entity, class name ends in `Configuration`, picked up via `modelBuilder.ApplyConfigurationsFromAssembly(...)` in `OnModelCreating`. Prefer fluent config over data annotations.

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Product");
        builder.HasKey(x => x.Id);

        builder.Property(x => x.Code).IsRequired().HasMaxLength(20);
        builder.Property(x => x.Name).IsRequired().HasMaxLength(200);
        builder.HasIndex(x => x.Code).IsUnique();

        builder.HasMany(x => x.Variants)
               .WithOne(x => x.Product)
               .HasForeignKey(x => x.ProductId)
               .OnDelete(DeleteBehavior.Cascade);
    }
}
```

## 6. Repository + Unit of Work

All EF Core access goes through repositories — **never inject `DbContext` directly into a handler.**

Per-entity interfaces never hand-declare CRUD signatures — they extend one shared `IRepository<T, TKey>`, so they can't drift out of sync with `BaseRepository<T, TKey>`'s actual method shapes.

```csharp
// Application/Common/Interfaces/IRepository.cs
public interface IRepository<T, TKey> where T : BaseEntity<TKey>
{
    Task<T?> GetByIdAsync(TKey id, CancellationToken cancellationToken = default, params Expression<Func<T, object>>[] includeProperties);
    Task<List<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default, params Expression<Func<T, object>>[] includeProperties);
    Task AddAsync(T entity, CancellationToken cancellationToken = default);
    void Update(T entity);
    void Remove(T entity);
    void SoftDelete(T entity);
}
```

```csharp
// Application/Common/Interfaces/IProductRepository.cs
public interface IProductRepository : IRepository<Product, int>
{
    // entity-specific methods beyond the shared CRUD contract go here, e.g.:
    // Task<Product?> GetByCodeAsync(string code, CancellationToken cancellationToken = default);
}
```

```csharp
// Infrastructure(.Tenant)/Data/Repositories/BaseRepository.cs
public abstract class BaseRepository<T, TKey>(DbContext context) : IRepository<T, TKey> where T : BaseEntity<TKey>
{
    protected readonly DbContext _context = context;
    protected readonly DbSet<T> _dbSet = context.Set<T>();

    public async Task<T?> GetByIdAsync(TKey id, CancellationToken cancellationToken = default,
        params Expression<Func<T, object>>[] includeProperties)
        => await Query(x => x.Id.Equals(id), includeProperties).FirstOrDefaultAsync(cancellationToken);

    public async Task<List<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default,
        params Expression<Func<T, object>>[] includeProperties)
        => await Query(predicate, includeProperties).ToListAsync(cancellationToken);

    public async Task AddAsync(T entity, CancellationToken cancellationToken = default)
        => await _dbSet.AddAsync(entity, cancellationToken);

    public void Update(T entity) => _dbSet.Update(entity);

    public void Remove(T entity) => _dbSet.Remove(entity);

    public void SoftDelete(T entity)
    {
        if (entity is ISoftDelete sd) { sd.IsDeleted = true; Update(entity); }
    }

    private IQueryable<T> Query(Expression<Func<T, bool>> predicate,
        params Expression<Func<T, object>>[] includeProperties)
    {
        IQueryable<T> query = _dbSet.AsNoTracking();
        if (typeof(ISoftDelete).IsAssignableFrom(typeof(T)))
            query = query.Where(x => !((ISoftDelete)(object)x).IsDeleted);
        query = query.Where(predicate);
        foreach (var include in includeProperties) query = query.Include(include);
        return query;
    }
}
```

> This is the minimal CRUD core, enough for ordinary command/query handlers. If the new project also needs a client-driven "advanced search" screen (a data-grid with dynamic filter/sort), paged stored-procedure-backed reports, or bulk import/update jobs, extend it with §6a–§6c below — otherwise skip straight to §7.

### 6a. Extended `BaseRepository` — dynamic advanced-search + stored-procedure support (optional)

This grows the core above with a **dynamic filter/sort/query-string builder** that turns a client-supplied filter payload into a string predicate, for a data-grid "advanced search" screen. It has two output modes: a [Dynamic LINQ](https://github.com/zzzprojects/System.Linq.Dynamic.Core) expression string fed straight into EF Core's `IQueryable.Where(string)`/`OrderBy(string)` (safe — Dynamic LINQ parses it as an expression, not raw SQL), and a raw SQL `WHERE`-fragment string for callers that hand a dynamic filter clause to a stored procedure (`…ForProc` methods) — see the security note below before using that second mode.

(Stored-procedure-backed reads that don't go through EF Core's LINQ provider at all live on `IDatabaseManager` instead — §6b — not here; see that section for why they're not repository methods.)

Add this only when a screen genuinely needs open-ended, client-driven filtering/sorting; for ordinary CRUD, §6's minimal core is enough — don't take on this complexity by default. Needs the `System.Linq.Dynamic.Core` package (§14) **and** a `global using System.Linq.Dynamic.Core;` in `Infrastructure(.Tenant)/GlobalUsings.cs` — without it, `query.Where(filterScript)`/`.OrderBy(orderByScript)` below silently bind to plain LINQ's `Where`/`OrderBy` overloads instead of Dynamic LINQ's string-based ones, and the compiler error you get (`cannot convert from 'string' to 'Func<T, bool>'`, or an unrelated-looking type-inference failure on `OrderBy`) doesn't point at a missing `using` at all.

```csharp
// Infrastructure(.Tenant)/Data/Repositories/BaseRepository.cs
public abstract class BaseRepository<T, TKey>(DbContext context) : IRepository<T, TKey> where T : BaseEntity<TKey>
{
    protected readonly DbContext _context = context;
    protected readonly DbSet<T> _dbSet = context.Set<T>();
    private readonly List<PropertyType> _tableProperties = typeof(T).GetProperties()
        .Select(p => new PropertyType { Name = p.Name, Type = p.PropertyType })
        .ToList();

    public Task<bool> AnyAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default)
        => GetQueryable(predicate).AnyAsync(cancellationToken);

    public void Remove(T entity) => _dbSet.Remove(entity);
    public void RemoveRange(ICollection<T> entities) => _dbSet.RemoveRange(entities);
    public async Task AddAsync(T entity, CancellationToken cancellationToken = default) => await _dbSet.AddAsync(entity, cancellationToken);
    public Task AddRangeAsync(ICollection<T> entities, CancellationToken cancellationToken = default) => _dbSet.AddRangeAsync(entities, cancellationToken);
    public void Update(T entity) => _dbSet.Update(entity);
    public void UpdateRange(ICollection<T> entities) => _dbSet.UpdateRange(entities);

    public void SoftDelete(T entity)
    {
        if (entity is ISoftDelete sd) { sd.IsDeleted = true; Update(entity); }
    }

    public Task<T?> GetByIdAsync(TKey id, CancellationToken cancellationToken = default, params Expression<Func<T, object>>[] includeProperties)
        => GetQueryable(x => x.Id.Equals(id), includeProperties).FirstOrDefaultAsync(cancellationToken);

    public Task<T?> GetSingleAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default, params Expression<Func<T, object>>[] includeProperties)
        => GetQueryable(predicate, includeProperties).FirstOrDefaultAsync(cancellationToken);

    public Task<List<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default, params Expression<Func<T, object>>[] includeProperties)
        => GetQueryable(predicate, includeProperties).ToListAsync(cancellationToken);

    public Task<List<T>> GetAllAsync(CancellationToken cancellationToken = default, params Expression<Func<T, object>>[] includeProperties)
        => GetQueryable(includeProperties).ToListAsync(cancellationToken);

    public IQueryable<T> GetQueryable(Expression<Func<T, bool>> predicate, params Expression<Func<T, object>>[] includeProperties)
    {
        var items = ApplySoftDeleteFilter(_dbSet.AsQueryable()).Where(predicate);
        foreach (var include in includeProperties ?? [])
            items = items.Include(include);
        return items;
    }

    public IQueryable<T> GetQueryable(params Expression<Func<T, object>>[] includeProperties)
    {
        var items = ApplySoftDeleteFilter(_dbSet.AsQueryable());
        foreach (var include in includeProperties ?? [])
            items = items.Include(include);
        return items;
    }

    private static IQueryable<T> ApplySoftDeleteFilter(IQueryable<T> query)
        => typeof(ISoftDelete).IsAssignableFrom(typeof(T))
            ? query.Where(x => !((ISoftDelete)(object)x).IsDeleted)
            : query;

    // Builds a dynamic, client-driven filter/sort query for an advanced-search grid.
    public IQueryable<T> SearchTable(AdvancedComplexSearchRequest searchPayload)
    {
        ArgumentNullException.ThrowIfNull(searchPayload);

        var query = ApplySoftDeleteFilter(_dbSet.AsQueryable()); // same soft-delete rule as GetQueryable

        var filterScript = DoFilter(searchPayload.Filter);
        if (!string.IsNullOrEmpty(filterScript)) query = query.Where(filterScript);

        var queryScript = DoQuery(searchPayload.Query);
        if (!string.IsNullOrEmpty(queryScript)) query = query.Where(queryScript);

        var orderByScript = DoOrderBy(searchPayload.OrderBy);
        if (!string.IsNullOrEmpty(orderByScript)) query = query.OrderBy(orderByScript);

        return query;
    }

    #region Dynamic LINQ filter/query/order-by builders (safe — parsed as an expression, not raw SQL)

    public string DoQuery(List<AdvancedSearchQueryItem> filters)
    {
        if (filters == null || filters.Count == 0) return string.Empty;

        var allConditions = string.Empty;
        foreach (var f in filters)
        {
            var condition = string.Join(" or ", f.SearchField.Select(field => GetFilterQueryString(field, f.SearchValue, f.Operator)));
            allConditions = string.IsNullOrEmpty(allConditions) ? $"({condition})" : $"{allConditions} and ({condition})";
        }
        return allConditions;
    }

    public string DoFilter(AdvancedComplexSearchFilterModel filter)
    {
        if (filter?.Filters == null) return string.Empty;

        var logic = string.IsNullOrEmpty(filter.Logic) ? "and" : filter.Logic;
        var allConditions = string.Empty;

        foreach (var f in filter.Filters)
        {
            if (f is not AdvancedSearchFilterItem fi)
                throw new InvalidDataException("Invalid filter type");

            var values = fi.SearchValue is { Length: > 0 } ? fi.SearchValue : [null];
            var condition = string.Join(" or ", values.Select(v => GetFilterQueryString(fi.SearchField, v, fi.Operator, fi.IncludeNull)));

            allConditions = string.IsNullOrEmpty(allConditions) ? $"({condition})" : $"{allConditions} {logic} ({condition})";
        }
        return allConditions;
    }

    public string DoOrderBy(string[] orderBys)
    {
        if (orderBys == null) return string.Empty;

        return string.Join(",", orderBys.Select(o =>
        {
            var parts = o.Split('|');
            var isDescending = parts.Length > 1 && parts[1].StartsWith("desc", StringComparison.OrdinalIgnoreCase);
            return isDescending ? $"{parts[0]} desc" : parts[0];
        }));
    }

    private string GetFilterQueryString(string field, string value, string op, bool? includeNull = null)
    {
        if (string.IsNullOrWhiteSpace(field)) throw new ArgumentException("Filter/query field is required");

        var prop = _tableProperties.FirstOrDefault(p => p.Name.Equals(field, StringComparison.OrdinalIgnoreCase))
            ?? throw new InvalidDataException($"Invalid field name '{field}' in filter/query.");

        var valueStr = GetDynamicLinqValueString(prop.Type, value);
        var result = op.ToUpperInvariant() switch
        {
            nameof(OperatorEnum.EQ) => $"{prop.Name} = {valueStr}",
            nameof(OperatorEnum.NEQ) => $"{prop.Name} != {valueStr}",
            nameof(OperatorEnum.LT) => $"{prop.Name} < {valueStr}",
            nameof(OperatorEnum.GT) => $"{prop.Name} > {valueStr}",
            nameof(OperatorEnum.LTE) => $"{prop.Name} <= {valueStr}",
            nameof(OperatorEnum.GTE) => $"{prop.Name} >= {valueStr}",
            nameof(OperatorEnum.CT) => $"{prop.Name}.Contains({valueStr})",
            nameof(OperatorEnum.NCT) => $"!{prop.Name}.Contains({valueStr})",
            nameof(OperatorEnum.BW) => $"{prop.Name}.StartsWith({valueStr})",
            nameof(OperatorEnum.NBW) => $"!{prop.Name}.StartsWith({valueStr})",
            nameof(OperatorEnum.EW) => $"{prop.Name}.EndsWith({valueStr})",
            nameof(OperatorEnum.NEW) => $"!{prop.Name}.EndsWith({valueStr})",
            nameof(OperatorEnum.NULL) => $"{prop.Name} = null",
            nameof(OperatorEnum.NNULL) => $"{prop.Name} != null",
            _ => throw new InvalidDataException($"Invalid operator {op}")
        };

        return includeNull == true ? $"{result} or {prop.Name} = null" : result;
    }

    private static string GetDynamicLinqValueString(Type type, string value)
    {
        var escaped = value?.Replace("\\", "\\\\").Replace("\"", "\\\"");
        var isQuotedType = type == typeof(string) || type == typeof(DateTime)
            || Nullable.GetUnderlyingType(type) is { } u && (u == typeof(string) || u == typeof(DateTime));
        return isQuotedType ? $"\"{escaped}\"" : escaped;
    }

    #endregion

    #region Raw-SQL filter builders for stored procedures — see security note below

    // Same shape as DoFilter but emits a literal SQL WHERE-clause fragment for a stored procedure that
    // accepts a dynamic filter parameter, instead of a Dynamic-LINQ expression string. searchKeys maps
    // each client-facing field name to its real SQL column — the field allow-list; only names present
    // in it are ever emitted into the fragment.
    public string DoFilterForProc(AdvancedComplexSearchFilterModel filter, List<AdvancedSearchKey> searchKeys)
    {
        if (filter?.Filters == null || searchKeys is not { Count: > 0 }) return string.Empty;

        var logic = string.IsNullOrEmpty(filter.Logic) ? "and" : filter.Logic;
        var allConditions = string.Empty;

        foreach (var f in filter.Filters)
        {
            if (f is not AdvancedSearchFilterItem fi)
                throw new InvalidDataException("Invalid filter type");

            var key = searchKeys.FirstOrDefault(k => k.Key.Equals(fi.SearchField, StringComparison.OrdinalIgnoreCase))
                ?? throw new InvalidDataException($"Field '{fi.SearchField}' does not exist");

            var values = fi.SearchValue is { Length: > 0 } ? fi.SearchValue : [null];
            var condition = string.Join(" or ", values.Select(v => GetFilterQueryStringForProc(key.Value, v, fi.Operator, fi.IncludeNull)));

            allConditions = string.IsNullOrEmpty(allConditions) ? $"({condition})" : $"{allConditions} {logic} ({condition})";
        }
        return allConditions;
    }

    private string GetFilterQueryStringForProc(string column, string value, string op, bool? includeNull = null)
    {
        if (op is nameof(OperatorEnum.BW) or nameof(OperatorEnum.NBW)) value += '%';
        if (op is nameof(OperatorEnum.EW) or nameof(OperatorEnum.NEW)) value = '%' + value;

        var valueStr = GetSqlLiteralString(value);
        var result = op.ToUpperInvariant() switch
        {
            nameof(OperatorEnum.EQ) => $"{column} = {valueStr}",
            nameof(OperatorEnum.NEQ) => $"{column} != {valueStr}",
            nameof(OperatorEnum.LT) => $"{column} < {valueStr}",
            nameof(OperatorEnum.GT) => $"{column} > {valueStr}",
            nameof(OperatorEnum.LTE) => $"{column} <= {valueStr}",
            nameof(OperatorEnum.GTE) => $"{column} >= {valueStr}",
            nameof(OperatorEnum.CT) => $"{column} LIKE '%{value}%'",
            nameof(OperatorEnum.NCT) => $"{column} NOT LIKE '%{value}%'",
            nameof(OperatorEnum.BW) or nameof(OperatorEnum.EW) => $"{column} LIKE {valueStr}",
            nameof(OperatorEnum.NBW) or nameof(OperatorEnum.NEW) => $"{column} NOT LIKE {valueStr}",
            nameof(OperatorEnum.NULL) => $"{column} IS NULL",
            nameof(OperatorEnum.NNULL) => $"{column} IS NOT NULL",
            _ => throw new InvalidDataException($"Invalid operator {op}")
        };

        return includeNull == true ? $"{result} or {column} = null" : result;
    }

    // Escapes a value for a single-quoted T-SQL literal. Doubling embedded ' is what keeps it safe
    // once the fragment reaches a stored procedure that runs it as dynamic SQL.
    private static string GetSqlLiteralString(string value) => $"'{value?.Replace("'", "''")}'";

    #endregion
}

internal class PropertyType
{
    public string Name { get; set; }
    public Type Type { get; set; }
}
```

```csharp
// Application/Common/Models/AdvancedSearch.cs — request/DTO shapes the builders above expect
public enum OperatorEnum { EQ, NEQ, LT, GT, LTE, GTE, CT, NCT, BW, NBW, EW, NEW, NULL, NNULL, IN }

public class AdvancedSearchQueryItem
{
    public string[] SearchField { get; set; }
    public string SearchValue { get; set; }
    public string Operator { get; set; }
}

public interface IAdvancedSearchFilter
{
    string SearchField { get; set; }
    string Operator { get; set; }
}

public class AdvancedSearchFilterItem : IAdvancedSearchFilter
{
    public string SearchField { get; set; }
    public string Operator { get; set; }
    public string[] SearchValue { get; set; }
    public bool? IncludeNull { get; set; }
}

public class AdvancedComplexSearchFilterModel
{
    public string Logic { get; set; }                          // "and" / "or"
    public List<IAdvancedSearchFilter> Filters { get; set; }
}

public class AdvancedComplexSearchRequest
{
    public AdvancedComplexSearchFilterModel Filter { get; set; }
    public List<AdvancedSearchQueryItem> Query { get; set; }
    public string[] OrderBy { get; set; }
}

// Maps a client-facing field name to the real SQL column/parameter it's allowed to touch —
// the allow-list the *ForProc builders check every field against before emitting SQL.
public class AdvancedSearchKey
{
    public string Key { get; set; }
    public string Value { get; set; }
}
```

⚠️ **Security note on the `…ForProc` builders:** unlike `DoFilter`/`DoQuery` (parsed by Dynamic LINQ as an expression, never concatenated into SQL), `DoFilterForProc` and `GetFilterQueryStringForProc` build a literal SQL text fragment meant to be handed to a stored procedure that runs it as part of a dynamic `WHERE` clause. That pattern is inherently injection-prone — only use it when the receiving procedure genuinely needs a free-form clause and there's no way to express the same thing as bound SP parameters instead. If you do use it:
- Field names must always go through the `searchKeys` allow-list (as above) — never let a client-supplied field name reach the SQL text directly.
- Values must go through `GetSqlLiteralString` (or equivalent) so embedded `'` can't break out of the literal — this is the bug fixed above; verify it before adopting this pattern from any other source.
- Prefer the Dynamic-LINQ path (`DoFilter`/`DoQuery` against `SearchTable`) by default; reach for the SP path only for genuine reporting/perf cases EF Core's LINQ provider can't handle well, and pin a recent `System.Linq.Dynamic.Core` version (it has had past CVEs around unrestricted type resolution).

### 6b. `DatabaseManager` — a reusable Dapper wrapper

Clean split, no overlap with §6a: `BaseRepository` owns EF-Core-typed access to one entity; `DatabaseManager` owns everything raw SQL / stored-procedure / not tied to a single entity type. A handler needing a paged SP-backed report or schema introspection injects `IDatabaseManager` directly — not through `IUnitOfWork`. `BaseRepository` carries no Dapper methods at all.

```csharp
// Application/Common/Interfaces/IDatabaseManager.cs
public interface IDatabaseManager
{
    Task<IEnumerable<T>> QueryAsync<T>(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default);

    Task<T> QueryFirstOrDefaultAsync<T>(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default);

    Task<T> QuerySingleAsync<T>(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default);

    Task<int> ExecuteAsync(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default);

    Task<T> ExecuteScalarAsync<T>(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default);

    // For a multi-result-set query. map must read every grid it needs — the connection closes as soon
    // as map returns, so a GridReader can't be handed back to the caller to read later.
    Task<TResult> QueryMultipleAsync<TResult>(string sql, Func<SqlMapper.GridReader, Task<TResult>> map,
        object parameters = null, CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default);

    Task<PagedListResponse<T>> QueryPagedAsync<T>(SqlSelectBuilder query, object parameters = null, CancellationToken cancellationToken = default);

    // Calls a paged stored procedure that takes @Page/@PageSize and returns one result set of rows.
    Task<IEnumerable<T>> GetPagedFromStoredProcedureAsync<T>(string procedureName, int page, int pageSize,
        object additionalParameters = null, CancellationToken cancellationToken = default);

    // Same as GetPagedFromStoredProcedureAsync, for a procedure that returns two result sets: the page of
    // rows, then an optional single-row "totals" grid (e.g. column sums) — omitted when no aggregate
    // columns were requested.
    Task<(IEnumerable<T> Rows, IDictionary<string, object> Totals)> GetPagedWithTotalsFromStoredProcedureAsync<T>(
        string procedureName, int page, int pageSize, object additionalParameters = null, CancellationToken cancellationToken = default);

    // Schema-drift guard for a reporting view.
    Task<IReadOnlyCollection<string>> GetViewColumnNamesAsync(string viewName, CancellationToken cancellationToken = default);
}
```

```csharp
// Infrastructure(.Tenant)/Services/DatabaseManager.cs
// Plain connection string, no EF Core dependency — the DI registration below resolves the tenant-correct
// one (§12). Deliberately NOT also IScoped — see the DI registration note below.
public class DatabaseManager(string connectionString, int? commandTimeout = null) : IDatabaseManager
{
    private async Task<T> WithConnectionAsync<T>(Func<IDbConnection, Task<T>> action, CancellationToken cancellationToken)
    {
        using var connection = new SqlConnection(connectionString);
        await connection.OpenAsync(cancellationToken);
        return await action(connection);
    }

    // Writes go through this instead of WithConnectionAsync: on failure the transaction rolls back and
    // the original exception propagates (§10).
    private Task<T> WithTransactionAsync<T>(Func<IDbConnection, IDbTransaction, Task<T>> action, CancellationToken cancellationToken)
        => WithConnectionAsync(async connection =>
        {
            using var transaction = connection.BeginTransaction(IsolationLevel.ReadCommitted);
            try
            {
                var result = await action(connection, transaction);
                transaction.Commit();
                return result;
            }
            catch
            {
                transaction.Rollback();
                throw;
            }
        }, cancellationToken);

    public Task<IEnumerable<T>> QueryAsync<T>(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default)
        => WithConnectionAsync(connection => connection.QueryAsync<T>(
            new CommandDefinition(sql, parameters, commandType: commandType, commandTimeout: commandTimeout, cancellationToken: cancellationToken)), cancellationToken);

    public Task<T> QueryFirstOrDefaultAsync<T>(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default)
        => WithConnectionAsync(connection => connection.QueryFirstOrDefaultAsync<T>(
            new CommandDefinition(sql, parameters, commandType: commandType, commandTimeout: commandTimeout, cancellationToken: cancellationToken)), cancellationToken);

    public Task<T> QuerySingleAsync<T>(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default)
        => WithConnectionAsync(connection => connection.QuerySingleAsync<T>(
            new CommandDefinition(sql, parameters, commandType: commandType, commandTimeout: commandTimeout, cancellationToken: cancellationToken)), cancellationToken);

    public Task<T> ExecuteScalarAsync<T>(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default)
        => WithConnectionAsync(connection => connection.ExecuteScalarAsync<T>(
            new CommandDefinition(sql, parameters, commandType: commandType, commandTimeout: commandTimeout, cancellationToken: cancellationToken)), cancellationToken);

    public Task<int> ExecuteAsync(string sql, object parameters = null,
        CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default)
        => WithTransactionAsync((connection, transaction) => connection.ExecuteAsync(
            new CommandDefinition(sql, parameters, transaction, commandType: commandType, commandTimeout: commandTimeout, cancellationToken: cancellationToken)), cancellationToken);

    public Task<TResult> QueryMultipleAsync<TResult>(string sql, Func<SqlMapper.GridReader, Task<TResult>> map,
        object parameters = null, CommandType commandType = CommandType.Text, CancellationToken cancellationToken = default)
        => WithConnectionAsync(async connection =>
        {
            using var grids = await connection.QueryMultipleAsync(
                new CommandDefinition(sql, parameters, commandType: commandType, commandTimeout: commandTimeout, cancellationToken: cancellationToken));
            return await map(grids);
        }, cancellationToken);

    // Runs query's count SQL and, if there's at least one row, its paged SQL — skips the second
    // round-trip on an empty result. Reuses §11's PagedListResponse<T> instead of a new shape.
    public async Task<PagedListResponse<T>> QueryPagedAsync<T>(SqlSelectBuilder query, object parameters = null, CancellationToken cancellationToken = default)
        => await WithConnectionAsync(async connection =>
        {
            var total = await connection.ExecuteScalarAsync<int>(new CommandDefinition(query.GetCountSql(), parameters, commandTimeout: commandTimeout, cancellationToken: cancellationToken));
            var result = new PagedListResponse<T> { Page = query.PageIndex, PageSize = query.PageSize, TotalRecords = total, TotalPages = (int)Math.Ceiling(total / (double)query.PageSize) };
            if (total == 0) return result;

            result.Data = (await connection.QueryAsync<T>(new CommandDefinition(query.GetPagedSql(), parameters, commandTimeout: commandTimeout, cancellationToken: cancellationToken))).ToList();
            return result;
        }, cancellationToken);

    public Task<IEnumerable<T>> GetPagedFromStoredProcedureAsync<T>(string procedureName, int page, int pageSize,
        object additionalParameters = null, CancellationToken cancellationToken = default)
    {
        var parameters = new DynamicParameters();
        parameters.Add("@Page", page);
        parameters.Add("@PageSize", pageSize);
        if (additionalParameters != null) parameters.AddDynamicParams(additionalParameters);

        return QueryAsync<T>(procedureName, parameters, CommandType.StoredProcedure, cancellationToken);
    }

    public Task<(IEnumerable<T> Rows, IDictionary<string, object> Totals)> GetPagedWithTotalsFromStoredProcedureAsync<T>(
        string procedureName, int page, int pageSize, object additionalParameters = null, CancellationToken cancellationToken = default)
    {
        var parameters = new DynamicParameters();
        parameters.Add("@Page", page);
        parameters.Add("@PageSize", pageSize);
        if (additionalParameters != null) parameters.AddDynamicParams(additionalParameters);

        return QueryMultipleAsync(procedureName, async grids =>
        {
            var rows = (await grids.ReadAsync<T>()).ToList();
            IDictionary<string, object> totals = grids.IsConsumed ? null : (await grids.ReadAsync()).FirstOrDefault();
            return ((IEnumerable<T>)rows, totals);
        }, parameters, CommandType.StoredProcedure, cancellationToken);
    }

    public async Task<IReadOnlyCollection<string>> GetViewColumnNamesAsync(string viewName, CancellationToken cancellationToken = default)
    {
        var names = await QueryAsync<string>(
            "SELECT c.name FROM sys.columns c WHERE c.object_id = OBJECT_ID(@viewName)",
            new { viewName }, cancellationToken: cancellationToken);
        return names.ToList();
    }
}
```

```csharp
// Application/Common/Models/SqlSelectBuilder.cs — assembles SELECT/COUNT/paged SQL from parts, ROW_NUMBER()-paginated
public class SqlSelectBuilder
{
    private string _with, _select, _from, _where, _groupBy, _having;
    private string _orderColumn, _orderDirection = "ASC";
    private string _thenOrderColumn, _thenOrderDirection = "ASC";

    public int PageIndex { get; private set; } = 1;
    public int PageSize { get; private set; } = 10;

    public SqlSelectBuilder AddWith(string with) { _with = with; return this; }
    public SqlSelectBuilder AddSelect(string selectColumns) { _select = selectColumns; return this; }
    public SqlSelectBuilder AddFrom(string from) { _from = from; return this; }
    public SqlSelectBuilder AddWhere(string where) { _where = where; return this; }
    public SqlSelectBuilder AddGroupBy(string groupByColumn) { _groupBy = groupByColumn; return this; }
    public SqlSelectBuilder AddHaving(string havingColumn) { _having = havingColumn; return this; }
    public SqlSelectBuilder AddPage(int pageIndex, int pageSize) { PageIndex = pageIndex; PageSize = pageSize; return this; }

    public SqlSelectBuilder AddOrderBy(string orderColumn, string orderDirection = "ASC")
    {
        _orderColumn = orderColumn;
        _orderDirection = orderDirection;
        return this;
    }

    public SqlSelectBuilder AddThenBy(string orderColumn, string orderDirection = "ASC")
    {
        _thenOrderColumn = orderColumn;
        _thenOrderDirection = orderDirection;
        return this;
    }

    // Counts rows the paged query would return — wraps the same WHERE/GROUP BY/HAVING as a CTE, so a
    // grouped query is counted by group, not by raw row.
    public string GetCountSql()
    {
        var sql = new StringBuilder();
        if (!string.IsNullOrWhiteSpace(_with)) sql.Append("WITH ").Append(_with).AppendLine(",");
        else sql.Append("WITH ");

        sql.AppendLine("countResult (TotalCount) AS (")
           .AppendLine("SELECT 0")
           .Append("FROM ").AppendLine(_from);
        if (!string.IsNullOrEmpty(_where)) sql.Append("WHERE ").AppendLine(_where);
        if (!string.IsNullOrEmpty(_groupBy)) sql.Append("GROUP BY ").AppendLine(_groupBy);
        if (!string.IsNullOrEmpty(_having)) sql.Append("HAVING ").AppendLine(_having);
        sql.Append(") SELECT COUNT(*) FROM countResult");

        return sql.ToString();
    }

    // ROW_NUMBER() requires a non-empty ORDER BY — call AddOrderBy() before this, or SQL Server rejects
    // the statement outright rather than paging in some arbitrary order.
    public string GetPagedSql()
    {
        if (string.IsNullOrEmpty(_orderColumn))
            throw new InvalidOperationException($"{nameof(SqlSelectBuilder)}.{nameof(GetPagedSql)} requires {nameof(AddOrderBy)} — ROW_NUMBER() has no default order.");

        var orderBy = $"{_orderColumn} {_orderDirection}";
        if (!string.IsNullOrEmpty(_thenOrderColumn)) orderBy += $", {_thenOrderColumn} {_thenOrderDirection}";

        var sql = new StringBuilder();
        if (!string.IsNullOrWhiteSpace(_with)) sql.Append("WITH ").AppendLine(_with);

        sql.Append("SELECT * FROM (")
           .AppendLine()
           .Append("SELECT ROW_NUMBER() OVER (ORDER BY ").Append(orderBy).AppendLine(") AS RN,")
           .AppendLine(_select)
           .Append("FROM ").AppendLine(_from);
        if (!string.IsNullOrEmpty(_where)) sql.Append("WHERE (").Append(_where).AppendLine(")");
        if (!string.IsNullOrEmpty(_groupBy)) sql.Append("GROUP BY ").AppendLine(_groupBy);
        if (!string.IsNullOrEmpty(_having)) sql.Append("HAVING ").AppendLine(_having);

        var firstRow = (PageIndex - 1) * PageSize + 1;
        sql.AppendLine(") [list]")
           .Append("WHERE RN BETWEEN ").Append(firstRow).Append(" AND ").AppendLine((firstRow + PageSize - 1).ToString())
           .Append("ORDER BY RN"); // required — the inner ROW_NUMBER() alone doesn't order the outer SELECT

        return sql.ToString();
    }
}
```

⚠️ Every part (`_select`/`_from`/`_where`/`_groupBy`/`_having`/`_orderColumn`/`_orderDirection`) is concatenated into SQL verbatim — T-SQL can't parameterize identifiers. Only pass values from trusted application code, or field names already validated against an allow-list like §6a's `searchKeys`.

DI registration — resolve the tenant-correct connection string once, at the point `DatabaseManager` is constructed, rather than giving the class an `IConfiguration`/`DbContext` dependency of its own:

```csharp
// Infrastructure(.Tenant)/IoCExtension.cs
services.AddScoped<IDatabaseManager>(sp =>
    new DatabaseManager(sp.GetRequiredService<ApplicationDbContext>().Database.GetConnectionString()));
```

Two things to know:

- **Register it manually.** Its constructor takes a plain `string`, so neither §7a's suffix scan nor §7b's marker-interface scan can construct it — don't add `IScoped` to the class either (Scrutor would try and fail, or collide with the manual registration).
- **Doesn't share a transaction with `IUnitOfWork`.** `ExecuteAsync` commits/rolls back on its own `SqlConnection`, separate from `ApplicationDbContext`'s. An EF Core write and a `DatabaseManager` write in the same handler aren't atomic — fine for read-only/reporting use; for an atomic mix, share `ApplicationDbContext`'s connection/transaction with Dapper explicitly instead of opening a second connection.

### 6c. `BulkManager` — bulk insert/update via `SqlBulkCopy` (optional)

For loading/updating thousands of rows at once — an import job, a batch reconciliation — row-by-row EF Core `SaveChangesAsync` or even a looped Dapper `ExecuteAsync` is too slow. This streams a `DataTable` into a `#temp` table with `SqlBulkCopy`, then `MERGE`s it into the target table in one statement, all inside one transaction.

⚠️ `tableName`, `keyColumnName`, and every column name in `dataTable` are interpolated straight into SQL (`CREATE TABLE #{tableName}`, `MERGE INTO {tableName}`, `[{columnName}]`) — T-SQL can't parameterize identifiers. Supply them from trusted application code only (hardcoded per import job, or config you control), never from a request.

```csharp
// Application/Common/Interfaces/IBulkManager.cs
public interface IBulkManager
{
    Task BulkInsertAsync(DataTable dataTable, string tableName, string keyColumnName, CancellationToken cancellationToken = default);
    Task BulkUpdateAsync(DataTable dataTable, string tableName, string keyColumnName, CancellationToken cancellationToken = default);
}
```

```csharp
// Infrastructure(.Tenant)/Services/BulkManager.cs
public class BulkManager(string connectionString, int? bulkCopyTimeout = null) : IBulkManager
{
    public Task BulkInsertAsync(DataTable dataTable, string tableName, string keyColumnName, CancellationToken cancellationToken = default)
        => UpsertAsync(dataTable, tableName, BuildInsertMergeSql(dataTable, tableName, keyColumnName), cancellationToken);

    public Task BulkUpdateAsync(DataTable dataTable, string tableName, string keyColumnName, CancellationToken cancellationToken = default)
        => UpsertAsync(dataTable, tableName, BuildUpdateMergeSql(dataTable, tableName, keyColumnName), cancellationToken);

    // Shared flow: stage into a #temp table via SqlBulkCopy, run the caller's MERGE into the real
    // table, drop the temp table — one transaction. On failure, roll back and let it propagate (§10).
    private async Task UpsertAsync(DataTable dataTable, string tableName, string mergeSql, CancellationToken cancellationToken)
    {
        using var connection = new SqlConnection(connectionString);
        await connection.OpenAsync(cancellationToken);
        using var transaction = connection.BeginTransaction(IsolationLevel.ReadCommitted);
        try
        {
            var tempTableSql = BuildCreateTempTableSql(dataTable, tableName);
            await connection.ExecuteAsync(new CommandDefinition(tempTableSql, transaction: transaction, cancellationToken: cancellationToken));

            using (var bulk = new SqlBulkCopy(connection, SqlBulkCopyOptions.Default, transaction))
            {
                bulk.DestinationTableName = $"#{tableName}";
                bulk.BulkCopyTimeout = bulkCopyTimeout ?? bulk.BulkCopyTimeout;
                bulk.BatchSize = Math.Min(dataTable.Rows.Count, 5000); // capped — one unbounded batch on a huge import is hard to cancel
                await bulk.WriteToServerAsync(dataTable, cancellationToken);
            }

            await connection.ExecuteAsync(new CommandDefinition(mergeSql, transaction: transaction, cancellationToken: cancellationToken));
            await connection.ExecuteAsync(new CommandDefinition($"DROP TABLE #{tableName}", transaction: transaction, cancellationToken: cancellationToken));

            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }

    private static string BuildCreateTempTableSql(DataTable dataTable, string tableName)
    {
        var columns = dataTable.Columns.Cast<DataColumn>()
            .Select(col => $"[{col.ColumnName}] {GetSqlDataType(col.DataType, col.AllowDBNull)}");
        return $"CREATE TABLE #{tableName} ({string.Join(", ", columns)});";
    }

    private static string BuildInsertMergeSql(DataTable dataTable, string tableName, string keyColumnName)
    {
        var columns = dataTable.Columns.Cast<DataColumn>().Select(c => c.ColumnName).ToArray();
        var insertColumns = string.Join(", ", columns.Select(c => $"[{c}]"));
        var sourceColumns = string.Join(", ", columns.Select(c => $"source.[{c}]"));

        return $"""
            MERGE INTO {tableName} AS target
            USING #{tableName} AS source ON target.{keyColumnName} = source.{keyColumnName}
            WHEN NOT MATCHED THEN INSERT ({insertColumns}) VALUES ({sourceColumns});
            """;
    }

    private static string BuildUpdateMergeSql(DataTable dataTable, string tableName, string keyColumnName)
    {
        var setClause = string.Join(", ", dataTable.Columns.Cast<DataColumn>()
            .Select(c => c.ColumnName).Where(c => c != keyColumnName)
            .Select(c => $"target.[{c}] = source.[{c}]"));

        return $"""
            MERGE INTO {tableName} AS target
            USING #{tableName} AS source ON target.{keyColumnName} = source.{keyColumnName}
            WHEN MATCHED THEN UPDATE SET {setClause};
            """;
    }

    private static string GetSqlDataType(Type dataType, bool isNullable) => Type.GetTypeCode(dataType) switch
    {
        TypeCode.Int32 => isNullable ? "INT NULL" : "INT NOT NULL",
        TypeCode.DateTime => isNullable ? "DATETIME NULL" : "DATETIME NOT NULL",
        TypeCode.Decimal => isNullable ? "DECIMAL(18, 2) NULL" : "DECIMAL(18, 2) NOT NULL",
        TypeCode.Boolean => isNullable ? "BIT NULL" : "BIT NOT NULL",
        TypeCode.Single or TypeCode.Double => isNullable ? "FLOAT NULL" : "FLOAT NOT NULL",
        _ when dataType == typeof(Guid) => "UNIQUEIDENTIFIER NULL",
        _ when dataType == typeof(DateTimeOffset) => "DATETIMEOFFSET NULL",
        _ => "NVARCHAR(MAX) NULL"
    };
}
```

DI registration follows the same factory pattern as §6b, for the same reason (a plain `string` constructor argument):

```csharp
services.AddScoped<IBulkManager>(sp =>
    new BulkManager(sp.GetRequiredService<ApplicationDbContext>().Database.GetConnectionString()));
```

```csharp
// Application/Common/Interfaces/IUnitOfWork.cs
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    ICategoryRepository Categories { get; }
    // ... one property per repository — add here whenever a repository is added

    Task<bool> SaveChangesAsync(CancellationToken cancellationToken = default);
    IDbContextTransaction BeginTransaction();
}
```

```csharp
// Infrastructure(.Tenant)/Data/Repositories/UnitOfWork.cs
public class UnitOfWork(ApplicationDbContext context, IProductRepository products, ICategoryRepository categories) : IUnitOfWork
{
    private readonly ApplicationDbContext _context = context;

    public IProductRepository Products { get; } = products;
    public ICategoryRepository Categories { get; } = categories;

    public async Task<bool> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        await _context.SaveChangesAsync(cancellationToken);
        return true;
    }

    public IDbContextTransaction BeginTransaction() => _context.Database.BeginTransaction();

    public void Dispose() { _context.Dispose(); GC.SuppressFinalize(this); }
}
```

Handlers inject `IUnitOfWork` only, never a repository or `DbContext` directly.

## 7. DI auto-registration convention

Rather than hand-writing `services.AddScoped<IFoo, Foo>()` for every repository/service, scan the assembly once. Two ways to do it — pick one per project, don't mix:

### 7a. Reference-project convention: reflection + name suffix

Scan the assembly once and bind by name suffix:

```csharp
// Infrastructure(.Tenant)/IoCExtension.cs
private static void RegisterInterfaces(IServiceCollection services)
{
    Assembly.GetExecutingAssembly()
        .GetTypes()
        .Where(t => (t.Name.EndsWith("Repository") || t.Name.EndsWith("Service"))
                    && !t.IsAbstract && !t.IsInterface)
        .Select(t => new { type = t, interfaces = t.GetInterfaces().ToList() })
        .ToList()
        .ForEach(x => x.interfaces.ForEach(i => services.AddScoped(i, x.type)));
}
```

Implications to document for future contributors:
- A class named `FooRepository`/`FooService` with **no interface** is not registered — give it one.
- A class implementing two interfaces gets the **same scoped instance** for both.
- Anything that doesn't match the suffix convention (e.g. `FooFactory`, `FooForwarder`) needs an explicit `services.AddScoped<IFoo, Foo>()` call — that's expected, not a bug.
- Everything registered this way is **Scoped, unconditionally** — a class that genuinely needs Transient or Singleton lifetime still needs a manual `services.AddTransient<...>()`/`AddSingleton<...>()` call outside this scan.
- Two unrelated classes ending in `Repository`/`Service` that happen to implement the *same* interface both get bound to it silently — last one registered wins, with no warning.

### 7b. Alternative (recommended for new projects): Scrutor + marker interfaces

[Scrutor](https://github.com/khellang/Scrutor) replaces the name-suffix guess with an explicit marker interface per lifetime, and lets each type opt into Transient/Scoped/Singleton individually instead of forcing everything to Scoped. It also **throws at startup** on an accidental duplicate registration instead of silently picking one.

```csharp
// Domain/Common/ITransient.cs, IScoped.cs, ISingleton.cs — empty marker interfaces
public interface ITransient { }
public interface IScoped { }
public interface ISingleton { }
```

```csharp
// Infrastructure(.Tenant)/DependencyInjection/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddTransientAsMatchingInterfaces(
        this IServiceCollection services, Assembly assembly) =>
        services.Scan(scan =>
            scan.FromAssemblies(assembly)
                .AddClasses(filter => filter.AssignableTo<ITransient>(), false)
                .UsingRegistrationStrategy(RegistrationStrategy.Throw)
                .AsImplementedInterfaces(type => type.Name != nameof(ITransient))
                .WithTransientLifetime());

    public static IServiceCollection AddScopedAsMatchingInterfaces(
        this IServiceCollection services, Assembly assembly) =>
        services.Scan(scan =>
            scan.FromAssemblies(assembly)
                .AddClasses(filter => filter.AssignableTo<IScoped>(), false)
                .UsingRegistrationStrategy(RegistrationStrategy.Throw)
                .AsImplementedInterfaces(type => type.Name != nameof(IScoped))
                .WithScopedLifetime());

    public static IServiceCollection AddSingletonAsMatchingInterfaces(
        this IServiceCollection services, Assembly assembly) =>
        services.Scan(scan =>
            scan.FromAssemblies(assembly)
                .AddClasses(filter => filter.AssignableTo<ISingleton>(), false)
                .UsingRegistrationStrategy(RegistrationStrategy.Throw)
                .AsImplementedInterfaces(type => type.Name != nameof(ISingleton))
                .WithSingletonLifetime());
}
```

```csharp
// Infrastructure(.Tenant)/IoCExtension.cs
public static IServiceCollection AddInfrastructureTenantServices(this IServiceCollection services, ...)
{
    var assembly = typeof(ApplicationDbContext).Assembly;
    services.AddTransientAsMatchingInterfaces(assembly);
    services.AddScopedAsMatchingInterfaces(assembly);
    services.AddSingletonAsMatchingInterfaces(assembly);
    // ... DbContext, interceptors, etc.
    return services;
}
```

```csharp
// Infrastructure(.Tenant)/Data/Repositories/ProductRepository.cs
public class ProductRepository(ApplicationDbContext context) : BaseRepository<Product, int>(context), IProductRepository, IScoped;
```

Trade-offs vs 7a:
- Naming is now cosmetic — `IProductRepository`/`ProductRepository` no longer need the `Repository`/`Service` suffix to be picked up; the marker interface is what matters. Keep the naming convention in §3 anyway for readability, just know it's no longer functionally load-bearing.
- Forgetting the marker interface on a new class still fails the same way as 7a — the class is silently not registered. That risk doesn't go away, it just moves from "wrong suffix" to "missing marker interface".
- Requires the `Scrutor` package (see §14) and a project reference from wherever the marker interfaces live (`Domain/Common`) to every project whose types need scanning — already satisfied since every layer references `Domain` transitively.

## 8. CQRS feature layout

`ICommand<TResponse>`/`IQuery<TResponse>` mark a request as a write or a read — both just extend MediatR's `IRequest<TResponse>`, so nothing about dispatch changes. The point is letting a pipeline behavior target one and not the other (e.g. a transaction-wrapping behavior on `ICommand<TResponse>` only, a caching behavior on `IQuery<TResponse>` only) — `ValidationBehavior` (§9) applies to both, so it stays constrained to plain `IRequest<TResponse>`.

```csharp
// Application/Common/Abstractions/Messaging/ICommand.cs
public interface ICommand<TResponse> : IRequest<TResponse>;
```

```csharp
// Application/Common/Abstractions/Messaging/IQuery.cs
public interface IQuery<TResponse> : IRequest<TResponse>;
```

One folder per command/query. Example — `Products/Commands/CreateProduct/`:

```csharp
// CreateProductCommand.cs
namespace YourApp.Application.Products.Commands.CreateProduct;

public sealed record CreateProductCommand(CreateProductRequestDto Request) : ICommand<int>;

public class CreateProductCommandHandler(IUnitOfWork unitOfWork, IMapper mapper, ILogger<CreateProductCommandHandler> logger)
    : IRequestHandler<CreateProductCommand, int>
{
    public async Task<int> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        try
        {
            var entity = mapper.Map<Product>(request.Request);
            await unitOfWork.Products.AddAsync(entity, cancellationToken);
            await unitOfWork.SaveChangesAsync(cancellationToken);
            return entity.Id;
        }
        catch (NotFoundException) { throw; }
        catch (ConflictDataException) { throw; }
        catch (Exception ex)
        {
            // Do NOT rewrap as InvalidDataException here — that maps to 400 and
            // surfaces ex.Message verbatim to the client (see §10 table). Only
            // rethrow/wrap exceptions you can attribute to a known client-facing
            // cause; let anything else propagate so ExceptionHandlerMiddleware's
            // generic 500 (message hidden, full detail logged) applies instead.
            logger.LogError(ex, "Failed to create product");
            throw;
        }
    }
}
```

```csharp
// CreateProductRequestDto.cs
namespace YourApp.Application.Products.Commands.CreateProduct;

public class CreateProductRequestDto : IMapFrom<Product>
{
    public string Code { get; set; }
    public string Name { get; set; }
}
```

```csharp
// CreateProductValidator.cs
namespace YourApp.Application.Products.Commands.CreateProduct;

public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Request.Code).NotEmpty().MaximumLength(20);
        RuleFor(x => x.Request.Name).NotEmpty().MaximumLength(200);
    }
}
```

Queries follow the same shape with `IQuery<TResponse>` instead of `ICommand<TResponse>` (`GetProductQuery : IQuery<ProductDto>` + handler returning a DTO, no validator needed unless the query has meaningful input constraints).

`IMapFrom<TEntity>` on a DTO auto-registers an AutoMapper profile — no hand-written `Profile` class needed for the common case.

## 9. Validation pipeline

Register an **open MediatR pipeline behavior** once; every request with a matching `IValidator<T>` is auto-validated before the handler runs — handlers never defensively re-validate.

```csharp
// Application/IoCExtension.cs
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(IoCExtension).Assembly);
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
services.AddValidatorsFromAssembly(typeof(IoCExtension).Assembly);
```

```csharp
public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators) : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(validators.Select(v => v.ValidateAsync(context, cancellationToken))))
            .SelectMany(r => r.Errors).Select(e => e.ErrorMessage).ToList();

        if (failures.Count > 0)
            throw new ValidationException(string.Join(";", failures));

        return await next();
    }
}
```

## 10. Error handling — exceptions, not a Result type

Custom exceptions in `Application/Common/Exceptions/`:

| Exception | When |
|---|---|
| `NotFoundException` | Entity id doesn't exist |
| `ConflictDataException` | Unique-constraint violation / optimistic-concurrency mismatch |
| `NoDataChangedException` | Update produced no row change |
| `InvalidConfigurationException` | A required config row/SP is missing |
| `InvalidDataException` | Generic wrapper for unexpected failures caught in a handler |

Handler pattern (see §8) — catch known typed exceptions and rethrow; for anything else, log and rethrow the original exception (don't rewrap it as `InvalidDataException`, or its message leaks to the client as a 400 instead of the generic, detail-hidden 500). Never `catch (Exception)` and swallow.

`ExceptionHandlerMiddleware` (registered in the `WebAPI` pipeline) maps every exception that escapes a handler to a status code + envelope body:

| Exception | Status |
|---|---|
| `ValidationException` | 400 |
| `NotFoundException` | 404 |
| `ConflictDataException` / `NoDataChangedException` | 409 |
| `InvalidConfigurationException` | 500 |
| `InvalidDataException` | 400 |
| anything else | 500 (generic message to client; full detail logged) |

Don't use `ProblemDetails` (RFC 9457) unless the whole team agrees — pick one envelope shape and keep every client on it.

## 11. Controllers + response envelope

```csharp
// Models/ResponseModel.cs
public class ResponseModel<T>
{
    public bool Success { get; set; }
    public T? Data { get; set; }
    public string? Message { get; set; }
}

public class ResponsePagedListModel<T>
{
    public bool Success { get; set; }
    public T? Data { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages { get; set; }
    public int TotalRecords { get; set; }
}

// Application/Common/Models/PagedListResponse.cs — internal paged-query result; BaseController.Success<T> below wraps it into ResponsePagedListModel<T> for the API response
public class PagedListResponse<T>
{
    public List<T> Data { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages { get; set; }
    public int TotalRecords { get; set; }
}
```

```csharp
// Controllers/BaseController.cs
public class BaseController : ControllerBase
{
    protected IActionResult Success<T>(T data) where T : class
        => Ok(new ResponseModel<T> { Success = true, Data = data });

    protected IActionResult Success(int id)
        => Ok(new ResponseModel<int> { Success = true, Data = id });

    protected IActionResult Success<T>(PagedListResponse<T> data) where T : class
        => Ok(new ResponsePagedListModel<IEnumerable<T>>
        {
            Success = true, Data = data.Data,
            Page = data.Page, PageSize = data.PageSize,
            TotalPages = data.TotalPages, TotalRecords = data.TotalRecords
        });

    protected IActionResult CreatedResult<T>(T data, string uri)
        => Created(uri, new ResponseModel<T> { Success = true, Data = data });

    protected IActionResult Failure()
    {
        var message = string.Join(" | ", ModelState.Values.SelectMany(v => v.Errors).Select(e => e.ErrorMessage));
        throw new InvalidDataException(message);
    }
}
```

```csharp
// Controllers/ProductsController.cs
[Route("Products")]
[Authorize(AuthenticationSchemes = "Bearer")]
public class ProductsController(ISender sender) : BaseController
{
    [HttpPost]
    [Authorize(Policy = "Products.Create")]
    public async Task<IActionResult> Create(CreateProductRequestDto request, CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid) return Failure();
        var id = await sender.Send(new CreateProductCommand(request), cancellationToken);
        return CreatedResult(id, $"/Products/{id}");
    }
}
```

Routes are `[Route("EntityNamesPlural")]`, PascalCase. Class-level `[Authorize(AuthenticationSchemes = "Bearer")]` always; every mutating action carries an explicit `[Authorize(Policy = "...")]` — never a bare `[Authorize]` on a write endpoint. Inject `ISender`, not `IMediator` — controllers only ever `Send()`, never `Publish()`, so the narrower interface is the correct fit (ISP).

## 12. Multi-tenancy (only if the new project needs it — otherwise skip this section)

Pattern used by the reference project, generalized:

```
Request → bearer token → a tenant-identifying claim (e.g. "tenant" / "org")
        → TenantIdentifierMiddleware stores it in HttpContext.Items["TENANT"]
        → first request for a never-seen tenant takes a distributed lock and
          runs Database.Migrate() for that tenant
        → ApplicationDbContext reads HttpContext.Items["TENANT"] and resolves
          its connection string from a master-DB "org config" table
```

Non-negotiables to carry over if you adopt this pattern:

1. The tenant-scoped `DbContext` **only resolves inside an HTTP request**. Background jobs / startup code must receive the tenant id explicitly and build the context manually — never rely on ambient `HttpContext`.
2. Every cache key for tenant-scoped data **must** include the tenant id (`$"{tenantId}:products:list"`), or one tenant's cache hit leaks into another tenant's response.
3. Never construct a tenant `DbContext` with a different tenant's connection string mid-request.
4. A new EF migration runs against **every** tenant on their first request after deploy — treat migrations as production-affecting and make them safe for concurrent first-run across tenants.

If the new project is single-tenant, drop `Infrastructure.Tenant` as a separate project, fold it into `Infrastructure`, and skip `TenantIdentifierMiddleware` entirely.

## 13. Composition root — `Program.cs` order

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((context, config) => config.ReadFrom.Configuration(context.Configuration)); // app code logs via ILogger<T>, never Serilog.Log directly

// Authentication (OAuth2/OIDC, or whatever the project's IdP is) + policy-based authorization
// — build named policies from config, not hardcoded in controllers
builder.Services.AddHttpContextAccessor();          // required if ApplicationDbContext resolves tenant from HttpContext.Items (§12)
builder.Services.AddApplicationServices();          // Application/IoCExtension.cs
builder.Services.AddInfrastructureServices(...);     // master/non-tenant, if applicable
builder.Services.AddInfrastructureTenantServices(...); // or just AddInfrastructureServices in single-tenant
builder.Services.AddControllers().AddNewtonsoftJson(...); // pick ONE serializer for the whole API
builder.Services.AddSwaggerGen(...);

var app = builder.Build();

// Initialise/seed the database on startup (master DB in multi-tenant)

if (app.Environment.IsDevelopment()) { app.UseSwagger(); app.UseSwaggerUI(); }

app.UseSerilogRequestLogging();
app.UseExceptionHandlerMiddleware(); // custom middleware — as early as possible, so it wraps everything below
app.UseHttpsRedirection();
app.UseCors(...);                 // AFTER UseHttpsRedirection, BEFORE auth
app.UseAuthentication();
app.UseTenantIdentifier();           // only if multi-tenant — needs the auth'd user's claims, must run BEFORE authorization/controllers
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Order matters, and **middleware registered after `MapControllers()` never runs for a matched request** — in the minimal hosting model, endpoint execution is wired in at the point `MapControllers()` is called, so anything added later falls outside the chain that wraps it. Concretely:
- `ExceptionHandlerMiddleware` must be registered near the very top (right after logging) — not after `MapControllers()` — or it will never see exceptions thrown by MediatR handlers/controllers.
- `TenantIdentifierMiddleware` must run after `UseAuthentication()` (it needs the tenant claim from the authenticated user) but still before `UseAuthorization()`/`MapControllers()` — never after, or `ApplicationDbContext` will try to resolve a tenant that was never set.
- CORS goes before auth, as before.

## 14. Recommended package set (pin exact versions; keep `Microsoft.*` packages on the same major)

| Concern | Package | Notes |
|---|---|---|
| CQRS mediator | `MediatR` | Register the open `ValidationBehavior<,>`. **License:** v13+ (2025) is commercial (paid per-organization); pin to the last Apache-2.0 release (12.x) or budget for a license before adopting later versions |
| DTO mapping | `AutoMapper` | `IMapFrom<T>` convention for profile auto-discovery. **License:** same commercial transition as MediatR (same author) — pin an Apache-2.0-era version or budget for a license |
| Validation | `FluentValidation` | Validators auto-registered from assembly |
| ORM | `Microsoft.EntityFrameworkCore` + `.SqlServer` + `.Design` | Target is SQL Server; §6b/§6c's raw SQL is T-SQL-specific regardless of EF provider |
| DI assembly scanning | `Scrutor` | Only if using §7b's marker-interface registration instead of §7a's reflection scan |
| Micro-ORM | `Dapper` | Only if using §6b's `DatabaseManager` / §6c's `BulkManager` |
| SQL Server client | `Microsoft.Data.SqlClient` | `SqlConnection`/`SqlBulkCopy` — needed by §6b/§6c |
| Dynamic LINQ | `System.Linq.Dynamic.Core` | Only if using §6a's `SearchTable`/`DoFilter`/`DoQuery` (string-based `Where`/`OrderBy`); pin a recent version — it has had past CVEs around unrestricted type resolution |
| Logging | `Microsoft.Extensions.Logging.Abstractions` | Inject `ILogger<T>`; wire whatever provider/sink in `Program.cs` |
| JSON | Pick **one**: `System.Text.Json` (default) or `Newtonsoft.Json` — don't mix | The reference project uses Newtonsoft; a fresh project should default to `System.Text.Json` unless there's a reason not to |
| API docs | `Swashbuckle.AspNetCore` | JWT bearer security scheme |
| Background jobs | `Hangfire` (+ SQL Server storage) | Only if the project needs scheduled/background work |
| Distributed cache | `StackExchange.Redis` / `Microsoft.Extensions.Caching.StackExchangeRedis` | Cache keys must include tenant id if multi-tenant |
| Distributed lock | `Medallion.Threading.SqlServer` | Only needed for the multi-tenant first-run-migration pattern |
| Tests | `xunit` + `Microsoft.NET.Test.Sdk` + `FluentAssertions` (stay on 7.x if license matters — 8.0+ is commercial) | |

Don't introduce a Result-pattern library (`OneOf`, `FluentResults`, `ErrorOr`) — this architecture uses exceptions + middleware (§10), and mixing patterns is worse than picking either one consistently.

## 15. "Add a new feature end-to-end" checklist

1. **Domain entity** → `Domain/Entities/{Entity}.cs` (right base class per §4).
2. **EF configuration** → `Infrastructure(.Tenant)/Data/Configurations/{Entity}Configuration.cs`.
3. **Repository interface** → `Application/Common/Interfaces/I{Entity}Repository.cs`.
4. **Repository implementation** → `Infrastructure(.Tenant)/Data/Repositories/{Entity}Repository.cs` (extends `BaseRepository<T,TKey>`, name ends `Repository`).
5. **Add to `IUnitOfWork` + `UnitOfWork`.**
6. **DTOs** → `Application/{Feature}/{Entity}Dto.cs` or per-command under `Commands/{Name}/`.
7. **Command/Query** → `Application/{Feature}/Commands|Queries/{Name}/{Name}Command.cs|Query.cs` + handler.
8. **Validator** in the same folder.
9. **Service interface** (only if business logic doesn't belong in the handler) → `Application/Common/Services/I{Feature}Service.cs`.
10. **Service implementation** → `Infrastructure(.Tenant)/Services/{Feature}Service.cs`.
11. **Controller** → `WebAPI/Controllers/{Feature}Controller.cs` (inherits `BaseController`, dispatches via `ISender`, returns through `Success`/`Failure`/`CreatedResult`).
12. **Migration** → `dotnet ef migrations add Add{Entity} --context ApplicationDbContext` (or the PMC equivalent).

## 16. Common pitfalls to flag in review

- Handler injects `DbContext` directly instead of `IUnitOfWork`.
- Write command has no validator → `ValidationBehavior` silently doesn't run (no `IValidator<T>` registered for it).
- Repository/service class name doesn't end in `Repository`/`Service` (§7a) or is missing its lifetime marker interface (§7b) → DI auto-registration skips it silently.
- New repository not added to `IUnitOfWork`/`UnitOfWork` → handlers can't resolve it.
- Controller returns raw `Ok(...)`/`BadRequest(...)` instead of the envelope helpers → breaks client contract.
- Cache key for tenant-scoped data missing the tenant id (multi-tenant projects only).
- `CancellationToken` accepted by the handler but not passed down to the repository/EF call.
- Custom middleware (`ExceptionHandlerMiddleware`, `TenantIdentifierMiddleware`) registered after `app.MapControllers()` in `Program.cs` → silently never runs for matched requests (see §13).
- Handler catches an unexpected exception and rewraps it as `InvalidDataException` → leaks internal error detail to the client as a 400 instead of the generic, detail-hidden 500 (see §10).
- Entities fetched via `GetByIdAsync`/`FindAsync` are `AsNoTracking()` (see §6); calling `Update(entity)` on one reattaches it as fully `Modified` (every column, not just changed ones) and, if the entity carries an optimistic-concurrency token, bypasses EF's concurrency check unless the original token value survived the round-trip through the DTO.
- §6a's `…ForProc` filter builders emit a literal SQL fragment for a stored procedure — a missing/incorrect value-escaping step there is a SQL-injection hole, not just a bug. Any change to `GetSqlLiteralString`/`GetFilterQueryStringForProc` needs a security-focused review, not just a functional one.
- `DatabaseManager`/`BulkManager` (§6b/§6c) forgotten from the explicit `services.AddScoped<IFoo>(sp => ...)` registration → resolves to "no service registered" at first use, not at startup, since neither §7a's suffix scan nor §7b's marker-interface scan can construct a type whose constructor takes a plain `string`.
- §6c's `tableName`/`keyColumnName`/column-name interpolation into `CREATE TABLE`/`MERGE` SQL text is only safe when those names come from trusted application code — passing a client-supplied table or column name into `BulkInsertAsync`/`BulkUpdateAsync` is a SQL-injection path with no parameterization escape hatch (T-SQL can't parameterize identifiers).
- A per-entity repository interface (`I{Entity}Repository`) hand-declaring CRUD methods instead of extending the shared `IRepository<T, TKey>` (§6) → the moment `BaseRepository`'s method shapes change, that interface stops matching and no concrete repository compiles.
- `SearchTable`/`DoFilter`/`DoQuery` (§6a) used without `global using System.Linq.Dynamic.Core;` in scope → `query.Where(filterScript)` silently resolves to plain LINQ's `Where` instead of Dynamic LINQ's string-based overload, and the resulting compiler error doesn't mention a missing `using` at all.

---

### What was deliberately left out of this guide

- Any company name, ticket-tracker id scheme, internal service names, real auth provider config, connection strings, or secrets.
- The full 80+-repository `IUnitOfWork` from the reference project — §6 shows the shape with two example repositories; grow it one entity at a time.
- The reference project's dynamic advanced-search/filter-builder repository extensions and stored-procedure execution helpers are now included as the optional §6a — add them only when a paged advanced-search endpoint is actually needed, and read the security note on the `…ForProc` builders before using that path.
- Git branching/commit conventions and a `.claude/rules` + `.claude/skills` governance layer — the reference project also encodes its conventions as auto-loaded Markdown rules for AI coding agents. Worth replicating in a new project once the team has enough conventions worth pinning down, but it's a documentation-process decision, not part of the architecture itself.
