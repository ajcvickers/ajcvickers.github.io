---
layout: post
title: EF Core Dependency Injection Internals
date: 2016-11-01 12:46
author: ajcvickers
comments: true
categories: [Dependency Injection, EF Core, EF Core Provider, Entity Framework]
---
A <a href="https://blog.oneunicorn.com/2016/10/27/dependency-injection-in-ef-core-1-1/">previous post</a> gave an overview of how dependency injection is used internally by EF Core, and how applications might interact with this. In this post we will look at some of the internal details. This post is aimed at provider writers and people who may want to contribute to the EF source code. Application developers should not need to know any of this.

<!--more-->

<h2>Principles</h2>

Two principles are used throughout EF's internal services:

<ul>
<li>All services are defined by interfaces. Services are not defined by classes. This means that when implementing a class that uses other services it should depend on only interfaces in its constructor.</li>
<li>Whenever possible, dependencies of one service on another should be made explicit using constructor injection.</li>
</ul>

When used together these principles make it easy to see the dependencies in the system. It ensures that services only make use of the public API service defined by other services and do not depend on any implementation details.

(There are some cases where constructor injection is not possible, usually when dependencies are conditional. For example, the QueryContextFactory depends on IStateManager for tracking entities. However, if the query is a no-tracking query, then the state manager is not needed. Therefore, as a perf optimization, the QueryContextFactory does not depend on IStateManager directly, but instead depends on ICurrentDbContext. This in turns allows IStateManager to be loaded if it is needed by using ICurrentDbContext as a service locator.)

<h2>Scopes</h2>

EF Core creates a scope for each context instance such that there is essentially a scope for each session. This scope has two purposes:

<ul>
<li>It allows services that should work together within the session to depend on each other in a natural way. For example, in a given scope (session) there is one IStateManager, one INavigationFixer, one IChangeDetector, and so on. These all work together with the currently tracked entities. These services are released/disposed at the end of the session leaving the system clean for the next session.</li>
<li>It allows different services to be resolved for different context instances. This is most important for resolving the correct services for the current session's database provider.</li>
</ul>

<h2>Service registration and lifetimes</h2>

Registration of core EF services is done in an IServiceCollection extension method called AddEntityFramework. A very trimmed down version of this method looks like this:

[code lang=csharp]
public static IServiceCollection AddEntityFramework(
    [NotNull] this IServiceCollection serviceCollection)
{
    serviceCollection.TryAddEnumerable(new ServiceCollection()
        .AddScoped&lt;IEntityStateListener, INavigationFixer&gt;(p =&gt; p.GetService&lt;INavigationFixer&gt;())
        .AddScoped&lt;INavigationListener, INavigationFixer&gt;(p =&gt; p.GetService&lt;INavigationFixer&gt;())
        .AddScoped&lt;IEntityStateListener, ILocalViewListener&gt;(p =&gt; p.GetService&lt;ILocalViewListener&gt;()));

    serviceCollection.TryAdd(new ServiceCollection()
        .AddSingleton&lt;IDbSetFinder, DbSetFinder&gt;()
        .AddScoped&lt;INavigationFixer, NavigationFixer&gt;()
        .AddScoped&lt;ValueGeneratorSelector&gt;()
        .AddScoped&lt;IModel&gt;(p =&gt; p.GetRequiredService&lt;IDbContextServices&gt;().Model)
        .AddScoped&lt;IValueGeneratorSelector&gt;(p =&gt; p.GetRequiredService&lt;IDbContextServices&gt;().DatabaseProviderServices.ValueGeneratorSelector)
        .AddScoped&lt;IValueGeneratorCache&gt;(p =&gt; p.GetRequiredService&lt;IDbContextServices&gt;().DatabaseProviderServices.ValueGeneratorCache));

    return serviceCollection;
}
[/code]

Most of the service registrations have been removed to leave just enough to explain the concepts going on here:

<ul>
<li>All services are registered using TryAdd methods. This means that calls to override services can be made either before or after calling AddEntityFramework. It also means that AddEntityFramework is idempotent and can be called multiple times.</li>
<li>Services that require a new instance per session are registered as scoped. For example, INavigationFixer.

<ul>
<li>Services that depend on scoped services must also be registered as scoped.</li>
</ul></li>
<li>Services that provide a root for data that should be cached across context instances must be registered as singletons. For example, DbSetFinder.

<ul>
<li>Singleton services must be thread-safe; this is why DbSetFinder uses a ConcurrentDictionary.</li>
</ul></li>
<li>Some services are registered as delegates so that:

<ul>
<li>The same service instance can be returned for more than one interface. For example, NavigationFixer is both an IEntityStateListener and an INavigationListener. The same instance of NavigationFixer must be returned regardless of whether it is requested as as an IEntityStateListener or an INavigationListener.</li>
<li>The service instance can come from something other than D.I. registration. For example, IModel is usually created and cached when the context is used for the first time. It can also be set in OnConfiguring or on DbContextOptions. This is all handled by the implementation of IDbContextServices which then exposes the model as property. All other code just depends on IModel through constructor injection and doesn't have to know anything about how the model was obtained.</li>
<li>The service is a database provider service--see below.</li>
</ul></li>
</ul>

<h3>Database provider services</h3>

Database providers interact with EF Core by implementing various services. These services are registered in D.I., as shown later, and are then used through constructor injection in the normal way. Multiple providers can register their services in the same D.I. container. The context is then responsible for determining which provider is in use for the current session and ensuring that the services for that provider are resolved from D.I.

For example, see the registration for IValueGeneratorSelector in the code above. This service is resolved as follows:

<ul>
<li>IDbContextServices is resolved. This is the bridge to services that can change from session to session, as described for IModel in the previous section.</li>
<li>IDatabaseProviderServices is obtained from IDbContextServices. Each database provider ships with an implementation of IDatabaseProviderServices. The one returned is selected by the context based on which provider is in use for the current session. </li>
<li>The actual IValueGeneratorSelector is returned from IDatabaseProviderServices--see below for more details.</li>
</ul>

<h3>Registering provider services</h3>

Database providers should ship with a method like AddEntityFramework. For example, the in-memory provider has a method called AddEntityFrameworkInMemoryDatabase. A cut-down version of this method looks like this:

[code lang=csharp]
public static IServiceCollection AddEntityFrameworkInMemoryDatabase(
    [NotNull] this IServiceCollection services)
{
    services.AddEntityFramework();

    services.TryAddEnumerable(ServiceDescriptor
        .Singleton&lt;IDatabaseProvider, DatabaseProvider&lt;InMemoryDatabaseProviderServices, InMemoryOptionsExtension&gt;&gt;());

    services.TryAdd(new ServiceCollection()
        .AddSingleton&lt;InMemoryValueGeneratorCache&gt;()
        .AddSingleton&lt;IInMemoryTableFactory, InMemoryTableFactory&gt;()
        .AddScoped&lt;InMemoryValueGeneratorSelector&gt;());

    return services;
}
[/code]

Things to notice:

<ul>
<li>AddEntityFrameworkInMemoryDatabasemethod calls AddEntityFramework. This means that application developers just need to call the 'Add...' method that ships with the provider and it will ensure that all core services for EF are added. It doesn't matter if AddEntityFramework gets call multiple times because it is idempotent.</li>
<li>An IDatabaseProvider for the in-memory provider is registered. This will be covered in more detail in a future post.</li>
<li>Provider services are registered. These can be of two forms:

<ul>
<li>Services that are defined entirely by the provider and are not implementations of EF services. For example, IInMemoryTableFactory.</li>
<li>Implementations of EF services. For example, InMemoryValueGeneratorSelector.</li>
</ul></li>
</ul>

<h3>IDatabaseProviderServices implementation</h3>

The IDatabaseProviderServices implementation ties all this together. Here is a cut-down version of the in-memory implementation:

[code lang=csharp]
public class InMemoryDatabaseProviderServices : DatabaseProviderServices
{
    public InMemoryDatabaseProviderServices([NotNull] IServiceProvider services)
        : base(services)
    {
    }

    public override IValueGeneratorSelector ValueGeneratorSelector 
        =&gt; GetService&lt;InMemoryValueGeneratorSelector&gt;();

    public override IValueGeneratorCache ValueGeneratorCache 
        =&gt; GetService&lt;InMemoryValueGeneratorCache&gt;();
}
[/code]

The ValueGeneratorSelector property is overridden to call GetService for InMemoryValueGeneratorSelector. This completes the story of how IValueGeneratorSelector is resolved end-to-end:

<ul>
<li>IDbContextServices is resolved.</li>
<li>The context has determined that the in-memory provider is in use and so returns the InMemoryDatabaseProviderServices implementation. </li>
<li>The ValueGeneratorSelector property of InMemoryDatabaseProviderServices is called.</li>
<li>InMemoryValueGeneratorSelector is resolved, which was registered in AddEntityFrameworkInMemoryDatabase.</li>
</ul>

Also notice that InMemoryDatabaseProviderServices extends from DatabaseProviderServices. This class provides implementations for some services. For example, a basic ValueGeneratorSelector implementation was registered in AddEntityFramework. This is returned by the ValueGeneratorSelector property of the DatabaseProviderServices base class. So if a provider doesn't need to provide its own implementation, then it doesn't need to register anything or override anything and it will get the basic implementation shipped with EF.

<h3>Registering concrete instances</h3>

At the very top of this post it was stated that all EF services are defined by interfaces. Why, then, are some concrete instances registered in D.I.? The answer is that when EF code resolves the interface for a provider service that service is obtained from D.I. by a call to GetService for the concrete implementation. For example, ValueGeneratorSelector or InMemoryValueGeneratorSelector, which are resolved by calls to GetService in DatabaseProviderServices or InMemoryDatabaseProviderServices respectively. Other services should not depend on the concrete implementations.

<h3>Singleton verses scoped provider services</h3>

All provider services are registered as scoped. This is because a different instance may be returned depending on which provider is being used for the current session. However, some provider services must also act as a cache root, which means that the service must be registered as a singleton. This is done by registering the concrete implementation as a singleton even though the service interface is registered as scoped.

For example, InMemoryValueGeneratorCache is registered as a singleton. This means there will be only one of these caches for all context instances. However, IValueGeneratorCache is registered in AddEntityFramework as scoped. This means that for any session where the in-memory provider is in use, the singleton InMemoryValueGeneratorCache will be used. But if a different provider is in use, then resolution will go through a different IDatabaseProviderServices, which will return a different singleton.

<h2>Relational providers</h2>

All of the examples above used the in-memory provider as an example. Relational providers are exactly the same except that there is an interface IRelationalDatabaseProviderServices that extends from IDatabaseProviderServices and adds relational-specific services. There is a base class implementation of this which the provider implementation should extend. Finally, there in an AddEntityFrameworkRelational method that should be called in the provider's 'Add...' method to ensure relational services are registered in addition to core services.

<h2>Summary</h2>

The internal use of D.I. by EF Core makes use of a variety of mechanisms to ensure that services can depend on other services in a natural way while still allowing services to be resolved in special ways. EF creates a new service scope per session and some services are resolved dynamically within that scope. This allows the correct provider services to be resolved depending on the provider in use for the current session.

Future posts will cover the other things providers need to do to get all their services working together with EF Core.
