---
layout: default
title: "So you want to write an EF Core provider..."
date: 2016-11-11 15:15
day: 11th
month: November
year: 2016
author: ajcvickers
permalink: 2016/11/11/so-you-want-to-write-an-ef-core-provider/
---

# EF Core 1.1
# So you want to write an EF Core provider...

Writing a database provider for EF Core can be daunting. I have written several posts over the last month containing information you need to know when writing a provider. This post pulls all that together to give an overview of the different building blocks with links to previous posts containing the details.



<h2>Step 1: Implement services</h2>

<h3>Implement IDatabaseProviderServices</h3>

For a non-relational provider, create a class <code>MyProviderDatabaseProviderServices</code> that inherits from <code>DatabaseProviderServices</code> and override its properties. The abstract properties define services for which a provider must supply an implementation. You may also need to override some of the virtual properties to return provider-specific implementations.

Do the same for a relational provider, except your class should inherit from <code>RelationalDatabaseProviderServices</code>.

The properties should be implemented as <code>GetService</code> calls for your provider's implementation of the service. For example:

``` c#
public class SqlServerDatabaseProviderServices : RelationalDatabaseProviderServices
{
    public SqlServerDatabaseProviderServices(IServiceProvider services)
        : base(services)
    {
    }

    public override IValueGeneratorSelector ValueGeneratorSelector
        => GetService<SqlServerValueGeneratorSelector>();

    public override IRelationalDatabaseCreator RelationalDatabaseCreator
        => GetService<SqlServerDatabaseCreator>();

    // More...
}
```

<h3>Create an "AddEntityFramework..." extension method</h3>

Create an extension method on <code>IServiceCollection</code> called "AddEntityFrameworkMyProvider". For example, <code>AddEntityFrameworkSqlServer</code>. This method should:

<ul>
<li>Call <code>AddEntityFramework()</code> for a non-relational provider, or <code>AddRelational()</code> for a relational provider.</li>
<li>Register <code>IDatabaseProvider</code> as DatabaseProvider<MyProviderDatabaseProviderServices, MyProviderOptionsExtension>.</li>
<li>Register the services for which you have GetService calls in your DatabaseProviderServices class.</li>
<li>Services should be registered as singletons unless they depend on other services that are scoped. Note that many EF services that you may need to depend on are scoped.</li>
</ul>

For example:

``` c#
public static IServiceCollection AddEntityFrameworkSqlServer(this IServiceCollection services)
{
    services.AddRelational();

    services.TryAddEnumerable(ServiceDescriptor
        .Singleton<IDatabaseProvider, DatabaseProvider<SqlServerDatabaseProviderServices, SqlServerOptionsExtension>>());

    services.TryAdd(new ServiceCollection()
        .AddSingleton<SqlServerTypeMapper>()
        .AddScoped<SqlServerValueGeneratorSelector>()
        .AddScoped<SqlServerDatabaseCreator>()
        // More...
        );

    return services;
}
```

See <a href="/2016/11/01/ef-core-dependency-injection-internals/">this post</a> for more information on provider services.

<h2>Step 2: Implement a 'Use...' method</h2>

<h3>Create an options extension</h3>

Create a class <code>MyProviderOptionsExtension</code> that implements <code>IDbContextOptionsExtension</code>. This class should:

<ul>
<li>Implement <code>ApplyServices</code> to call the <code>AddEntityFrameworkMyProvider</code> defined above.</li>
<li>Have properties for provider configuration, such as the connection string.</li>
<li>Have a copy-constructor for use when it is being cloned.</li>
</ul>

When implementing a relational provider your class should inherit from <code>RelationalOptionsExtension</code>. For example:

``` c#
public class SqlServerOptionsExtension : RelationalOptionsExtension
{
    private bool? _rowNumberPaging;

    public SqlServerOptionsExtension()
    {
    }

    public SqlServerOptionsExtension([NotNull] SqlServerOptionsExtension copyFrom)
        : base(copyFrom)
    {
        _rowNumberPaging = copyFrom._rowNumberPaging;
    }

    public virtual bool? RowNumberPaging
    {
        get { return _rowNumberPaging; }
        set { _rowNumberPaging = value; }
    }

    public override void ApplyServices(IServiceCollection services)
        => services.AddEntityFrameworkSqlServer();
}
```

<h3>Create a 'Use...' method</h3>

Create an extension method on <code>DbContextOptionsBuilder</code> called <code>UseMyProvider</code>. This method should:

<ul>
<li>Take any required provider configuration, such as the connection string.</li>
<li>Ensure that your options extension is registered.</li>
<li>Set provider configuration on the options extension.</li>
<li>Have an overload for the generic DbContextOptionsBuilder<TContext></li>
<li>Create a nested closure overload for any optional provider configuration.</li>
</ul>

For example:

``` c#
public static DbContextOptionsBuilder UseSqlServer(
    this DbContextOptionsBuilder optionsBuilder,
    string connectionString,
    Action<SqlServerDbContextOptionsBuilder> sqlServerOptionsAction = null)
{
    var extension = GetOrCreateExtension(optionsBuilder);
    extension.ConnectionString = connectionString;
    ((IDbContextOptionsBuilderInfrastructure)optionsBuilder).AddOrUpdateExtension(extension);

    sqlServerOptionsAction?.Invoke(new SqlServerDbContextOptionsBuilder(optionsBuilder));

    return optionsBuilder;
}

public static DbContextOptionsBuilder<TContext> UseSqlServer<TContext>(
    this DbContextOptionsBuilder<TContext> optionsBuilder,
    string connectionString,
    Action<SqlServerDbContextOptionsBuilder> sqlServerOptionsAction = null)
    where TContext : DbContext
    => (DbContextOptionsBuilder<TContext>)UseSqlServer(
        (DbContextOptionsBuilder)optionsBuilder, connectionString, sqlServerOptionsAction);
```

See <a href="/2016/11/03/implementing-a-provider-use-method-for-ef-core-1-1/">this post</a> for more details on options extensions and creating a 'Use...' method.

<h2>Step 3: Create metadata extension methods</h2>

<h3>Define annotations</h3>

Create an annotation prefix name for your provider and names for any provider-specific metadata. Use these names to create a MyProviderFullAnnotationNames class, inheriting from RelationalFullAnnotationNames if your provider is relational. For example:

``` c#
public class SqlServerFullAnnotationNames : RelationalFullAnnotationNames
{
    protected SqlServerFullAnnotationNames(string prefix)
        : base(prefix)
    {
        Clustered = prefix + SqlServerAnnotationNames.Clustered;
        MemoryOptimized = prefix + SqlServerAnnotationNames.MemoryOptimized;
    }

    public new static SqlServerFullAnnotationNames Instance { get; } 
        = new SqlServerFullAnnotationNames(SqlServerAnnotationNames.Prefix);

    public readonly string Clustered;
    public readonly string MemoryOptimized;
}
```

<h3>Create MyProvider() extension methods</h3>

Create extension methods named after your provider for each mutable and immutable metadata type. For example:

``` c#
public static SqlServerEntityTypeAnnotations SqlServer(this IMutableEntityType entityType)
    => (SqlServerEntityTypeAnnotations)SqlServer((IEntityType)entityType);

public static ISqlServerEntityTypeAnnotations SqlServer(this IEntityType entityType)
    => new SqlServerEntityTypeAnnotations(entityType);

public static SqlServerPropertyAnnotations SqlServer(this IMutableProperty property)
    => (SqlServerPropertyAnnotations)SqlServer((IProperty)property);

public static ISqlServerPropertyAnnotations SqlServer(this IProperty property)
    => new SqlServerPropertyAnnotations(property);
```

Create the IMyProviderXxxAnnotations interfaces returned from these methods, inheriting from IRelationalXxxAnnotations for relational providers. Create classes that implement these interfaces, inheriting from RelationalXxxAnnotations for relational providers. For example:

``` c#
public interface ISqlServerEntityTypeAnnotations : IRelationalEntityTypeAnnotations
{
    bool IsMemoryOptimized { get; }
}

public class SqlServerEntityTypeAnnotations
    : RelationalEntityTypeAnnotations, ISqlServerEntityTypeAnnotations
{
    public SqlServerEntityTypeAnnotations(IEntityType entityType)
        : base(entityType, SqlServerFullAnnotationNames.Instance)
    {
    }

    public virtual bool IsMemoryOptimized
    {
        get { return EntityType[SqlServerFullAnnotationNames.Instance.MemoryOptimized] as bool? ?? false; }
        set { ((IMutableAnnotatable)EntityType)[SqlServerFullAnnotationNames.Instance.MemoryOptimized] = value; }
    }
}
```

Make sure that the interface defines read-only properties, while the class adds in setters. The interface is used for immutable metadata, while the class is used with mutable metadata.

<h3>Create fluent API extensions</h3>

Create fluent API extension methods that make use of the core metadata extensions described above. These methods should be named <code>MyProviderDoSomething</code>. Make sure to include generic and non-generic overloads of these methods. For example:

``` c#
public static class SqlServerEntityTypeBuilderExtensions
{
    public static EntityTypeBuilder ForSqlServerIsMemoryOptimized(
        this EntityTypeBuilder entityTypeBuilder, bool memoryOptimized = true)
    {
        entityTypeBuilder.Metadata.SqlServer().IsMemoryOptimized = memoryOptimized;

        return entityTypeBuilder;
    }

    public static EntityTypeBuilder<TEntity> ForSqlServerIsMemoryOptimized<TEntity>(
        this EntityTypeBuilder<TEntity> entityTypeBuilder, bool memoryOptimized = true)
        where TEntity : class
        => (EntityTypeBuilder<TEntity>)ForSqlServerIsMemoryOptimized((EntityTypeBuilder)entityTypeBuilder, memoryOptimized);
}
```

Note that these methods can contain logic to obtain their values from the model hierarchy and to return reasonable defaults.

See <a href="/2016/11/10/implementing-provider-extension-methods-in-ef-core-1-1/">this post</a> for more information on metadata extensions.

<h2>Look at examples</h2>

The <a href="https://github.com/aspnet/EntityFramework">EF Core codebase on GitHub</a> contains relational providers for SQL Server, SQLite, and a non-relational in-memory store. This code is the best place to look for guidance on how to implement a provider.

<h2>Summary</h2>

Creating an EF Core database provider requires implementation of many services together with extension methods to integrate with the API surface. Following the patterns outlined in this post will result in a provider that is consistent with the experience for other providers making it easy for developers to switch between different providers.
