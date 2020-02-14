---
layout: post
title: Implementing a provider "Use..." method for EF Core 1.1
date: 2016-11-03 15:24
author: ajcvickers
comments: true
categories: [DbContextOptionsBuilder, EF Core, EF Core Provider, Entity Framework]
---
The previous post contained lots of information about how dependency injection works with database providers. This post adds more to the provider story by explaining how to implement a method like UseSqlServer that allows applications to select the provider to use.

<!--more-->

<h2>IDatabaseProvider</h2>

The previous post showed how database providers implement IDatabaseProviderServices to define provider-specific services. The post also covered how services are registered with an "Add..." method. One of the services registered must be for IDatabaseProvider. This service is registered using TryAddEnumerable, which means that if three providers have been registered, then there will be three registrations for IDatabaseProvider in the container.

The context decides which provider is in use by calling IsConfigured on each registered IDatabaseProvider, passing in the current DbContextOptions. (Note that an exception is thrown if anything other than exactly one IDatabaseProvider returns true for IsConfigured. This means an exception is thrown if an application makes calls to multiple "Use.." for the same context instance.)

Once an IDatabaseProvider has been selected, its GetProviderServices method is called to get the IDatabaseProviderServices for the provider.

The result of all this is that IDatabaseProvider is the mechanism by which a provider is chosen from the current DbContextOptions.

<h2>IDbContextOptionsExtension</h2>

Options extensions are the mechanism whereby providers can add information to the DbContextOptions for the context. An options extension is registered by the "Use..." method, as described below. Each provider's extension implements the IDbContextOptionsExtension. EF will then call the ApplyServices method to register provider services in the D.I. container.

For example, a trimmed down version of the in-memory provider's options extension looks something like this:

[code lang=csharp]
public class InMemoryOptionsExtension : IDbContextOptionsExtension
{
    public InMemoryOptionsExtension()
    {
    }

    public InMemoryOptionsExtension([NotNull] InMemoryOptionsExtension copyFrom)
    {
        StoreName = copyFrom.StoreName;
    }

    public virtual string StoreName { get; set; }

    public virtual void ApplyServices(IServiceCollection services)
        =&gt; services.AddEntityFrameworkInMemoryDatabase();
}
[/code]

Notice that ApplyServices simply calls the AddEntityFrameworkInMemoryDatabase that was described in the previous post.

The options extension also holds additional information needed by the provider. For example, relational options extensions will contain the connection string to use. The in-memory extension shown above holds the name of the in-memory database to use.

<h2>Implementing the "Use..." method</h2>

A trimmed-down version of the UseInMemoryDatabase method looks something like this:

[code lang=csharp]
public static DbContextOptionsBuilder UseInMemoryDatabase(
    this DbContextOptionsBuilder optionsBuilder,
    string databaseName)
{
    var extension = optionsBuilder.Options.FindExtension&lt;InMemoryOptionsExtension&gt;();

    extension = extension != null
        ? new InMemoryOptionsExtension(extension)
        : new InMemoryOptionsExtension();

    extension.StoreName = databaseName;

    ((IDbContextOptionsBuilderInfrastructure)optionsBuilder).AddOrUpdateExtension(extension);

    return optionsBuilder;
}
[/code]

Walking through this code:

<ul>
<li>The first line finds a previously registered InMemoryOptionsExtension. This is to ensure any previously applied options are preserved.</li>
<li>A new extension is created. If an existing extension was found, then the new extension should clone it. It should not mutate the extension because that could cause singleton options registered in D.I. (for example, from AddDbContext) to change.</li>
<li>Data is added to the extension. In this case, the in-memory database name is set.</li>
<li>The extension is added to the options, or if there was an existing extension, then it is replaced with the new one.</li>
<li>The builder is returned to allow method chaining.</li>
</ul>

Notice that after this call, the options will have an InMemoryOptionsExtension registered.

<h2>Coupling IDatabaseProvider and IDbContextOptionsExtension</h2>

As discussed above, the IsConfigured method of IDatabaseProvider must return true if the provider is being used. The previous section showed that calling a "Use..." method will result in an options extension being added to the options. Therefore, IsConfigured should return true if an options extension for the provider has been added to the options.

The DatabaseProvider class implements IDatabaseProvider to do this:

[code lang=csharp]
public class DatabaseProvider&lt;TProviderServices, TOptionsExtension&gt; : IDatabaseProvider
    where TProviderServices : class, IDatabaseProviderServices
    where TOptionsExtension : class, IDbContextOptionsExtension
{
    public virtual IDatabaseProviderServices GetProviderServices(IServiceProvider serviceProvider)
        =&gt; serviceProvider.GetRequiredService&lt;TProviderServices&gt;();

    public virtual bool IsConfigured(IDbContextOptions options)
        =&gt; options.Extensions.OfType&lt;TOptionsExtension&gt;().Any();
}
[/code]

This means that a provider should register a DatabaseProvider class to couple provider selection with the "Use..." method. For example:

[code lang=csharp]
services.TryAddEnumerable(ServiceDescriptor
    .Singleton&lt;IDatabaseProvider, DatabaseProvider&lt;InMemoryDatabaseProviderServices, InMemoryOptionsExtension&gt;&gt;());
[/code]

When an InMemoryOptionsExtension is registered, the InMemoryDatabaseProviderServices will be selected.

<h2>Additional "Use..." method details</h2>

<h3>The generic overload</h3>

A provider should implement an overload of its "Use..." method that works with the generic DbContextOptionsBuilder&lt;TContext&gt;. For example:

[code lang=csharp]
public static DbContextOptionsBuilder&lt;TContext&gt; UseInMemoryDatabase&lt;TContext&gt;(
    this DbContextOptionsBuilder&lt;TContext&gt; optionsBuilder,
    string databaseName)
    where TContext : DbContext
    =&gt; (DbContextOptionsBuilder&lt;TContext&gt;)UseInMemoryDatabase(
        (DbContextOptionsBuilder)optionsBuilder, databaseName);
[/code]

This ensures that any chained methods will still get the generic DbContextOptionsBuilder&lt;TContext&gt;.

<h3>Nested provider-specific configuration</h3>

Configuration fundamental to the use of the provider should be passed as arguments to the "Use..." method. For example, the connection string. Other provider-specific configuration can be done in a nested closure. For example:

[code lang=csharp]
optionsBuilder.UseSqlServer(
    _connectionString,
    b =&gt; b.CommandTimeout(10));
[/code]

Here, CommandTimeout is defined on SqlServerDbContextOptionsBuilder. The relevant parts of UseSqlServer look something like this:

[code lang=csharp]
public static DbContextOptionsBuilder UseSqlServer(
    this DbContextOptionsBuilder optionsBuilder,
    string connectionString,
    Action&lt;SqlServerDbContextOptionsBuilder&gt; sqlServerOptionsAction = null)
{
    // Usual options extension stuff...

    sqlServerOptionsAction?.Invoke(new SqlServerDbContextOptionsBuilder(optionsBuilder));

    return optionsBuilder;
}
[/code]

Notice that the delegate has a default of null since often code does not need to set any additional options. The CommandTimeout method looks very similar to the "Use..." method:

[code lang=csharp]
public virtual SqlServerDbContextOptionsBuilder CommandTimeout(int? commandTimeout)
{
    var extension = new SqlServerOptionsExtension(
        OptionsBuilder.Options.GetExtension&lt;SqlServerOptionsExtension&gt;());

    extension.CommandTimeout = commandTimeout;

    ((IDbContextOptionsBuilderInfrastructure)OptionsBuilder).AddOrUpdateExtension(extension);

    return this;
}
[/code]

This method works just as is described above for the "Use..." method:

<ul>
<li>The extension is cloned. (In this case it is known to exist.)</li>
<li>The option is set.</li>
<li>The extension is updated on the options.</li>
<li>The builder is returned for further chaining.</li>
</ul>

<h2>Summary</h2>

The "Use.." method configures a provider for use by adding an options extension to the DbContextOptions. The context uses this in provider selection to determine which IDatabaseProviderServices to return.
