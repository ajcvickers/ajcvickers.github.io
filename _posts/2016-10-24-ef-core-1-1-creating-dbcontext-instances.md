---
layout: default
title: "EF Core 1.1 - Creating DbContext instances"
date: 2016-10-24 13:40
day: 24th
month: October
year: 2016
author: ajcvickers
permalink: 2016/10/24/ef-core-1-1-creating-dbcontext-instances/
---

# EF Core 1.1
# Creating DbContext instances

This post describes the different ways to create and configure instances of DbContext in EF Core 1.1. This includes:

<ul>
<li>Calling a constructor directly and overriding OnConfiguring</li>
<li>Passing DbContextOptions to the constructor</li>
<li>Using Dependency Injection (D.I.) to create instances</li>
</ul>



<h2>Explicitly creating DbContext instances</h2>

There is no requirement to use EF with Dependency Injection (D.I.). New instances can be created with <code>new</code> in the normal C# way.

<h3>Constructor with OnConfiguring</h3>

The simplest way to create a context instance is to create a class derived from DbContext and call its parameterless constructor. Don't forget that context instances should always be disposed, so often the instance is creating within a using statement. For example:

``` c#
using (var context = new MyContext())
{
}
```

Configuration of the context instance is done by overriding OnConfiguring. For example:

``` c#
public class MyContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) 
        => optionsBuilder.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFTest;Trusted_Connection=True;");
}
```

Note that OnConfiguring is called every time a new context instance is initialized; this is different from OnModelCreating which is usually only called once to build the model. This means that OnConfiguring can make use of arguments passed to the context constructor or other instance data, as shown in the next section.

<h3>Passing parameters to the constructor</h3>

The EF6 DbContext had many constructors taking things like the connections string, model, etc. It is easy to use a similar pattern with EF Core by stashing those things in fields and using them in OnConfiguring.

``` c#
public class MyContext : DbContext
{
    private readonly string _connectionString;

    public MyContext(string connectionString)
    {
        _connectionString = connectionString;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder.UseSqlServer(_connectionString);
}
```

<h3>Using DbContextOptions without D.I.</h3>

The other constructor exposed by DbContext takes a DbContextOptions object. This is often used when creating DbContext instances from a D.I. container, as explained below. However, it can also be used explicitly. For example, an application can create a DbContextOptions object separately from the context, stash it away somewhere, and then use that same options object for every context instance created.

DbContextOptions objects are created using a builder. This is the same builder that is passed to OnConfiguring, so all the configuration methods are the same. Builder methods are designed to be chained in a fluent-like manner. The Options property on the builder returns the built options.

For example, a console app that takes a connection string as its first argument might do something like this:

``` c#
public class Program
{
    private static DbContextOptions _contextOptions;

    public static void Main(string[] args)
    {
        _contextOptions = new DbContextOptionsBuilder()
            .UseSqlServer(args[0])
            .Options;

        // My app stuff...
    }
}
```

Now the class derived from DbContext can take the options and pass them to the base constructor:

``` c#
public class MyContext : DbContext
{
    public MyContext(DbContextOptions options)
        : base(options)
    {
    }
}
```

There is no longer any need to override OnConfiguring. However, OnConfiguring can still be overridden and will still be called. This means that configuration can be passed to the constructor and then tweaked in OnConfiguring.

<h2>Using Dependency Injection to create instances</h2>

If the context type is registered in a D.I. container, then it can be resolved from that container in the normal way.

Two things to note when using D.I. to resolve a DbContext are:

<ul>
<li>The context instance must be disposed after it has been used. Your D.I. container might do this automatically, especially if you use a scope. However, depending on your container, it may not happen if your context is registered as a transient.</li>
<li>DbContext is not thread-safe. This means that a context must not be registered as a singleton and then used concurrently without additional locking. Also, some containers may allow a scoped service to be obtained without first creating a scope, in which case it acts like a singleton and will have the same issues.</li>
</ul>

<h3>D.I. with a parameterless constructor</h3>

A derived DbContext type can have a parameterless constructor and override OnConfiguring, just as described above. In this case the context class does not have any dependencies and so nothing else needs to be registered in the container other than the context type itself.

<h3>D.I. with DbContextOptions</h3>

It may be useful to register a DbContextOptions instance in D.I. This can often be created once and registered as a singleton. For example, using the <code>Microsoft.Extensions.DependencyInjection</code> abstractions:

``` c#
public class Program
{
    private static IServiceProvider _serviceProvider;

    public static void Main(string[] args)
    {
        var contextOptions = new DbContextOptionsBuilder()
            .UseSqlServer(args[0])
            .Options;

        var services = new ServiceCollection()
            .AddSingleton(contextOptions)
            .AddScoped<MyContext>();

        _serviceProvider = services.BuildServiceProvider();

        // My app stuff...
    }
}
```

Now whenever MyContext is resolved from the container the DbContextOptions instance will be injected into its constructor.

<h3>Generic DbContextOptions</h3>

So far all DbContextOptions objects have been non-generic. There is also a version that is generic on the type of the DbContext class. This is useful if there are multiple different DbContext types registered in D.I. by allowing each context type to depend on its own options. For example:

``` c#
public class MyContext1 : DbContext
{
    public MyContext1(DbContextOptions<MyContext1> options)
        : base(options)
    {
    }
}

public class MyContext2 : DbContext
{
    public MyContext2(DbContextOptions<MyContext2> options)
        : base(options)
    {
    }
}

var contextOptions1 = new DbContextOptionsBuilder<MyContext1>()
    .UseSqlServer(connectionString1)
    .Options;

var contextOptions2 = new DbContextOptionsBuilder<MyContext2>()
    .UseSqlServer(connectionString2)
    .Options;

var services = new ServiceCollection()
    .AddSingleton(contextOptions1)
    .AddScoped<MyContext1>()
    .AddSingleton(contextOptions2)
    .AddScoped<MyContext2>();

_serviceProvider = services.BuildServiceProvider();
```

Resolving MyContext1 will result in DbContextOptions<MyContext1> being injected, while resolving MyContext2 will result in DbContextOptions<MyContext2> being injected. There is nothing magical going on here--it's just D.I. doing appropriate dependency resolution.

<h3>Using AddDbContext</h3>

EF includes the AddDbContext sugar method that makes it easier to register a DbContextOptions and a DbContext when using the <code>Microsoft.Extensions.DependencyInjection</code> abstractions. For example:

``` c#
var services = new ServiceCollection()
     .AddDbContext<MyContext>(
         b => b.UseSqlServer(connectionString));
```

This registers MyContext as scoped and registers DbContextOptions<MyContext> as a singleton built from the code provided in the delegate. The code in the delegate again uses the same DbContextOptionsBuilder as is used in OnConfiguring. It is worth noting that the delegate is not executed immediately, but rather just before the first time DbContextOptions<MyContext> is resolved from D.I.

The DbContext type is registered as scoped by default. This can be changed by passing an argument to AddDbContext, but be aware of the notes at the start of this section if adding as a transient or singleton service.

AddDbContext doesn't do much more than register the context and options. It does also ensure that if your service provider contains Logging and/or MemoryCache services, then these will be used by EF, but there is nothing in AddDbContext that can't be done by any other mechanism for configuring your D.I. container.

<h2>Summary</h2>

Configuration of a DbContext is done using DbContextOptionsBuilder to build DbContextOptions. This can be done by overriding OnConfiguring or by building the options outside and passing them to a DbContext constructor, or by a combination of both these things. DbContext instances are usually created either with the parameterless constructor or with a constructor taking DbContextOptions. This is the same regardless of whether the instance is created explicitly or if D.I. is used.
