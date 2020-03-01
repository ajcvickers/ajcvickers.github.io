---
layout: default
title: "EF 6.1: Creating indexes with IndexAttribute"
date: 2014-02-15 16:01
day: 15th
month: February
year: 2014
author: ajcvickers
permalink: 2014/02/15/ef-6-1-creating-indexes-with-indexattribute/
---

# Entity Framework 6.1
# Creating indexes with IndexAttribute

Since EF 4.3 it has been possible to use CreateIndex and DropIndex in Code First Migrations to create and drop indexes. However this had to be done manually by editing the migration because the index was not included anywhere in the Code First model. Now with EF 6.1 it is possible to add index specifications to the model such that creating and dropping indexes can be handled automatically by Migrations.
<h2>Single column indexes</h2>
Consider a simple Blog entity:

``` c#
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int Rating { get; set; }
    public virtual ICollection<Post> Posts { get; set; }
}
```

Let's assume this entity is already in our model and migrations have been created and applied so the model and database are both up-to-date. The easiest way to add an index is to place IndexAttribute onto a property. For example, let's add an index to the column mapped to by the Rating property:

``` c#
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }

    [Index]
    public int Rating { get; set; }

    public virtual ICollection<Post> Posts { get; set; }
}
```

After doing this using Add-Migration will scaffold a migration something like this:

``` c#
public partial class Two : DbMigration
{
    public override void Up()
    {
        CreateIndex("dbo.Blogs", "Rating");
    }

    public override void Down()
    {
        DropIndex("dbo.Blogs", new[] { "Rating" });
    }
}
```

The index is being created with a default name and default options. The defaults are as follows:
<ul>
	<li>Name: IX_[column_name]</li>
	<li>Not unique</li>
	<li>Not clustered</li>
</ul>
You can also use IndexAttribute to give the index a specific name and options. For example, let's add a name to the index for the Rating column:

``` c#
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }

    [Index("RatingIndex")]
    public int Rating { get; set; }

    public virtual ICollection<Post> Posts { get; set; }
}
```

Scaffolding another migration for this change results in:

``` c#
public partial class Three : DbMigration
{
    public override void Up()
    {
        RenameIndex(table: "dbo.Blogs", name: "IX_Rating", newName: "RatingIndex");
    }

    public override void Down()
    {
        RenameIndex(table: "dbo.Blogs", name: "RatingIndex", newName: "IX_Rating");
    }
}
```

Notice that Migrations has scaffolded a rename for the index from the default name to the new name.
<h2>Multiple column indexes</h2>
Indexes that span multiple columns can also be scaffolded by using the same index name on multiple properties. For example:

``` c#
public class Blog
{
    [Index("IdAndRating", 1)]
    public int Id { get; set; }

    public string Title { get; set; }

    [Index("RatingIndex")]
    [Index("IdAndRating", 2, IsUnique = true)]
    public int Rating { get; set; }

    public virtual ICollection<Post> Posts { get; set; }
}
```

Notice that the order of columns in the index is also specified. The unique and clustered options can be specified in one or all IndexAttributes. If these options are specified on more than one attribute with a given name then they must match.

Scaffolding a migration for this change results in:

``` c#
public partial class Four : DbMigration
{
    public override void Up()
    {
        CreateIndex("dbo.Blogs", new[] { "Id", "Rating" }, unique: true, name: "IdAndRating");
    }

    public override void Down()
    {
        DropIndex("dbo.Blogs", "IdAndRating");
    }
}
```
<h2>Index conventions</h2>
The ForeignKeyIndexConvention Code First convention causes indexes to be created for the columns of any foreign key in the model unless these columns already have an index specified using IndexAttribute. If you don't want indexes for your FKs you can remove this convention:

``` c#
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    modelBuilder.Conventions.Remove<ForeignKeyIndexConvention>();
}
```
<h2>What IndexAttribute doesn't do</h2>
IndexAttribute can be used to create a unique index in the database. However, this does not mean that EF will be able to reason about the uniqueness of the column when dealing with relationships, etc. This feature usually referred to as support for “unique constraints” which can be voted for as a <a href="http://data.uservoice.com/forums/72025-entity-framework-feature-suggestions/suggestions/1050579-unique-constraint-i-e-candidate-key-support">feature suggestion</a> and on the <a href="http://entityframework.codeplex.com/workitem/299">CodePlex work item</a>.
