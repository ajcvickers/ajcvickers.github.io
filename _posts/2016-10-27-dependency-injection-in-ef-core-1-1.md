---
layout: default
title: "Dependency Injection in EF Core 1.1"
date: 2016-10-24 13:40
day: 24th
month: October
year: 2016
author: ajcvickers
permalink: 2016/10/27/dependency-injection-in-ef-core-1-1/
---

# EF Core 1.1
# Dependency Injection in EF Core 1.1

EF Core can interact with dependency injection (D.I.) in two ways:

<ul>
<li>A D.I. container can be used to create DbContext instances</li>
<li>EF uses a D.I. container internally for its own services</li>
</ul>

The first of these was covered in a <a href="/2016/10/24/ef-core-1-1-creating-dbcontext-instances/">previous post</a>. This post covers how EF uses dependency injection internally and how it can interact with an external container.



<h2>The internal service provider</h2>

EF will create a service provider (container) when an application using EF Core is started for the first time. EF Core is built in a service-oriented way, so this service provider is loaded with all the services that EF uses internally. EF uses the default implementation of <code>Microsoft.Extensions.DependencyInjection</code> for this service provider.

For most applications the internal service provider is an implementation detail of EF. Applications should not need to care about it at all.

<h2>Integrating with application services</h2>

An application that is using D.I. may configure services that EF should then use. As of EF 1.1, these services are logging (via ILoggerFactory) and caching (via IMemoryCache). EF provides methods on DbContextOptionsBuilder (see the <a href="/2016/10/24/ef-core-1-1-creating-dbcontext-instances/">previous post</a>) that allow these services to be wired up. For example, the following code takes an ILoggerFactory in its constructor (which may be injected) and wires up the context to use it:

``` c#
public class MyContext : DbContext
{
    private readonly ILoggerFactory _loggerFactory;

    public MyContext(ILoggerFactory loggerFactory)
    {
        _loggerFactory = loggerFactory;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) 
        => optionsBuilder
            .UseInMemoryDatabase()
            .UseLoggerFactory(_loggerFactory);
}
```

Note that there is no need to do this when using AddDbContext because AddDbContext already does this under the covers. This is how EF logging goes to the right place when using AddDbContext in an ASP.NET application.

<h2>Replacing EF services</h2>

Sometimes it may be useful to replace one of EFs internal services. For example, the IValueGeneratorSelector service determines what type of value generator to use for properties that need value generation. You may want to change this to use different types of value generator. Prior to EF Core 1.1 you would need to take control of the internal service provider to do this. Now there is a ReplaceService call on DbContextOptionsBuilder, so you can do something like this:

``` c#
public class MyContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder
            .UseInMemoryDatabase()
            .ReplaceService<IValueGeneratorSelector, CustomInMemoryValueGeneratorSelector>();
}
```

Or in AddDbContext:

``` c#
services
    .AddDbContext<MyContext>(b => b
        .UseInMemoryDatabase()
        .ReplaceService<IValueGeneratorSelector, CustomInMemoryValueGeneratorSelector>());
```

<h2>Taking control of the internal service provider</h2>

Applications should not normally need to interact directly with the internal service provider. External services used by EF are hooked up automatically in AddDbContext, or can be hooked up manually as shown above. Service replacement can be done with ReplaceService, as shown above. However, there are a few scenarios where the application may want to take control of the internal service provider. For example:

<ul>
<li>To use a different D.I. container instead of the one provided with <code>Microsoft.Extensions.DependencyInjection</code></li>
<li>To have EF and the application share the same service provider, perhaps so that EF services can be replaced by services that make use of application dependencies.</li>
<li>To take control of EF caching</li>
</ul>

The UseInternalServiceProvider method on DbContextOptionsBuilder is used to tell EF which service provider to use for its services. This service provider must have all the services configured for EF and any providers. Each provider has an extension method to do this. For example, the following code configures a service provider for use with the in-memory store and then creates options to use it:

``` c#
_serviceProvider = new ServiceCollection()
    .AddEntityFrameworkInMemoryDatabase()
    .BuildServiceProvider();

var contextOptions = new DbContextOptionsBuilder()
    .UseInternalServiceProvider(_serviceProvider)
    .UseInMemoryDatabase()
    .Options;
```

This can also be done with AddDbContext. However, when using AddDbContext the application service provider is often not yet built. Therefore, there is an overload of AddDbContext that takes the application service provider in the builder delegate so it can be passed to UseInternalServiceProvider:

``` c#
services
    .AddEntityFrameworkInMemoryDatabase()
    .AddDbContext<MyContext>((p, b) => b
        .UseInMemoryDatabase()
        .UseInternalServiceProvider(p));
```

It is worth noting that you never need to call any of the AddEntityFramework... methods unless you are calling UseInternalServiceProvider. This is different from pre-release version of EF Core where they were needed.

When taking control of the internal service provider like this it is important to understand how EF uses the service provider for scopes and caching.

<h3>Scopes</h3>

EF creates a scope in the internal service provider for each context instance and disposes that scope when the context instance is disposed. This is normally transparent to applications using EF, but if your application is controlling the service provider and is also using scopes, then there is potential for the two to collide. For example, a second scope should not be created inside EF's scope, and services from the application's scopes should not be resolved by EF's scoped services.

<h3>Caching</h3>

When EF controls the internal service provider it will create one per app domain. (In reality, EF creates a different service provider for each set of service definitions used, but it is rare to have more that one set of service definitions.) This is important because the service provider acts as the root for caching expensive data that should persist across context instances. Therefore, if the application is managing the internal service provider, then it should usually only create one per app domain such that caching is not defeated. An application should not create a new service provider per context instance.

In EF6 and earlier, caching is always done at the app domain level. This made it hard to clear out caching without creating new app domains. This is why the service provider is the root for all caching in EF Core. This means that allowing the service provider to be garbage collected will cause all EF caching to be flushed.

<h2>Summary</h2>

EF Core uses a service provider for its own internal services. This is usually distinct from the application service provider (if one exists) and is transparent to most application code. Internal services can be replaced and common external services can be integrated with EF without taking control of the internal service provider. Applications can take control of the internal service provider for certain specialized scenarios, but must be aware of the caching and scoping requirements.
