---
layout: default
title: "EF Core 1.1 metadata overview"
date: 2016-11-09 15:50
day: 9th
month: November
year: 2016
author: ajcvickers
permalink: 2016/11/09/ef-core-1-1-metadata-overview/
---

# EF Core 1.1
# Metadata overview

This post provides a high-level overview of metadata structure and APIs in EF Core 1.1. It covers the core metadata interfaces and shows how they are extended with particular focus on provider extensions.



<h2>Core metadata</h2>

<h3>Read-only interfaces</h3>

At the lowest level, EF Core model metadata is defined by a set of read-only interfaces. These are:

<ul>
<li>IModel - The model, which contains entity types</li>
<li>IEntityType - An entity type, which maps to a .NET class</li>
<li>ITypeBase - The base for types, of which only entity types are currently implemented</li>
<li>IProperty - A scalar/simple property of an entity type</li>
<li>INavigation - A navigation property from one entity type to another</li>
<li>IPropertyBase - The base for properties and navigation properties</li>
<li>IForeignKey - A relationship between entity types defined by shared property values</li>
<li>IKey - A unique key (primary or alternate) that gives an entity type identity</li>
<li>IIndex - An index over properties of an entity type</li>
</ul>

<h3>Extension methods</h3>

These interfaces have a relatively minimal API surface which is supplemented with extension methods. For example, IModel defines two members:

<ul>
<li>IEnumerable<IEntityType> GetEntityTypes()</li>
<li>IEntityType FindEntityType(string name)</li>
</ul>

But also has three extension methods:

<ul>
<li>IEntityType FindEntityType(this IModel model, Type type)</li>
<li>ChangeTrackingStrategy GetChangeTrackingStrategy(this IModel model)</li>
<li>PropertyAccessMode? GetPropertyAccessMode(this IModel model)</li>
</ul>

Extension methods are the primary way that the basic model is given richness. This includes integration with provider-specific functionality, as described below.

<h3>Annotations</h3>

All metadata items can be annotated with key/value pairs. This is represented by the IAnnotable interface, which is inherited by all the interfaces listed above.

Annotations are the primary means by which a model is extended with functionality that is not strictly part of main metadata model. This includes provider extensions.

It is rare for application code to access annotations directly. Instead annotations are exposed through extension methods, such as the GetPropertyAccessMode method listed above. The annotations themselves are considered implementation details and are much more likely to change than the extension methods, which are part of the public API surface.

<h3>Mutable metadata</h3>

The interfaces and extension methods described above are read-only. This is the API surface that must be used for most application code because once the model is built it is considered immutable.

However, the model elements can be changed while it is being built. Therefore, during model building, the model is exposed through a mutable set of interfaces that inherit from the interfaces above. These are:

<ul>
<li>IMutableModel</li>
<li>IMutableEntityType</li>
<li>IMutableTypeBase</li>
<li>IMutableProperty</li>
<li>IMutableNavigation</li>
<li>IMutablePropertyBase</li>
<li>IMutableForeignKey</li>
<li>IMutableKey</li>
<li>IMutableIndex</li>
</ul>

There is also a mutable annotation interface, IMutableAnnotatable, which is used to create, modify, and remove annotations.

These interfaces are minimal and supplemented with extension methods in the same way as the read-only interfaces. Annotations should be modified through these extension methods rather than directly.

<h2>Provider extensions</h2>

Providers extend the core metadata through annotations. These annotations are accessed through extension methods that follow a common pattern. For example, access to the flag that indicates whether a SQL Server table is memory-optimized looks like this:

``` c#
var isMemOptimized = entityType.SqlServer().IsMemoryOptimized;
```

Notice that there is an extension method <code>SqlServer()</code> that matches the provider name. This method then provides access to all SQL Server extensions of the entity type. There are also <code>SqlServer()</code> extension methods for properties, keys, etc. Other providers have similar methods--for example, <code>Sqlite()</code>.

We will look at how a provider implements these extensions in a later post.

<h3>Relational extensions</h3>

There are a set of extensions that are common to all relational providers. This allows things like table names, column names, etc. to be set for any relational provider making the model less tightly coupled to a specific provider.

These can be accessed through the "Relational" extension method. For example:

``` c#
var tableName = entityType.Relational().TableName;
```

Relational annotations like this can be overridden by provider-specific annotations. This allows, for example, a table name to be set for all relational providers but then overridden to a different name on SQL Server. For this reason all relational annotations can also be read using provider-specific extensions. For example:

``` c#
var tableName = entityType.SqlServer().TableName;
```

This code will return the SQL Server specific table name, if one has been set, otherwise the relational table name, if it has been set, otherwise the default table name created by convention.

Again, we will look at how this is implemented in a later post.

<h3>Mutable provider extensions</h3>

The provider extension methods work the same way when using the mutable metadata interfaces. For example, to set the table name for any relational provider:

``` c#
entityType.Relational().TableName = "BlogsTable";
```

Or to set the table name specifically for SQL Server:

``` c#
entityType.SqlServer().TableName = "BlogsTable";
```

If only one provider is being used, then both of these do effectively the same thing.

<h2>Fluent API</h2>

This post does not cover the fluent API exposed by the ModelBuilder class. The fluent API is the primary means by which the model should be configured and is documented in the main EF Core docs.

The fluent API is built on top of the core interfaces and extensions described above. The mutable interfaces are exposed from the fluent API through Metadata properties. For example, an IMutableEntityType can be obtained like this:

``` c#
var entityType = modelBuilder.Entity<Blog>().Metadata;
```

<h3>Fluent API extensions</h3>

Providers can also extend the fluent API using extension methods. In this case, the relational-level extension methods are named naturally and do not go through a level of indirection. For example:

``` c#
modelBuilder.Entity<Blog>().ToTable("BlogsTable");
```

This is equivalent to setting <code>Relational().TableName</code>.

The fluent API for setting the table name for only SQL Server is:

``` c#
modelBuilder.Entity<Blog>().ForSqlServerToTable("BlogsTable");
```

This is equivalent to setting <code>SqlServer().TableName</code>.

There is no need to use the "ForSqlServer" versions of the methods unless you are using multiple providers and need to set the table name specifically for SQL Server. Most code should just use <code>TableName</code>.

<h2>Summary</h2>

EF Core metadata is represented by a small set of interfaces that provide read-only access to the model. These are extended at model building time to allow the model to be mutated. Metadata is extended through annotations, but these should not be accessed directly. Instead a rich set of extension methods is provided for accessing the extended model. The fluent API builds on top of these metadata interfaces and extensions and is the primary means for building and configuring the model.
