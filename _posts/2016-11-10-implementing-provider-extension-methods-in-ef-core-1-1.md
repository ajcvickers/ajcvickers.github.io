---
layout: default
title: "Implementing provider extension methods in EF Core 1.1"
date: 2016-11-10 15:49
day: 10th
month: November
year: 2016
author: ajcvickers
permalink: 2016/11/10/implementing-provider-extension-methods-in-ef-core-1-1/
---

# EF Core 1.1
# Implementing provider extension methods

A <a href="/2016/11/09/ef-core-1-1-metadata-overview/">previous post</a> gave an outline of EF Core metadata. That post showed the extension methods used by providers to add provider-specific functionality to EF. This post describes how to implement those methods. This post is aimed at provider writers or those who may want to contribute to the EF Core source code.



<h2>Annotations</h2>

All provider-specific functionality is implemented through annotations applied to the various elements of the model. These annotations are expected to follow a standard naming convention of <provider_name>:<annotation name>. For example, the SQL Server memory-optimized annotation name is <code>SqlServer:MemoryOptimized</code>.

These are often defined as constants in the code. For example:

``` c#
public static class SqlServerAnnotationNames
{
    public const string Prefix = "SqlServer:";
    public const string Clustered = "Clustered";
    public const string MemoryOptimized = "MemoryOptimized";
    // More...
}
```

<h3>Relational annotations</h3>

There are a set of annotations that are common to all relational providers. These are defined in the RelationalAnnotationNames class:

``` c#
public static class RelationalAnnotationNames
{
    public const string Prefix = "Relational:";
    public const string TableName = "TableName";
    public const string Schema = "Schema";
    // More...
}
```

This is an internal class and provider code should not need to access it directly, as we will see below.

Relational annotations can be used in two ways:

<ul>
<li>With the <code>Relational:</code> prefix, meaning that the annotation applies to all relational providers. For example, <code>Relational:TableName</code>.</li>
<li>With the provider prefix, meaning that the annotation applies to only that provider. For example, <code>SqlServer:TableName</code>.</li>
</ul>

This allows, for example, the table name to be set for all providers but then overridden to something else just for one provider, as was described in the <a href="/2016/11/09/ef-core-1-1-metadata-overview/">metadata overview post</a>.

<h3>The FullAnnotationNames class</h3>

It was found that the repeated concatenation of prefix and annotation name could cause perf issues. Therefore, each provider should create a FullAnnotationNames class where the concatenation can be done only once. For relational providers, this should inherit from RelationalFullAnnotationNames. For example:

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

Notice that this class is a singleton, with the single instance available via the <code>Instance</code> property. This will be used in the code below.

<h2>Core metadata extensions</h2>

As was discussed in the <a href="/2016/11/09/ef-core-1-1-metadata-overview/">metadata overview post</a>, use of the annotations described above is hidden behind provider extension methods, such as the <code>SqlServer()</code> method. These methods can be implemented for the entity types, properties, etc. as required. Here is the <code>SqlServer()</code> method for extending entity types:

``` c#
public static SqlServerEntityTypeAnnotations SqlServer(this IMutableEntityType entityType)
    => (SqlServerEntityTypeAnnotations)SqlServer((IEntityType)entityType);

public static ISqlServerEntityTypeAnnotations SqlServer(this IEntityType entityType)
    => new SqlServerEntityTypeAnnotations(entityType);
```

It's actually two methods: one for the read-only IEntityType, and one for read-write IMutableEntityType. They both return the same object, but the read-only version is only exposed as an immutable interface.

<h3>The IXxxTypeAnnotations interface</h3>

The ISqlServerEntityTypeAnnotations interface returned looks something like this:

``` c#
public interface ISqlServerEntityTypeAnnotations : IRelationalEntityTypeAnnotations
{
    bool IsMemoryOptimized { get; }
}
```

Pretty simple. Notice that since SQL Server is a relational provider this interface inherits from IRelationalEntityTypeAnnotations, which has all the common relational extensions on it:

``` c#
public interface IRelationalEntityTypeAnnotations
{
    string TableName { get; }
    string Schema { get; }
    // More...
}
```

<h3>The XxxTypeAnnotations class</h3>

The implementation of this interface is also pretty simple:

``` c#
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

Since this is a relational provider the class inherits from RelationalEntityTypeAnnotations. This base class takes care of all the common relational extensions such as TableName and Schema so the provider doesn't have to do anything. The base class uses the SqlServerFullAnnotationNames.Instance passed to the base constructor to determine which annotations to use.

Provider-specific extensions, such as IsMemoryOptimized, simply access annotations on the IEntityType passed to the constructor. Two things to notice about this are:

<ul>
<li>It is normal for the extensions to provide a default if no annotation has been set. So if the annotation for MemoryOptimized is null, then false is returned. Extension methods may provide more complex defaults, such as looking for a value set further up the model if one has not been set on this element. The goal is for application developers to get back a useful value without thinking about where it comes from.</li>
<li>The cast to IMutableAnnotatable will fail if the IEntityType is not mutable. However, notice that IsMemoryOptimized on the interface above does not have a setter, so the only way this can fail is if the interface is cast inappropriately by the application, after which an exception can be expected. (We did at one time have more types here, but given how the API is used we were able to remove a bunch of code just by doing this way with no real loss of usability.)</li>
</ul>

<h2>Fluent API extension methods</h2>

Most application code will configure the model using the fluent API rather than the lower-level core metadata described above. Fortunately, once the core metadata extensions have been implemented it becomes trivial to add fluent API. For example:

``` c#
public static class SqlServerEntityTypeBuilderExtensions
{
    public static EntityTypeBuilder ForSqlServerToTable(
        this EntityTypeBuilder entityTypeBuilder, string name, string schema)
    {
        entityTypeBuilder.Metadata.SqlServer().TableName = name;
        entityTypeBuilder.Metadata.SqlServer().Schema = schema;

        return entityTypeBuilder;
    }

    public static EntityTypeBuilder<TEntity> ForSqlServerToTable<TEntity>(
        this EntityTypeBuilder<TEntity> entityTypeBuilder, string name, string schema)
        where TEntity : class
        => (EntityTypeBuilder<TEntity>)ForSqlServerToTable((EntityTypeBuilder)entityTypeBuilder, name, schema);

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

The things to notice about this code are:

<ul>
<li>Provider-specific (e.g. IsMemoryOptimized) and common relational (e.g. ToTable) are implemented in the same way. There is no base class here.</li>
<li>The fluent API gets the core metadata type from the builder and calls the <code>SqlServer()</code> method to set extension properties</li>
<li>The methods return the passed-in builder to allow chaining</li>
<li>There are generic and non-generic overloads so that genericness is preserved when chaining. The generic overload just calls the non-generic overload and then casts the builder as generic again before returning it.</li>
</ul>

<h2>Summary</h2>

Providers extend metadata through the use of annotations, which should follow a standard naming convention. Providers use extension methods over core metadata so that application code never needs to see the annotations directly. Providers should also ship with fluent API extensions for use in common model building scenarios.
