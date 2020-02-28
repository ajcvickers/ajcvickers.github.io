---
layout: post
title: Secrets of DetectChanges Part 1: What does DetectChanges do?
date: 2012-03-10 18:18
author: ajcvickers
comments: true
categories: [Change Tracking, DbContext API, DetectChanges, EF4, EF4.1, EF4.2, EF4.3, EF5, Entity Framework, Foreign Keys, POCO, Proxies, SaveChanges]
---
Entity Framework change tracking is often something that doesn't require too much thought from the app developer. However, change tracking and the DetectChanges method are a part of the stack where there is a tension between ease-of-use and performance. For this reason it can be useful to have an idea of what is actually going on such you can make an informed decision on what to do if the default behavior is not right for your app.
<h3>
Overview</h3>
I already <a href="/2011/12/05/should-you-use-entity-framework-change-tracking-proxies/">blogged about change tracking proxies</a> which touched on some areas of change tracking. In this four part series I'll focus on the DetectChanges method.
<ul>
	<li>Part 1 (this post) provides an introduction and describes briefly what DetectChanges does.</li>
	<li><a href="/2012/03/11/secrets-of-detectchanges-part-2-when-is-detectchanges-called-automatically/">Part 2</a> details when DetectChanges is called automatically and what the consequences of these calls are.</li>
	<li><a href="/2012/03/12/secrets-of-detectchanges-part-3-switching-off-automatic-detectchanges/">Part 3</a> shows how to switch off automatic calls to DetectChanges and covers when you must then call it manually.</li>
	<li><a href="/2012/03/13/secrets-of-detectchanges-part-4-binary-properties-and-complex-types/">Part 4</a> looks at special considerations for DetectChanges when your entities contain complex types or byte arrays.</li>
</ul>
All the examples in this series will use the following simple Code First model and context.

``` c#
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }

    public virtual ICollection Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }

    public int BlogId { get; set; }
    public virtual Blog Blog { get; set; }
}

public class AnotherBlogContext : DbContext
{
    public DbSet Blogs { get; set; }
    public DbSet Posts { get; set; }
}
```
<h3>The change tracking problem</h3>
Most EF applications make use of <a href="/2011/12/05/entity-types-supported-by-the-entity-framework/">persistent ignorant POCO entities</a> and snapshot change tracking. This means that there is no code in the entities themselves to keep track of changes or notify the context of changes.

Let’s look at the following code to illustrate the point:

``` c#
using (var context = new AnotherBlogContext())
{
    var post = context.Posts
                   .Single(p => p.Title == "My First Post");

    post.Title = "My Best Post";
    context.SaveChanges();
}
```

In this code, a single post is queried from the database, its title is changed, and then this change is saved back to the database.

But when Title property is changed nothing special happens because it is just a simple automatic property of the C# class. There is nothing in the entity that keeps track of the fact that it has changed (such as an <em>isDirty</em> flag) or of the fact that the original value of this property was "My First Post". There is also nothing in the entity that notifies the context that this change has happened.

So how does SaveChanges know that it must send an update to the database for the changed title? The answer is that is uses snapshot change tracking and DetectChanges.
<h3>Snapshot change tracking and DetectChanges</h3>
The EF context makes a snapshot of the properties of each entity when it is queried from the database. So in the example above, the context recorded in a snapshot that the Post entity had a Title property with the value "My First Post".

When SaveChanges is called it will in turn automatically call the DetectChanges method. DetectChanges scans all entities in the context and compares the current value of each property to the original value stored in the snapshot. If the property is found to have changed, then EF knows that it must send an update for that property to the database. In the example above, the current Title value of “My Best Post” is detected as different from the original Title value of “My First Post” and an appropriate update is generated.
<h3>What else does DetectChanges do?</h3>
In reality, DetectChanges does quite a bit more than what is described above. Most of this falls under the category of "fixup" which I hope to describe in more detail in a future post. Briefly, fixup is the process of updating the references between entities and adjusting internal state and indexes appropriately.

For example, imagine that in addition to changing the Title property, the foreign key of the Post entity is also changed:

``` c#
using (var context = new AnotherBlogContext())
{
    context.Blogs.Load();
    var post = context.Posts
                   .Single(p => p.Title == "My First Post");

    post.Title = "My Best Post";
    post.BlogId = 7;
    context.SaveChanges();
}
```

When DetectChanges is called (as part of SaveChanges) it will notice this change to the FK and do a few things:
<ul>
	<li>It will make sure that the new FK is saved to the database, just like for any property change.</li>
	<li>It will check whether or not a Blog entity with a primary key that matches the new value of the BlogId FK is being tracked by the context and, if it is, it will change the Blog navigation property to point to this entity.</li>
	<li>Since the Blog type contains a Posts inverse navigation property, DetectChanges will also make sure that these collections are updated. That is, the Post will be removed from the Posts collection on the original Blog entity and added to the Posts collection of the Blog with which it is now associated.</li>
	<li>It will update the state of the Blog entity, the state of its properties, and the state of any other affected entities and their properties.</li>
	<li>It will update internal indexes that keep track of dangling foreign keys and conceptually (but not actually) null foreign keys as appropriate.</li>
</ul>
Note that similar things would have happened if the navigation property had been updated instead of the FK, or if the inverse navigation property had been changed.
<h3>Isn't all this work very expensive?</h3>
If your context is tracking thousands of entities and doing a lot of work with them, then calling DetectChanges frequently can be expensive. If this is a problem for your app then read the upcoming posts to find out what to do about it.

That being said, for many applications, even when dealing with lots of entities, DetectChanges does not result in a performance bottleneck. Trying to optimize for DetectChanges when you don't need to can be an excellent way of introducing subtle bugs into your code.

Up next: Secrets* of DetectChanges Part 2: When is DetectChanges called automatically?

*Okay, so these aren't really <em>secrets</em> as such. But “secrets” sounded better than "things that EF geeks might be interested in knowing about". And it's shorter to tweet.
