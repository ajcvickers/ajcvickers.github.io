---
layout: post
title: EF 6.1: Creating indexes with IndexAttribute
date: 2014-02-15 16:01
author: ajcvickers
comments: true
categories: [Code First, Code First Migrations, Data Annotations, EF6.1, Entity Framework, Indexs]
---
Since EF 4.3 it has been possible to use CreateIndex and DropIndex in Code First Migrations to create and drop indexes. However this had to be done manually by editing the migration because the index was not included anywhere in the Code First model. Now with EF 6.1 it is possible to add index specifications to the model such that creating and dropping indexes can be handled automatically by Migrations.<!--more-->
<h2>Single column indexes</h2>
Consider a simple Blog entity:

[code language="csharp"]
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int Rating { get; set; }
    public virtual ICollection&lt;Post&gt; Posts { get; set; }
}
[/code]

Let’s assume this entity is already in our model and migrations have been created and applied so the model and database are both up-to-date. The easiest way to add an index is to place IndexAttribute onto a property. For example, let’s add an index to the column mapped to by the Rating property:

[code language="csharp"]
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }

    [Index]
    public int Rating { get; set; }

    public virtual ICollection&lt;Post&gt; Posts { get; set; }
}
[/code]

After doing this using Add-Migration will scaffold a migration something like this:

[code language="csharp"]
public partial class Two : DbMigration
{
    public override void Up()
    {
        CreateIndex(&quot;dbo.Blogs&quot;, &quot;Rating&quot;);
    }

    public override void Down()
    {
        DropIndex(&quot;dbo.Blogs&quot;, new[] { &quot;Rating&quot; });
    }
}
[/code]

The index is being created with a default name and default options. The defaults are as follows:
<ul>
	<li>Name: IX_[column_name]</li>
	<li>Not unique</li>
	<li>Not clustered</li>
</ul>
You can also use IndexAttribute to give the index a specific name and options. For example, let’s add a name to the index for the Rating column:

[code language="csharp"]
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }

    [Index(&quot;RatingIndex&quot;)]
    public int Rating { get; set; }

    public virtual ICollection&lt;Post&gt; Posts { get; set; }
}
[/code]

Scaffolding another migration for this change results in:

[code language="csharp"]
public partial class Three : DbMigration
{
    public override void Up()
    {
        RenameIndex(table: &quot;dbo.Blogs&quot;, name: &quot;IX_Rating&quot;, newName: &quot;RatingIndex&quot;);
    }

    public override void Down()
    {
        RenameIndex(table: &quot;dbo.Blogs&quot;, name: &quot;RatingIndex&quot;, newName: &quot;IX_Rating&quot;);
    }
}
[/code]

Notice that Migrations has scaffolded a rename for the index from the default name to the new name.
<h2>Multiple column indexes</h2>
Indexes that span multiple columns can also be scaffolded by using the same index name on multiple properties. For example:

[code language="csharp"]
public class Blog
{
    [Index(&quot;IdAndRating&quot;, 1)]
    public int Id { get; set; }

    public string Title { get; set; }

    [Index(&quot;RatingIndex&quot;)]
    [Index(&quot;IdAndRating&quot;, 2, IsUnique = true)]
    public int Rating { get; set; }

    public virtual ICollection&lt;Post&gt; Posts { get; set; }
}
[/code]

Notice that the order of columns in the index is also specified. The unique and clustered options can be specified in one or all IndexAttributes. If these options are specified on more than one attribute with a given name then they must match.

Scaffolding a migration for this change results in:

[code language="csharp"]
public partial class Four : DbMigration
{
    public override void Up()
    {
        CreateIndex(&quot;dbo.Blogs&quot;, new[] { &quot;Id&quot;, &quot;Rating&quot; }, unique: true, name: &quot;IdAndRating&quot;);
    }

    public override void Down()
    {
        DropIndex(&quot;dbo.Blogs&quot;, &quot;IdAndRating&quot;);
    }
}
[/code]
<h2>Index conventions</h2>
The ForeignKeyIndexConvention Code First convention causes indexes to be created for the columns of any foreign key in the model unless these columns already have an index specified using IndexAttribute. If you don’t want indexes for your FKs you can remove this convention:

[code language="csharp"]
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    modelBuilder.Conventions.Remove&lt;ForeignKeyIndexConvention&gt;();
}
[/code]
<h2>What IndexAttribute doesn’t do</h2>
IndexAttribute can be used to create a unique index in the database. However, this does not mean that EF will be able to reason about the uniqueness of the column when dealing with relationships, etc. This feature usually referred to as support for “unique constraints” which can be voted for as a <a href="http://data.uservoice.com/forums/72025-entity-framework-feature-suggestions/suggestions/1050579-unique-constraint-i-e-candidate-key-support">feature suggestion</a> and on the <a href="http://entityframework.codeplex.com/workitem/299">CodePlex work item</a>.