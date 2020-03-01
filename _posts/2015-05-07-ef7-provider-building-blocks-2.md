---
layout: default
title: "EF7 Provider Building Blocks"
date: 2015-05-07 11:02
day: 7th
month: May
year: 2015
author: ajcvickers
permalink: 2015/05/07/ef7-provider-building-blocks-2/
---

# EF Core 1.0
# Provider Building Blocks

In this post I'll outline the basic building blocks needed for an EF7 provider. The idea is not to show how everything should be implemented, but rather to show what pieces are needed and how they fit together. The best examples of EF7 providers are the SQL Server and SQLite providers, which ca both be found in the <a href="https://github.com/aspnet/EntityFramework">EF repro on GitHub</a>.

EF7 providers should be shipped as NuGet packages. This post does not cover NuGet packaging, but you can look at the GitHib repro for some ideas on how to do this.



Note that EF7 is still very much pre-release software and the types and APIs described here are likely to change as we evolve towards the RTM release.

<h2>Dependency Injection</h2>

EF7 makes extensive use of <a href="http://en.wikipedia.org/wiki/Dependency_injection">dependency injection</a> (DI). That is, EF is a collection of services that work together and depend on each other largely through constructor injection. Implementing an EF provider is really a matter of creating provider-specific implementations of these services as needed.

<h2>Context scopes</h2>

A single EF7 application can make use of multiple providers. For example, the same application could access a SQL Server database and a SQLite database. However, the session defined by each context instance is restricted to using a single provider. So you can create one context instance to access SQL Server and another context instance to access SQLite, but you cannot create one context instance and access both SQLite and SQL Server with that single instance.

This is achieved through each context instance creating a new DI scope. Services for the provider in use are then resolved through that scope using the mechanisms outlined in this post.

<h2>Types of services</h2>

EF7 has three broad categories of services:

<ul>
<li>Core services, which are not intended to be implemented differently for different providers.</li>
<li>Provider-specific services for which there are no common concrete implementations. All providers must provide implementations of these services. However, there are often abstract base classes that can be used as the basis for provider-specific implementations.</li>
<li>Provider-specific services for which there are common concrete implementations. For these services, a provider only needs to provide an implementation if the common implementation needs to be changed in some way.</li>
</ul>

<h2>IDataStoreServices</h2>

As stated above, the selection of which provider to use for a given context instance is done by creating a new DI scope. Provider-specific services are then resolved for the scope using delegates that map from the contract interface to the provider-specific implementation. This mapping is done using an implementation of the IDataStoreServices interface, and therefore each provider must create its own implementation of this interface and register it with DI.

For relational providers there is an IRelationalDataStoreServices interface which adds additional services used by the EF7 relational infrastructure. Providers that connection to a relational database should implement this interface.

There are base implementations of IRelationalDataStoreServices and IDataStoreServices (called RelationalDataStoreServices and DataStoreServices respectively) that contain predefined mappings for provider-specific services for which there are common concrete implementations. Providers should generally use one of these base classes and only override virtual methods as needed.

For example, here is the implementation of IRelationalDataStoreServices used by the SQLite provider, as of the time of writing:

``` c#
public class SqliteDataStoreServices : RelationalDataStoreServices
{
    public SqliteDataStoreServices(IServiceProvider services)
        : base(services)
    {
    }

    public override IDataStoreConnection Connection 
        => GetService<SqliteDataStoreConnection>();

    public override IDataStoreCreator Creator 
        => GetService<SqliteDataStoreCreator>();

    public override IHistoryRepository HistoryRepository 
        => GetService<SqliteHistoryRepository>();

    public override IMigrationSqlGenerator MigrationSqlGenerator 
        => GetService<SqliteMigrationSqlGenerator>();

    public override IModelSource ModelSource 
        => GetService<SqliteModelSource>();

    public override IRelationalConnection RelationalConnection 
        => GetService<SqliteDataStoreConnection>();

    public override ISqlGenerator SqlGenerator 
        => GetService<SqliteSqlGenerator>();

    public override IDataStore Store 
        => GetService<SqliteDataStore>();

    public override IValueGeneratorCache ValueGeneratorCache 
        => GetService<SqliteValueGeneratorCache>();

    public override IRelationalTypeMapper TypeMapper 
        => GetService<SqliteTypeMapper>();

    public override IModificationCommandBatchFactory ModificationCommandBatchFactory 
        => GetService<SqliteModificationCommandBatchFactory>();

    public override ICommandBatchPreparer CommandBatchPreparer 
        => GetService<SqliteCommandBatchPreparer>();

    public override IRelationalDataStoreCreator RelationalDataStoreCreator 
        => GetService<SqliteDataStoreCreator>();
}
```

Notice that it derives from RelationalDataStoreServices to use common services where possible and overrides for all services where a custom implementation is required.

Also notice that each property is mapping a concrete type in the GetService call to the contract interface that other services will depend on. That is, services should take the contract interface, not the concrete implementation, in their constructors so that all services are programmed against the contracts of other services.

<h2>Registering services</h2>

Each of the concrete services must be registered in the DI container so that the GetService calls will have something to find. This is done in an AddXxx extension method, where Xxx is the name of the provider. For example, the extension method for SQLite looks like this:

``` c#
public static EntityFrameworkServicesBuilder AddSqlite(
    this EntityFrameworkServicesBuilder services)
{
    ((IAccessor<IServiceCollection>)services.AddRelational()).Service
        .AddSingleton<IDataStoreSource, SqliteDataStoreSource>()
        .TryAdd(new ServiceCollection()
            .AddSingleton<SqliteValueGeneratorCache>()
            .AddSingleton<SqliteSqlGenerator>()
            .AddScoped<SqliteTypeMapper>()
            .AddScoped<SqliteModificationCommandBatchFactory>()
            .AddScoped<SqliteCommandBatchPreparer>()
            .AddSingleton<SqliteModelSource>()
            .AddScoped<SqliteDataStoreServices>()
            .AddScoped<SqliteDataStore>()
            .AddScoped<SqliteDataStoreConnection>()
            .AddScoped<SqliteMigrationSqlGenerator>()
            .AddScoped<SqliteDataStoreCreator>()
            .AddScoped<SqliteHistoryRepository>());

    return services;
}
```

The AddXxx method is an extension method on EntityFrameworkServicesBuilder and should return the EntityFrameworkServicesBuilder that is passed in. This allows it to be chained from an AddEntityFramework call when registering DI services and allows additional AddXxx methods to be chained from it.

For a relational provider, a call To AddRelational is required to add all common relational services.

The registration of an IDataStoreSource  facilitates selection of a given provider as described below. The remaining calls register the concrete services that will be resolved by the provider's IDataStoreServices implementation.

<h2>Registration scopes</h2>

In most cases provider-specific services should be registered in DI as "scoped" using AddScoped. This is because the context creates scopes in order to use different providers for different context instances. Also, any service that depends on a scoped service must itself be registered as scoped.

In a few cases services can be registered using AddSingleton where the service does not depend on any other scoped service and a single instance can be used concurrently by all context instances--for example, the SqliteSqlGenerator in the code above. Note that all singleton services must be thread-safe.

There are currently two services (IModelSource and IValueGeneratorCache) which act as provider-specific caches. These services must be registered as singletons so that the cache persists across context instances, which is the point of having the cache. They therefore cannot depend on any scoped services. Also, each provider must use its own concrete implementation for these services so that each provider gets a different instance and cache collisions are avoided.

<h2>Provider selection</h2>

EF7 applications choose which provider to use for a given context instance my making a call to a UseXxx method, typically in the OnConfiguring method of the context. For example, to use SQLite an application might do something like this:

``` c#
protected override void OnConfiguring(
    DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlite("MyDb.db");
}
```

The UseSqlite method is an extension method on DbContextOptionsBuilder. Its job is to create (or update an existing) IDbContextOptionsExtension object and register it onto the options builder. For example:

``` c#
public static SqliteDbContextOptionsBuilder UseSqlite(
    this DbContextOptionsBuilder options,
    string connectionString)
{
    var extension = GetOrCreateExtension(options);
    extension.ConnectionString = connectionString;
    ((IOptionsBuilderExtender)options).AddOrUpdateExtension(extension);

    return new SqliteDbContextOptionsBuilder(options);
}
```

Note that a new builder with the updated options is returned to allow chaining of further configuration.

Presence of IDbContextOptionsExtension in the options builder is used by the EF stack to determine that a provider has been selected. It is also used to store provider-specific configuration for the session. Some of this configuration, like the connection string, is handled by the relational base class.

An IDbContextOptionsExtension is also used to register DI services automatically for the case where EF is taking care of all DI registration internally. An example implementation for SQLite (trimmed) looks something like:

``` c#
public class SqliteOptionsExtension : RelationalOptionsExtension
{
    public override void ApplyServices(
        EntityFrameworkServicesBuilder builder)
        => builder.AddSqlite();
}
```

When EF needs to register the current provider's services it calls ApplyServices, which in turn calls the AddXxx extension method that was defined above.

<h2>IDataStoreSource</h2>

The final piece of the puzzle that connects selection of a provider with the services used for that provider is an implementation of the IDataStoreSource interface. A typical implementation uses the DataStoreSource generic base class, passing in the types for the provider's IDataStoreServices and IDbContextOptionsExtension implementations. For example, SqliteDataStoreSource looks something like:

``` c#
public class SqliteDataStoreSource 
    : DataStoreSource<SqliteDataStoreServices, SqliteOptionsExtension>
{
    public override void AutoConfigure(DbContextOptionsBuilder optionsBuilder)
    {
    }

    public override string Name => "SQLite Data Store";
}
```

SqliteDataStoreSource is then registered in the AddSqlite extension method as shown above.

All of this means that when EF finds that a SqliteOptionsExtension has been added to the options builder through a call to UseSqlite, then SqliteDataStoreServices will be used to resolve services for the scope of that context instance.

<h2>Summary</h2>

The steps for creating an EF7 provider are:

<ul>
<li>Implement provider-specific services, using IDataStoreServices and IRelationalDataStoreServices as a guide for the services needed</li>
<li>Create an implementation of IDataStoreServices or IRelationalDataStoreServices to map your implementations, using one of the base classes as a starting point</li>
<li>Create an AddXxx extension method to register your services in DI</li>
<li>Create an IDbContextOptionsExtension implementation to handle provider selection and configuration</li>
<li>Create a UseXxx extension method to allow applications to select your provider</li>
<li>Create an IDataStoreSource implementation to tie everything together</li>
</ul>

Of course, the hard part about creating any provider is actually implementing the services. Also, you will likely want to create some extension methods for provider-specific model building and/or runtime functionality. If there is enough interest I may blog about these things at a later date.
