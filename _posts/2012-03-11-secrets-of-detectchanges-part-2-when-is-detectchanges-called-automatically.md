---
layout: default
title: "Secrets of DetectChanges Part 2: When is DetectChanges called automatically?"
date: 2012-03-11 10:34
day: 11th
month: March
year: 2012
author: ajcvickers
permalink: 2012/03/11/secrets-of-detectchanges-part-2-when-is-detectchanges-called-automatically/
---

# Secrets of DetectChanges
# Part 2: When is DetectChanges called automatically?

### Relevance

These posts were written in 2012 for Entity Framework 4.3.
However, the information is fundamentally correct for all versions up to and including EF6.

The general concepts are also relevant for EF Core.

As always, [use your noggin](/noggin/).

---

<a href="/2012/03/10/secrets-of-detectchanges-part-1-what-does-detectchanges-do/">Part 1</a> of this series on DetectChanges described why DetectChanges is needed to discover the changes that have been made to POCO entities. This part will expand on that information and look at when it is important for the context to know about these changes. This will provide the basis for detailing when the context calls DetectChanges automatically.
<h3>When does the context need to know?</h3>
SaveChanges is the most important time at which the context needs to know what changes have been made to entities. This is somewhat obvious; if the changes are not known, then SaveChanges will not know which inserts/updates/deletes to send to the database. This is why DetectChanges is always called as part of SaveChanges (unless this has been explicitly disabled) even when using ObjectContext in EF4.

However, the context also needs to know about changes at other times. For example, if you ask the context for the state of an entity, then the context needs to know if any properties of an entity have changed in order to report that the entity is either Unchanged or Modified. Consider this code:

``` c#
using (var context = new AnotherBlogContext())
{
    var post = context.Posts
        .Single(p => p.Title == "My First Post");
    post.Title = "My Best Post";

    Console.WriteLine(context.Entry(post).State);
}
```


The post entity has been changed, so it would be reasonable for the output to be “Modified”…and indeed it is. The reason it is Modified and not Unchanged is that the Entry method calls DetectChanges.

Fixup, which was covered in the context of DetectChanges in part 1, also needs to know about changes. Fixup can happen at various times—I'll blog about this sometime. One time it can happen is when an entity is brought into the context with Find. For example:

``` c#
using (var context = new AnotherBlogContext())
{
    var post = context.Posts.First(p => p.BlogId == 1);
    post.BlogId = 2;

    var blog2 = context.Blogs.Find(2);

    Assert.Same(blog2, post.Blog);
    Assert.Contains(post, blog2.Posts);
}
```


This code queries for a Post with an BlogId FK of 1. It then changes the FK to 2. Next Find is used to find the Blog entity with primary key 2. EF will attempt to do fixup when this Post entity is brought into the context. This will only be possible if EF knows that the FK of the post is now 2, and this can only happen if DetectChanges has been called. Thankfully, Find calls DetectChanges and so fixup happens correctly and the Asserts pass.
<h3>The methods that call DetectChanges</h3>
From the examples above it is clear that DetectChanges often needs to be called for the methods of DbContext and its associated classes to behave as expected. This is why DetectChanges is called automatically by each of these methods:
<ul>
	<li>DbSet.Find</li>
	<li>DbSet.Local</li>
	<li>DbSet.Remove</li>
	<li>DbSet.Add</li>
	<li>DbSet.Attach</li>
	<li>DbContext.SaveChanges</li>
	<li>DbContext.GetValidationErrors</li>
	<li>DbContext.Entry</li>
	<li>DbChangeTracker.Entries</li>
</ul>
<h3>Special considerations when overriding SaveChanges or ValidateEntity</h3>
DetectChanges is called as part of the implementation of the SaveChanges. This means that if you override SaveChanges in your context, then DetectChanges will not have been called <em>before</em> your SaveChanges method is called. This can sometimes catch people out, especially when checking if an entity has been modified or not since its state may not be set to Modified until DetectChanges is called. Luckily this happens a lot less with the DbContext SaveChanges than it used to with ObjectContext SaveChanges because the Entry and Entries methods that are used to access entity state automatically call DetectChanges.

ValidateEntity is used for performing custom validation of an entity during GetValidationErrors or SaveChanges. Unlike SaveChanges, ValidateEntity is called <em>after</em> DetectChanges has been called. This is because validation needs to be done on what is going to be saved, and this is only known after DetectChanges has been called. Usually code in ValidateEntity will not modify the property values of the entity, just validate them. However, if it does change a property value then it may need to call DetectChanges again manually for that change to be saved correctly. An alternative to this is to modify the property in a way that DetectChanges is not needed—the way to do this is shown in Part 3 of this series.
<h3>So why don't all context methods call DetectChanges?</h3>
Calling DetectChanges any time that something in a POCO entity might have changed turns out to be prohibitively expensive. For example, when executing a query, control returns to the application code after each entity is materialized—in other words, many, many times in a single query. If the application code changes something while processing the entity, then it could theoretically affect the behavior for fixing up the next entity, and so on.

I'm not going to write code that demonstrates this because it gets fairly convoluted. The bottom line is that if DetectChanges was called automatically any time that it could theoretically affect behavior, then running this code with 100 Posts in the database would result in EF calling DetectChanges at least 100 times:

``` c#
using (var context = new AnotherBlogContext())
{
    foreach (var post in context.Posts)
    {
        Console.WriteLine(post.Title);
    }
}
```


So calling DetectChanges at every possible opportunity is prohibitively expensive, but never calling it automatically results in unexpected and unintuitive behavior in common scenarios. This is why DetectChanges is called automatically in the common places it is needed, but not everywhere it might possibly be needed.
<h3>Summary</h3>
The DbContext API calls DetectChanges automatically in the most common and useful places where it might be needed without calling it so frequently as to be a hindrance. In Part 3 we'll look at how to switch off these automatic calls to DetectChanges and how you can then know whether or not you need to call it manually.
