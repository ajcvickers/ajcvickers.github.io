---
layout: post
title: Collection navigation properties and fields in EF Core 1.1
date: 2016-10-28 10:37
author: ajcvickers
comments: true
categories: [Backing fields, Code First, EF Core, Entity Framework]
---
There has recently been some confusion about what mappings are supported for collection navigation properties in EF Core. This post is an attempt to clear things up by showing:

<ul>
<li>What types of collection are supported</li>
<li>When the backing field can be used directly</li>
</ul>

<!--more-->

<h2>Mapping to auto-properties</h2>

Perhaps the most common way to define a collection navigation property is to use a simple ICollection&lt;T&gt; property:

[code lang=csharp]
public class Blog
{
    public int Id { get; set; }

    public ICollection&lt;Post&gt; Posts { get; set; }
}
[/code]

EF will create the collection for you, usually as a HashSet&lt;T&gt; when using Include and during fixup, or the application can set it to any ICollection&lt;T&gt; that allows objects to be added.

EF Core 1.1 also allows a collection to be exposed as IEnumerbale&lt;T&gt;. For example:

[code lang=csharp]
public class Blog
{
    public int Id { get; set; }

    public IEnumerable&lt;Post&gt; Posts { get; set; }
}
[/code]

This means that there is no public surface for adding entities to the collection. The collection itself still needs to be a proper collection with a working Add method, and EF will still create one for you, or the application can create it itself.

There is also no need for a setter as long as the entity always has a collection created so that EF never needs to do it.

[code lang=csharp]
public class Blog
{
    public int Id { get; set; }

    public IEnumerable&lt;Post&gt; Posts { get; } = new List&lt;Post&gt;();
}
[/code]

<h2>Properties with backing fields</h2>

<h3>Using an Add method</h3>

Taking this one step further, you can expose your IEnumerable&lt;T&gt; navigation property and use a backing field to control other mutations. For example:

[code lang=csharp]
public class Blog
{
    private readonly List&lt;Post&gt; _posts = new List&lt;Post&gt;();

    public int Id { get; set; }

    public IEnumerable&lt;Post&gt; Posts =&gt; _posts;

    public void AddPost(Post post)
    {
        // Do some buisness logic here...
        _posts.Add(post);
    }
}
[/code]

<h3>Using a defensive copy</h3>

In all of the examples above the Posts navigation property returns the actual collection. This means that application code could add entities directly by casting to an ICollection&lt;T&gt;. This can be fixed by making a defensive copy, but this requires that EF always read and write directly to the field because adding entities to the defensive copy obviously won't work. There is currently no way to do this with the fluent API, but it is still easy to do by dropping to core metadata. For example:

[code lang=csharp]
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    var navigation = modelBuilder.Entity&lt;Blog&gt;().Metadata.FindNavigation(nameof(Blog.Posts));

    navigation.SetPropertyAccessMode(PropertyAccessMode.Field);
}
[/code]

Adding fluent API for this is being tracked by <a href="https://github.com/aspnet/EntityFramework/issues/6674">issue 6674</a>.

Once we tell EF to always use the field we can get something like this:

[code lang=csharp]
public class Blog
{
    private readonly List&lt;Post&gt; _posts = new List&lt;Post&gt;();

    public int Id { get; set; }

    public IEnumerable&lt;Post&gt; Posts =&gt; _posts.ToList();

    public void AddPost(Post post)
    {
        // Do some buisness logic here...
        _posts.Add(post);
    }
}
[/code]

In this example:

<ul>
<li>EF accesses the navigation property directly through the field</li>
<li>Mutation of the collection is controlled by

<ul>
<li>Creating a defensive copy</li>
<li>Using an Add method with business logic</li>
</ul></li>
</ul>

<h2>Summary</h2>

EF Core allows navigation properties to be defined in the traditional way. It also allows navigation properties to be exposed as IEnumerable&lt;T&gt;. Finally, EF Core can make use of backing fields, which allows for full encapsulation of the collection.
