---
layout: post
title: The key to AddOrUpdate
date: 2012-05-03 19:58
author: ajcvickers
comments: true
categories: [Change Tracking, DbContext, DbContext API, DbSet.Add, DbSet.Attach, EF4.1, EF4.2, EF4.3, EF5, Entity Framework]
---
The <a href="http://msdn.microsoft.com/en-us/library/gg696418(v=vs.103).aspx">DbSet.Find</a> method provides an easy way to lookup an entity given its primary key. One place this can be useful is in an AddOrUpdate method when loading an entity which can then be updated with values from another application tier—for example, updating an entity with values from a client in a web application.

However, it isn’t so easy to do this in a generic way on any entity type without specific knowledge of which properties make up the primary key. This is something we will make easier in a future release of EF, but for now this blog post shows how to write some extension methods that make this process easier.
<h3>The scenario</h3>
Let’s say we have an object that has been created outside the context and contains values that need to be written back to the database. There are many opinions out there on how to do this and they each have their advantages and disadvantages. Some options are:
<ol>
	<li>Write all property values (or a fixed sub-set) to the database always even if this means sending updates for properties that haven’t changed.</li>
	<li>Query for the entity before updating and use the property values of the queried entity to determine which values have actually changed so that updates are only sent for only those properties.</li>
	<li>Track the original values of all properties across tiers and then use these values to determine which property values have actually changed so that updates are sent only for those properties.</li>
</ol>
Concurrency tokens may also be used with options 2 and 3. The point of this post is not to make any judgment about which is best but rather to show how using EF to obtain primary key information can enable generic methods to be written for options 1 and 2.
<h3>The non-generic AddOrUpdate</h3>
Assuming we’re going for option 2 above (option 1 is covered later), then, using the model at the end of this post, a method to add or update an OrderLine might look like this:

``` c#
public OrderLine AddOrUpdate(WindyContext context, OrderLine orderLine)
{
    var trackedOrderLine = context.OrderLines.Find(orderLine.OrderId, orderLine.ProductId);
    if (trackedOrderLine != null)
    {
        context.Entry(trackedOrderLine).CurrentValues.SetValues(orderLine);
        return trackedOrderLine;
    }

    context.OrderLines.Add(orderLine);
    return orderLine;
}
```

The keys points from this code are:
<ul>
	<li>Find is used to query the database for the entity with the same primary key values as the object passed to AddOrUpdate. Two values are passed to Find since OrderLine has a two-part composite key.</li>
	<li>If Find returns null it means that no OrderLine with the given key was found. In this case the new OrderLine needs to be inserted, so it is passed to the Add method.</li>
	<li>If Find returned an entity then it must be updated with values from the object passed to AddOrUpdate. The SetValues method does this. Crucially, only those properties with values that are different will be marked as modified.</li>
	<li>After calling AddOrUpdate SaveChanges must be called. This either inserts a new entity or sends an update for the properties that were modified. SaveChanges does nothing if no properties were modified.</li>
	<li>The method returns the entity added or updated because this can be a <a href="/2011/04/19/ef-4-1-quick-tip-addattach-returns-your-entity/">useful pattern for composition</a>.</li>
</ul>
<h3>A generic AddOrUpdate using Find</h3>
To make the AddOrUpdate method generic all uses of OrderLine need to be replaced by uses of a generic type—let’s call it TEntity. This is easy for the OrderLines property—it can be replaced with a call to Set<Tentity>().

The trickier part is the use of the OrderLine to get the primary key values for Find. This is where the extension method will be used—let’s call it KeyValuesFor. The generic code will then look like this:

``` c#
public TEntity AddOrUpdate<TEntity>(DbContext context, TEntity entity)
    where TEntity : class
{
    var tracked = context.Set<TEntity>().Find(context.KeyValuesFor(entity));
    if (tracked != null)
    {
        context.Entry(tracked).CurrentValues.SetValues(entity);
        return tracked;
    }

    context.Set<TEntity>().Add(entity);
    return entity;
}
```

<h3>Finding primary key property names</h3>
Before looking at property values let’s take a step back and write a method to find the property names that make up the primary key. As of EF5 this code requires dropping down to ObjectContext and using the MetadataWorkspace:

``` c#
public static IEnumerable<string> KeysFor(this DbContext context, Type entityType)
{
    Contract.Requires(context != null);
    Contract.Requires(entityType != null);

    entityType = ObjectContext.GetObjectType(entityType);

    var metadataWorkspace =
        ((IObjectContextAdapter)context).ObjectContext.MetadataWorkspace;
    var objectItemCollection = 
        (ObjectItemCollection)metadataWorkspace.GetItemCollection(DataSpace.OSpace);

    var ospaceType = metadataWorkspace
        .GetItems<EntityType>(DataSpace.OSpace)
        .SingleOrDefault(t => objectItemCollection.GetClrType(t) == entityType);

    if (ospaceType == null)
    {
        throw new ArgumentException(
            string.Format(
                "The type '{0}' is not mapped as an entity type.",
                entityType.Name),
            "entityType");
    }

    return ospaceType.KeyMembers.Select(k => k.Name);
}
```

This method returns a list of names because for entities with composite keys (like OrderLine) there are multiple properties that form the key. For entities that don’t have composite keys (which is the common case) the returned list will only contain one item.

You don’t really need to know the details of what is happening here, but for those interested:
<ul>
	<li>Metadata for all the o-space types known about by the context is requested. This is the ObjectItemCollection. (O-space is jargon for object-space which means metadata about your CLR types.)</li>
	<li>O-space metadata specifically for entity types is requested.</li>
	<li>This is filtered for the o-space type that matches the CLR type of the entity. Since the code is using EF 4.1 or above its safe to assume there will be zero or one matches.</li>
	<li>Assuming that a type is found the KeyMembers property of its metadata is used to obtain and return a list of key names.</li>
	<li>The code uses <a href="http://research.microsoft.com/en-us/projects/contracts/">Code Contracts</a> but if you’re not using these then just remove the Contract.Requires calls.</li>
	<li>Thanks to wiky87 for pointing out that the original code didn't work for proxy types. This is fixed by the GetObjectType call which returns the real entity type when given a proxy type.</li>
</ul>
<h3>Finding primary key property values</h3>
Once the primary key properties are known it is fairly easy to get the values of these properties:

``` c#
public static object[] KeyValuesFor(this DbContext context, object entity)
{
    Contract.Requires(context != null);
    Contract.Requires(entity != null);

    var entry = context.Entry(entity);
    return context.KeysFor(entity.GetType())
        .Select(k => entry.Property(k).CurrentValue)
        .ToArray();
}
```

This method returns an array of values because that’s what the params parameter of Find requires.
<h3>A different generic AddOrUpdate</h3>
Suppose that instead of using Find you decided to go with option 1 from those listed above. A common non-generic way to write this method is:

``` c#
public OrderLine AddOrUpdate(WindyContext context, OrderLine orderLine)
{
    context.Entry(orderLine).State =
        (orderLine.OrderId == 0 && orderLine.ProductId == 0)
            ? EntityState.Added
            : EntityState.Modified;
    
    return orderLine;
}
```

This method uses the convention that if the primary key is zero then the entity is new and so should be inserted. Otherwise it already exists in the database and so should be updated.

This convention can be generalized to say that if all the properties that make up an entity’s primary key have default values (0, null, etc.) then the entity is new otherwise it already exists.

Using this generalized convention and the KeyValusFor method the AddOrUpdate can again be made generic:

``` c#
public TEntity AddOrUpdate<TEntity>(DbContext context, TEntity entity)
    where TEntity : class
{
    context.Entry(entity).State =
        context.KeyValuesFor(entity).All(IsDefaultValue)
            ? EntityState.Added
            : EntityState.Modified;

    return entity;
}

private static bool IsDefaultValue(object keyValue)
{
    return keyValue == null
           || (keyValue.GetType().IsValueType
               && Equals(Activator.CreateInstance(keyValue.GetType()), keyValue));
}
```

(Note that the helper method that checks whether or not a key value is default might not work for some corner case types like void, but that doesn’t matter for our purposes because such types will never be used for EF key properties anyway.)
<h3>The model</h3>
For reference here’s the model I used while writing this post:

``` c#
public class Order
{
    public int Id { get; set; }

    public virtual ICollection<OrderLine> OrderLines { get; set; }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }

    public virtual ICollection<OrderLine> OrderLines { get; set; }
}

public class OrderLine
{
    [Key, Column(Order = 1)]
    public int OrderId { get; set; }
    [Key, Column(Order = 2)]
    public int ProductId { get; set; }

    public int Quantity { get; set; }

    public virtual Order Order { get; set; }
    public virtual Product Product { get; set; }
}

public class WindyContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<OrderLine> OrderLines { get; set; }
}
```

Thanks
Arthur
