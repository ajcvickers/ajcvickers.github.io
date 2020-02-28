---
layout: default
title: Rejecting changes to entities in EF 4.1
date: 2011-04-03 22:13
day: 3rd
month: April
year: 2011
author: ajcvickers
permalink: 2011/04/03/rejecting-changes-to-entities-in-ef-4-1/
---

# Entity Framework 4.1
# Rejecting changes to entities

<p>Imagine your app has an entity to which changes have been made and you want to reject those changes such that they won’t be saved to the database when SaveChanges is called. Using <a href="https://docs.microsoft.com/archive/blogs/adonet/ef-4-1-release-candidate-available">Entity Framework 4.1</a> you can do this by setting the state of the entity to Unchanged. For example:</p>  

``` c#
context.Entry(myEntity).State = EntityState.Unchanged;
```

<p>Here’s a simple test that demonstrates this behavior:</p>

``` c#
[TestMethod]
public void Setting_state_of_Modified_entity_to_Unchanged_rejects_changes()
{
    using (var context = new BlogContext())
    {
        // Find an entity and modify some property.
        var post = context.Posts.Where(p => p.Title == "Lazy Unicorns").Single();
        post.Title = "Bouncy Unicorns";
        Assert.AreEqual(EntityState.Modified, context.Entry(post).State);

        // Reject the change
        context.Entry(post).State = EntityState.Unchanged;

        Assert.AreEqual(EntityState.Unchanged, context.Entry(post).State);
        Assert.AreEqual("Lazy Unicorns", post.Title);

        // Nothing will be saved.
        Assert.AreEqual(0, context.SaveChanges());
    }
}
```

<p>There isn’t a single method to reject changes to all entities, but you can easily find all Modified entities in the context and set the state of each to Unchanged. For example:</p>

``` c#
foreach (var entry in context.ChangeTracker.Entries()
                        .Where(e => e.State == EntityState.Modified))
{
    entry.State = EntityState.Unchanged;
}
```

<p>If you may also have Added entities then you might want to detach these entities at the same time. For example:</p>

``` c#
foreach (var entry in context.ChangeTracker.Entries())
{
    if (entry.State == EntityState.Modified)
    {
        entry.State = EntityState.Unchanged;
    }
    else if (entry.State == EntityState.Added)
    {
        entry.State = EntityState.Detached;
    }
}
```

<p>Under the covers, changing the state of an entity from Modified to Unchanged first sets the values of all properties to the original values that were read from the database when it was queried, and then marks the entity as Unchanged. This will also reject changes to FK relationships since the original value of the FK will be restored.</p>

<p>One caveat to this is that if you are using independent associations in your model and want to reject changes to the relationships between entities then you need to write a lot more nasty code dealing with ObjectStateEntry and RelatedEnd objects for relationships. This is another reason to map FKs in your model if you can and not use independent associations. (At some point we hope to allow you to map FKs to the conceptual model but still not expose them in your object model—this is sometimes known as shadow-state. This would allow simple code like that above to work without having FK properties on your model.)</p>
