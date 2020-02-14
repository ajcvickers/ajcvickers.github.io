---
layout: post
title: So you want to write an EF Core provider...
date: 2016-11-11 15:15
author: ajcvickers
comments: true
categories: [Dependency Injection, EF Core, EF Core Metadata, EF Core Provider, Entity Framework, Metadata]
---
Writing a database provider for EF Core can be daunting. I have written several posts over the last month containing information you need to know when writing a provider. This post pulls all that together to give an overview of the different building blocks with links to previous posts containing the details.

<!--more-->

<h2>Step 1: Implement services</h2>

<h3>Implement IDatabaseProviderServices</h3>

For a non-relational provider, create a class <code>MyProviderDatabaseProviderServices</code> that inherits from <code>DatabaseProviderServices</code> and override its properties. The abstract properties define services for which a provider must supply an implementation. You may also need to override some of the virtual properties to return provider-specific implementations.

Do the same for a relational provider, except your class should inherit from <code>RelationalDatabaseProviderServices</code>.

The properties should be implemented as <code>GetService</code> calls for your provider's implementation of the service. For example:

[code lang=csharp]
public class SqlServerDatabaseProviderServices : RelationalDatabaseProviderServices
{
    public SqlServerDatabaseProviderServices(IServiceProvider services)
        : base(services)
    {
    }

    public override IValueGeneratorSelector ValueGeneratorSelector
        =&gt; GetService&lt;SqlServerValueGeneratorSelector&gt;();

    public override IRelationalDatabaseCreator RelationalDatabaseCreator
        =&gt; GetService&lt;SqlServerDatabaseCreator&gt;();

    // More...
}
[/code]

<h3>Create an "AddEntityFramework..." extension method</h3>

Create an extension method on <code>IServiceCollection</code> called "AddEntityFrameworkMyProvider". For example, <code>AddEntityFrameworkSqlServer</code>. This method should:

<ul>
<li>Call <code>AddEntityFramework()</code> for a non-relational provider, or <code>AddRelational()</code> for a relational provider.</li>
<li>Register <code>IDatabaseProvider</code> as DatabaseProvider&lt;MyProviderDatabaseProviderServices, MyProviderOptionsExtension&gt;.</li>
<li>Register the services for which you have GetService calls in your DatabaseProviderServices class.</li>
<li>Services should be registered as singletons unless they depend on other services that are scoped. Note that many EF services that you may need to depend on are scoped.</li>
</ul>

For example:

[code lang=csharp]
public static IServiceCollection AddEntityFrameworkSqlServer(this IServiceCollection services)
{
    services.AddRelational();

    services.TryAddEnumerable(ServiceDescriptor
        .Singleton&lt;IDatabaseProvider, DatabaseProvider&lt;SqlServerDatabaseProviderServices, SqlServerOptionsExtension&gt;&gt;());

    services.TryAdd(new ServiceCollection()
        .AddSingleton&lt;SqlServerTypeMapper&gt;()
        .AddScoped&lt;SqlServerValueGeneratorSelector&gt;()
        .AddScoped&lt;SqlServerDatabaseCreator&gt;()
        // More...
        );

    return services;
}
[/code]

See <a href="https://blog.oneunicorn.com/2016/11/01/ef-core-dependency-injection-internals/">this post</a> for more information on provider services.

<h2>Step 2: Implement a "Use..." method</h2>

<h3>Create an options extension</h3>

Create a class <code>MyProviderOptionsExtension</code> that implements <code>IDbContextOptionsExtension</code>. This class should:

<ul>
<li>Implement <code>ApplyServices</code> to call the <code>AddEntityFrameworkMyProvider</code> defined above.</li>
<li>Have properties for provider configuration, such as the connection string.</li>
<li>Have a copy-constructor for use when it is being cloned.</li>
</ul>

When implementing a relational provider your class should inherit from <code>RelationalOptionsExtension</code>. For example:

[code lang=csharp]
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
        =&gt; services.AddEntityFrameworkSqlServer();
}
[/code]

<h3>Create a "Use..." method</h3>

Create an extension method on <code>DbContextOptionsBuilder</code> called <code>UseMyProvider</code>. This method should:

<ul>
<li>Take any required provider configuration, such as the connection string.</li>
<li>Ensure that your options extension is registered.</li>
<li>Set provider configuration on the options extension.</li>
<li>Have an overload for the generic DbContextOptionsBuilder&lt;TContext&gt;</li>
<li>Create a nested closure overload for any optional provider configuration.</li>
</ul>

For example:

[code lang=csharp]
public static DbContextOptionsBuilder UseSqlServer(
    this DbContextOptionsBuilder optionsBuilder,
    string connectionString,
    Action&lt;SqlServerDbContextOptionsBuilder&gt; sqlServerOptionsAction = null)
{
    var extension = GetOrCreateExtension(optionsBuilder);
    extension.ConnectionString = connectionString;
    ((IDbContextOptionsBuilderInfrastructure)optionsBuilder).AddOrUpdateExtension(extension);

    sqlServerOptionsAction?.Invoke(new SqlServerDbContextOptionsBuilder(optionsBuilder));

    return optionsBuilder;
}

public static DbContextOptionsBuilder&lt;TContext&gt; UseSqlServer&lt;TContext&gt;(
    this DbContextOptionsBuilder&lt;TContext&gt; optionsBuilder,
    string connectionString,
    Action&lt;SqlServerDbContextOptionsBuilder&gt; sqlServerOptionsAction = null)
    where TContext : DbContext
    =&gt; (DbContextOptionsBuilder&lt;TContext&gt;)UseSqlServer(
        (DbContextOptionsBuilder)optionsBuilder, connectionString, sqlServerOptionsAction);
[/code]

See <a href="https://blog.oneunicorn.com/2016/11/03/implementing-a-provider-use-method-for-ef-core-1-1/">this post</a> for more details on options extensions and creating a "Use..." method.

<h2>Step 3: Create metadata extension methods</h2>

<h3>Define annotations</h3>

Create an annotation prefix name for your provider and names for any provider-specific metadata. Use these names to create a MyProviderFullAnnotationNames class, inheriting from RelationalFullAnnotationNames if your provider is relational. For example:

[code lang=csharp]
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
[/code]

<h3>Create MyProvider() extension methods</h3>

Create extension methods named after your provider for each mutable and immutable metadata type. For example:

[code lang=csharp]
public static SqlServerEntityTypeAnnotations SqlServer(this IMutableEntityType entityType)
    =&gt; (SqlServerEntityTypeAnnotations)SqlServer((IEntityType)entityType);

public static ISqlServerEntityTypeAnnotations SqlServer(this IEntityType entityType)
    =&gt; new SqlServerEntityTypeAnnotations(entityType);

public static SqlServerPropertyAnnotations SqlServer(this IMutableProperty property)
    =&gt; (SqlServerPropertyAnnotations)SqlServer((IProperty)property);

public static ISqlServerPropertyAnnotations SqlServer(this IProperty property)
    =&gt; new SqlServerPropertyAnnotations(property);
[/code]

Create the IMyProviderXxxAnnotations interfaces returned from these methods, inheriting from IRelationalXxxAnnotations for relational providers. Create classes that implement these interfaces, inheriting from RelationalXxxAnnotations for relational providers. For example:

[code lang=csharp]
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
[/code]

Make sure that the interface defines read-only properties, while the class adds in setters. The interface is used for immutable metadata, while the class is used with mutable metadata.

<h3>Create fluent API extensions</h3>

Create fluent API extension methods that make use of the core metadata extensions described above. These methods should be named <code>MyProviderDoSomething</code>. Make sure to include generic and non-generic overloads of these methods. For example:

[code lang=csharp]
public static class SqlServerEntityTypeBuilderExtensions
{
    public static EntityTypeBuilder ForSqlServerIsMemoryOptimized(
        this EntityTypeBuilder entityTypeBuilder, bool memoryOptimized = true)
    {
        entityTypeBuilder.Metadata.SqlServer().IsMemoryOptimized = memoryOptimized;

        return entityTypeBuilder;
    }

    public static EntityTypeBuilder&lt;TEntity&gt; ForSqlServerIsMemoryOptimized&lt;TEntity&gt;(
        this EntityTypeBuilder&lt;TEntity&gt; entityTypeBuilder, bool memoryOptimized = true)
        where TEntity : class
        =&gt; (EntityTypeBuilder&lt;TEntity&gt;)ForSqlServerIsMemoryOptimized((EntityTypeBuilder)entityTypeBuilder, memoryOptimized);
}
[/code]

Note that these methods can contain logic to obtain their values from the model hierarchy and to return reasonable defaults.

See <a href="https://blog.oneunicorn.com/2016/11/10/implementing-provider-extension-methods-in-ef-core-1-1/">this post</a> for more information on metadata extensions.

<h2>Look at examples</h2>

The <a href="https://github.com/aspnet/EntityFramework">EF Core codebase on GitHub</a> contains relational providers for SQL Server, SQLite, and a non-relational in-memory store. This code is the best place to look for guidance on how to implement a provider.

<h2>Summary</h2>

Creating an EF Core database provider requires implementation of many services together with extension methods to integrate with the API surface. Following the patterns outlined in this post will result in a provider that is consistent with the experience for other providers making it easy for developers to switch between different providers.
