---
layout: post
title: Code First Building Blocks
date: 2012-04-21 19:35
author: ajcvickers
comments: true
categories: [Code First, DbModelBuilder, EF4.1, EF4.2, EF4.3, EF5, Entity Framework]
---
There are plenty of examples out there showing how to <a href="http://blogs.msdn.com/b/adonet/archive/2011/09/28/ef-4-2-code-first-walkthrough.aspx">use DbContext to create a Code First model</a>. But DbContext itself uses some fundamental Code First building blocks to do its stuff. This post shows how these building blocks can be used directly, which can be useful in situations where you need more control over how the model is created or cached.<!--more-->
<h3>When would you do this?</h3>
DbContext can be used in the normal way for most applications. However, there are currently two areas for which we get quite a few questions where the best approach is to fall back to using the lower-level building blocks.

The first area is caching of the model. DbContext caches the model based on the type of your context. For example, if you have a context class called <em>BlogContext</em>, then DbContext will build the model once the first time it is used and for subsequent uses the cached model will be used. This doesn’t work if you have one context type but want to use it with different models.

The second area is direct control over the type of database that the model will target. DbContext creates a connection to the database to determine appropriate mappings. For example, if the database is SQL Server 2008, then mappings for types only available in SQL Server 2008 might be used. If you need the model to also work with SQL Server 2005, then you’ll need to tell Code First explicitly to do this.
<h3>It’s a river rock thing…</h3>
The paragraphs above describe concrete situations in which in makes sense to use the Code First building blocks. However, the main reason the building blocks are there is to account for situations we haven’t anticipated. The building blocks provide flexibility to Code First allowing it to be used in new and interesting ways.
<h3>Let’s do it!</h3>
Let’s take this very simple model and context as an example:

[sourcecode language="csharp"]
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public virtual ICollection Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int BlogId { get; set; }
    public virtual Blog Blog { get; set; }
}

public class BlogContext : DbContext
{
    public DbSet&lt;Blog&gt; Blogs { get; set; }
    public DbSet&lt;Post&gt; Posts { get; set; }
}
[/sourcecode]

Normally you would use the context like this:

[sourcecode language="csharp"]
using (var context = new BlogContext())
{
    context.Blogs.Add(new Blog { Name = &quot;One Unicorn&quot; });
    context.SaveChanges();
}
[/sourcecode]


When the context is used for the first time (in the Add call above) it will normally go through the process of building the model. Let’s instead see how this can be done using the lower-level building blocks.
<h3>It’s all about DbModelBuilder</h3>
Building a Code First model always starts with an instance of <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.dbmodelbuilder(v=VS.103).aspx">DbModelBuilder</a>. You’ve probably seen DbModelBuilder in the <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.dbcontext.onmodelcreating(v=vs.103).aspx">OnModelCreating</a> method. This is where DbContext gives you access to the model builder it is using to create a model. To use the lower-level building blocks you create the DbModelBuilder directly:

[sourcecode language="csharp"]
var builder = new DbModelBuilder();
[/sourcecode]


Once you have a model builder you need to give it some starting points to discover the model. This is done by calling the <a href="http://msdn.microsoft.com/en-us/library/gg696542(v=vs.103).aspx">Entity</a> method on the model builder:

[sourcecode language="csharp"]
builder.Entity&lt;Post&gt;();
builder.Entity&lt;Blog&gt;();
[/sourcecode]


DbContext normally does this for you by looking at the DbSets you have declared on your context and calling the Entity method for each of these.

Some other things to note here:
<ul>
	<li>You don’t need to call Entity for every entity type in your model. Code First follows references in the types it is given and so will discover all the model that is reachable from the types you give it. For example, in the code above calling Entity for only Blog or only Post would result in the same model because Post is reachable from Blog and vice versa.</li>
	<li>You can also use the <a href="http://msdn.microsoft.com/en-us/library/gg679474(v=vs.103).aspx">ComplexType</a> method to tell the builder about complex types.</li>
	<li>You can create <a href="http://msdn.microsoft.com/en-us/library/gg696117(v=vs.103).aspx">EntityTypeConfiguration</a> and <a href="http://msdn.microsoft.com/en-us/library/gg696149(v=vs.103).aspx">ComplexTypeConfiguration</a> instances and <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.modelconfiguration.configuration.configurationregistrar(v=vs.103).aspx">add them to the builder</a> instead of calling the Entity/ComplexType methods.</li>
	<li>You can provide additional configuration starting from each Entity call using the fluent API just as you would when overriding OnModelCreating</li>
</ul>
<h3>Building the model</h3>
Once you have the model builder configured you need to build the model. This is where you have more options using the building blocks. DbContext normally builds the model in this way:

[sourcecode language="csharp"]
var model = builder.Build(connection);
[/sourcecode]


As described above, this uses the given connection to connect to the database and determine appropriate mappings. If, for example, you want to explicitly tell Code First to build a database for SQL Server 2005, then you would instead build the model in this way:

[sourcecode language="csharp"]
var model = builder.Build(
    new DbProviderInfo(&quot;System.Data.SqlClient&quot;, &quot;2005&quot;));
[/sourcecode]


The <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.infrastructure.dbmodel(v=vs.103).aspx">DbModel</a> instance returned by the <a href="http://msdn.microsoft.com/en-us/library/gg696764(v=vs.103).aspx">Build</a> method is an object model representation of the underlying Entity Data Model (EDM). Unfortunately as of EF5 this EDM object model is not exposed publicly, so the DbModel instance is opaque. We intend to make this object model public in EF6, but as with anything that hasn’t happened yet, that could change. Once it is public you will be able to manipulate the EDM directly to tweak the model created by Code First.
<h3>Compiling the model</h3>
The EDM object model isn’t used directly by DbContext while the app runs, so it first needs to be compiled into a form that can be used. To do this call the <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.infrastructure.dbmodel.compile(v=vs.103).aspx">Compile</a> method:

[sourcecode language="csharp"]
var compiledModel = model.Compile();
[/sourcecode]


For EF geeks, the <a href="http://msdn.microsoft.com/en-us/library/system.data.entity.infrastructure.dbcompiledmodel(v=vs.103).aspx">DbCompiledModel</a> returned is currently a thin wrapper around a <a href="http://msdn.microsoft.com/en-us/library/system.data.metadata.edm.metadataworkspace(v=vs.110).aspx">MetadataWorkspace</a>, but this may not always be the case.
<h3>Caching the compiled model</h3>
The DbCompiledModel instance is the thing that DbContext caches. When using these building blocks you should find a way to cache the compiled model in your app. Likewise, do your own caching of the compiled model instances in situations where you need to use multiple models with the same context type.
<h3>Using the compiled model</h3>
Once you have a compiled model it’s easy to use it with DbContext—just pass an instance to the appropriate constructor. You’ll need to modify your context class to allow this:

[sourcecode]
public class BlogContext : DbContext
{
    public BlogContext(DbCompiledModel model)
        : base(model)
    {
    }

    public DbSet&lt;Blog&gt; Blogs { get; set; }
    public DbSet&lt;Posts&gt; Posts { get; set; }
}
[/sourcecode]

And then call it where you use the context:

[sourcecode language="csharp"]
using (var context = new BlogContext(compiledModel))
{
    context.Blogs.Add(new Blog { Name = &quot;One Unicorn&quot; });
    context.SaveChanges();
}
[/sourcecode]


DbContext will then use the given model instead of going through the process of creating one itself.
<h3>What about pluggable conventions?</h3>
Code First has a pluggable system for the conventions used to build and configure the model. Unfortunately, as of EF5 this system is internal. As with the EDM object model we intend to make this system public in EF6. (Disclaimer: as I said above, this is our intention, but things are not set in stone and it could change.)
<h3>Summary</h3>
By using the DbModelBuilder directly you can take more control over the way your Code First model is created and cached. In the future we will add additional interception/extensibility points such as a public EDM object model and pluggable conventions to provide even more control.
