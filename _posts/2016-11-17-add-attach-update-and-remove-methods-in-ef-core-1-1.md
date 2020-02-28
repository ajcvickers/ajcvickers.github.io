---
layout: post
title: Add, Attach, Update, and Remove methods in EF Core 1.1
date: 2016-11-17 16:50
author: ajcvickers
comments: true
categories: [Change Tracking, DbContext, DbContext API, DbSet.Add, DbSet.Attach, DbSet.Renove, DbSet.Update, DetectChanges, EF Core, Entity Framework, EntityState, SaveChanges]
---
EF Core provides a variety of ways to start tracking entities or change their state. This post gives a brief overview of the different approaches.



<h2>Tracking queries</h2>

Queries will automatically track returned entities unless tracking has been turned off. If you want to track entities from a query, then just use this automatic tracking. Do not use a no-tracking query and then try to attach entities--doing so is slower and can require special handling for shadow properties.

<h2>Entity states</h2>

Tracked entities can be in one of four states. The state of an entity determines how it is processed when SaveChanges is called.

<ul>
<li>Added: The entity does not yet exist in the database. SaveChanges should insert it.</li>
<li>Unchanged: The entity exists in the database and has not been modified on the client. SaveChanges should ignore it.</li>
<li>Modified: The entity exists in the database and has been modified on the client. SaveChanges should send updates for it.</li>
<li>Deleted: The entity exists in the database but should be deleted. SaveChanges should delete it.</li>
</ul>

The Detached state is given to any entity that is not being tracked by the context.

<h2>Add, Attach, and Update methods</h2>

These methods are found on DbContext and also on DbSet<TEntity>. The DbContext and DbSet methods behave in exactly the same way.

All these methods work with the graph of untracked entities reachable from the entity passed to the method. So if you have a Blog which references two Posts and you call Add for the Blog, then the Posts will also be Added.

These methods do not automatically call DetectChanges. This is different from EF6 where these methods would automatically call DetectChanges.

<h3>Add</h3>

Add puts all entities in the graph into the Added state. This means they will be inserted into the database on SaveChanges.

<h3>Attach</h3>

Attach puts all entities in the graph into the Unchanged state. However, entities will be put in the Added state if they have store-generated keys (e.g. Identity column) and no key value has been set.

This means that when exclusively using store-generated keys, Attach can be used to start tracking a mix of new and existing entities where the existing entities have not changed. The new entities will be inserted while the existing entities will not be saved other than to update any necessary FK values.

(In EF Core 1.1 the root entity is special-cased such that it will always be marked as Unchanged even if it has a store-generated key with no value set. A fix for this has been checked in for the next release.)

<h3>Update</h3>

Update works the same as Attach except that entities are put in the Modified state instead of the Unchanged state.

This means that when exclusively using store-generated keys, Update can be used to start tracking a mix of new and existing entities where the existing entities may have some modifications. The new entities will be inserted while the existing entities will be updated.

<h2>Remove</h2>

Remove affects only the entity passed to the method. It does not traverse the graph of reachable entities.

If the entity is in the Unchanged or Modified state, indicating that it exists in the database, then it is put in the Deleted state.

If the entity is in the Added state, indicating that it has not yet been inserted, then it is detached from the context and is no longer tracked.

Remove is usually used with entities that are already being tracked. If the entity is not tracked, then it is first Attached and then placed in the Deleted state.

<h2>Range methods</h2>

The AddRange, AttachRange, UpdateRange, and RemoveRange methods work exactly the same as calling the non-Range methods multiple times. They are slightly more efficient than multiple calls to the non-Range methods, but the efficiency gain is very small because none of these methods now automatically call DetectChanges.

<h2>Setting entity state directly</h2>

The state of an entity can always be set directly using the DbContext.Entry method. For example:

[code lang=csharp]
context.Entry(blog).State = EntityState.Unchanged;
[/code]

Setting the state this way:

<ul>
<li>Only affects the given entity. It does not traverse the graph of reachable entities.</li>
<li>Always sets the state specified, regardless of the current state of the entity.</li>
</ul>

<h2>TrackGraph</h2>

The TrackGraph method provides full control over the state set for entities in a graph. For example, imagine your entities all have integer keys called "Id", and that these keys are set to negative temporary values before they are inserted into the database. TrackGraph can then be used to set the state of each entity appropriately and inform EF that the negative values are temporary.

[code lang=csharp]
context.ChangeTracker.TrackGraph(blog, node =>
{
    var entry = node.Entry;

    if ((int)entry.Property("Id").CurrentValue < 0)
    {
        entry.State = EntityState.Added;
        entry.Property("Id").IsTemporary = true;
    }
    else
    {
        entry.State = EntityState.Modified;
    }
});
[/code]

<h2>Using navigation properties</h2>

New entities can be tracked by setting them on a reference navigation property or by adding them to a collection navigation property of an entity that is already being tracked. In this case the new entities are always put in the Added state.

Usually DetectChanges does the job of finding these new entities. However, you can always use <a href="https://blog.oneunicorn.com/2016/11/16/notification-entities-in-ef-core-1-1/">notification entities</a> or use the <code>context.Entry</code> methods for references and collections, in which case DetectChanges is not needed,

<h2>Async Add</h2>

You might notice that there are async versions of the Add methods. It is very rare that you will need these. They are included so that client-side value generation that needs to access the database can do so asyncronously. Even then it is usually fine to use the synchronous methods.

<h2>Summary</h2>

EF Core provides a variety of different ways to start tracking entities or change their state. Most of the time the Add, Attach, Update, and Remove methods should be used. For more control, use TrackGraph coupled with setting the entity state directly.
