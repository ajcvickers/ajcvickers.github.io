---
layout: post
title: Rejecting changes to entities in EF 4.1
date: 2011-04-03 22:13
author: ajcvickers
comments: true
categories: [Change Tracking, Code First, DbContext, DbContext API, Entity Framework]
---
<p>Imagine your app has an entity to which changes have been made and you want to reject those changes such that they won’t be saved to the database when SaveChanges is called. Using <a href="http://blogs.msdn.com/b/adonet/archive/2011/03/15/ef-4-1-release-candidate-available.aspx">Entity Framework 4.1</a> you can do this by setting the state of the entity to Unchanged. For example:</p>  <div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:3fc98555-2086-4be4-9bc6-0fc8f6fe013d" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
context.Entry(myEntity).State = EntityState.Unchanged;
[/sourcecode]
</pre>
</div>

<p>&#160;</p><!--more--><p>Here’s a simple test that demonstrates this behavior:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:421f9455-cd05-449c-a9e4-3af0e9f8e271" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp" padlinenumbers="true"]
[TestMethod]
public void Setting_state_of_Modified_entity_to_Unchanged_rejects_changes()
{
    using (var context = new BlogContext())
    {
        // Find an entity and modify some property.
        var post = context.Posts.Where(p =&gt; p.Title == &quot;Lazy Unicorns&quot;).Single();
        post.Title = &quot;Bouncy Unicorns&quot;;
        Assert.AreEqual(EntityState.Modified, context.Entry(post).State);

        // Reject the change
        context.Entry(post).State = EntityState.Unchanged;

        Assert.AreEqual(EntityState.Unchanged, context.Entry(post).State);
        Assert.AreEqual(&quot;Lazy Unicorns&quot;, post.Title);

        // Nothing will be saved.
        Assert.AreEqual(0, context.SaveChanges());
    }
}
[/sourcecode]
</pre>
</div>

<p>There isn’t a single method to reject changes to all entities, but you can easily find all Modified entities in the context and set the state of each to Unchanged. For example:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:7f88c7f8-1ca9-497f-961d-0d7add273ac4" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
foreach (var entry in context.ChangeTracker.Entries()
                        .Where(e =&gt; e.State == EntityState.Modified))
{
    entry.State = EntityState.Unchanged;
}
[/sourcecode]
</pre>
</div>

<p>If you may also have Added entities then you might want to detach these entities at the same time. For example:</p>

<div style="display:inline;float:none;margin:0;padding:0;" id="scid:C89E2BDB-ADD3-4f7a-9810-1B7EACF446C1:c0b19bb4-26d0-43cc-bcfe-5cff44e251a6" class="wlWriterEditableSmartContent"><pre>
[sourcecode language="csharp"]
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
[/sourcecode]
</pre>
</div>

<p>Under the covers, changing the state of an entity from Modified to Unchanged first sets the values of all properties to the original values that were read from the database when it was queried, and then marks the entity as Unchanged. This will also reject changes to FK relationships since the original value of the FK will be restored.</p>

<p>One caveat to this is that if you are using independent associations in your model and want to reject changes to the relationships between entities then you need to write a lot more nasty code dealing with ObjectStateEntry and RelatedEnd objects for relationships. This is another reason to map FKs in your model if you can and not use independent associations. (At some point we hope to allow you to map FKs to the conceptual model but still not expose them in your object model—this is sometimes known as shadow-state. This would allow simple code like that above to work without having FK properties on your model.)</p>

<p>Thanks for reading!
  <br />Arthur</p>
