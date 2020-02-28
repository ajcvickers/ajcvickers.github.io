---
layout: default
title: Extra-Lazy Collection Count with EF 4.1 (Part 1)
date: 2011-03-28 21:39
day: 28th
month: March
year: 2011
author: ajcvickers
permalink: 2011/03/28/extra-lazy-collection-count-with-ef-4-1-part-1/
---

# Entity Framework 4.1
# Extra-lazy collection count with EF 4.1 (Part 1)

<h3>Lazy loading</h3>  <p>Lazy loading of collections is the process whereby a collection of entities is automatically loaded from the database the first time that the collection property referring to the entities is accessed. Support for lazy loading was added to Entity Framework 4.0 and <a href="https://docs.microsoft.com/archive/blogs/adonet/using-dbcontext-in-ef-4-1-part-6-loading-related-entities">is described here for EF 4.1</a>. EF Lazy loading works with (almost) any implementation of ICollection<T> and does not mess with that implementation.</p>  <p>Lazy loading is useful in that it doesn’t require all entities that an application might use to be eagerly loaded up-front and also doesn’t require special code to be written to explicitly load related entities when they are needed—you just access the collection and it is automatically loaded if needed.</p>  <h3>Extra-laziness</h3>  <p>For some people lazy loading doesn’t go far enough. In particular, it is not uncommon to want to know how many related entities exist without actually loading all those entities. </p><p>For example, in a blogging app I might want to display a list of posts together with the number of comments for each post, and only load the comments if the post is actually opened. But with EF lazy loading if I access Comments.Count then all the comments will be loaded…even though I only need the count. What I really want here is for Count to ask the database for the number of comments without loading them; this is sometimes called being “extra-lazy”.</p>  <h3>Extra lazy in EF </h3>  <p>The Entity Framework has contained all the hooks to implement an extra-lazy Count property since EF 4.0. However, starting with EF 4.1 the DbContext API makes these hooks much easier to use such that the implementation is now not that difficult. There are two key parts:</p>  <ul>   <li>Creating a wrapper for ICollection<T> that intercepts calls to its methods and properties so that the Count property can be made to behave appropriately </li>    <li>Injecting the wrapped collections into entities as they are queried from the database      <br />LazyCountCollection </li> </ul>  <p>For the first part of the implementation we will create a class called LazyCountCollection. (In this post the collection class will be very specific to the implementation of the extra-lazy Count property. Look for an upcoming post that generalizes this for other things that you might do with the underlying query.) Here is LazyCountCollection:</p>  

``` c#
namespace LazyUnicorns
{
    using System.Collections;
    using System.Collections.Generic;
    using System.Data.Entity.Infrastructure;
    using System.Linq;

    public class LazyCountCollection<TElement> : ICollection<TElement>
    {
        private readonly ICollection<TElement> _collection;
        private readonly DbCollectionEntry _entry;
        private bool _isLoaded;

        public LazyCountCollection(ICollection<TElement> collection,
                                   DbCollectionEntry entry)
        {
            _collection = collection ?? new HashSet<TElement>();
            _entry = entry;
        }

        public void Add(TElement item)
        {
            _collection.Add(item);
        }

        public void Clear()
        {
            LazyLoad();
            _collection.Clear();
        }

        public bool Contains(TElement item)
        {
            LazyLoad();
            return _collection.Contains(item);
        }

        public void CopyTo(TElement[] array, int arrayIndex)
        {
            LazyLoad();
            _collection.CopyTo(array, arrayIndex);
        }

        public int Count
        {
            get
            {
                return _isLoaded ?
                       _collection.Count :
                       _entry.Query().Cast<TElement>().Count();
            }
        }

        public bool IsReadOnly
        {
            get
            {
                return _collection.IsReadOnly;
            }
        }

        public bool Remove(TElement item)
        {
            LazyLoad();
            return _collection.Remove(item);
        }

        public IEnumerator<TElement> GetEnumerator()
        {
            LazyLoad();
            return _collection.GetEnumerator();
        }

        IEnumerator IEnumerable.GetEnumerator()
        {
            // Can't call LazyLoad here due to bug in EF, but usually
            // when writing code the generic enumerator is called anyway.
            return ((IEnumerable)_collection).GetEnumerator();
        }

        private void LazyLoad()
        {
            if (!_isLoaded)
            {
                _isLoaded = true;
                _entry.Load();
            }
        }
    }
}

```

<p>The key things to notice in this code are:</p>

<ul>
  <li>The constructor is called with an existing ICollection<T> implementation (or null) and a DbCollectionEntry. The existing collection is wrapped (or a new HashSet<T> is wrapped if null is passed) and most methods and properties simply delegate behavior to the underlying collection. </li>

  <li>The object maintains its own flag indicating whether or not the collection has been loaded. This allows lazy loading to be performed at the method/property level. For example, I have chosen not to trigger lazy loading when adding a new item to the collection with the Add method. </li>

  <li>The Count method is implemented using the Query method from DbCollectionEntry—for <a href="https://docs.microsoft.com/archive/blogs/adonet/using-dbcontext-in-ef-4-1-part-6-loading-related-entities">more details of Query see this blog post</a>. If the collection has been loaded then Count just returns the size of the loaded collection. However, if the collection is not loaded then Count will use Query with the Count LINQ extension method to query the database for the number of entities in the collection without loading all those entities into memory. </li>

  <li>The collection uses the non-generic version of DbCollectionEntry because this makes other things easier. However, this means that the Query method returns a non-generic IQueryable and hence we use Cast to get the generic version on which we can call Count. </li>
</ul>

<h3>Initializing collections using ObjectMaterialized</h3>

<p>Okay, so that’s the collection implementation, but how do we get instances of these collections onto our entities? If possible we want to do this such that:</p>

<ul>
  <li>We don’t compromise the POCO-ness of our entities by requiring that they use special classes with references to EntityFramework.dll. </li>

  <li>We don’t require the app developer to write special code to initialize the collection each time that it is needed. </li>
</ul>

<p>One way to do this is to use the ObjectMaterialized event that was introduced in EF4. Materialized is a funny word—it describes the process of creating and initializing an object using values returned from a database query. We tried to think of a better name for this event…but we failed. (Sorry.) So anyway, this event is fired whenever an entity has been created and initialized by an EF query. We can register for this event and use it to process the entity after it has been initialized such that we can wrap its collections with instances of our special LazyCountCollection.</p>

<p>Here’s the event handler code:</p>

``` c#
namespace LazyUnicorns
{
    using System;
    using System.Collections.Generic;
    using System.Data.Entity;
    using System.Data.Entity.Infrastructure;
    using System.Linq;
    using System.Reflection;

    public abstract class CollectionInitializer
    {
        public virtual Type TryGetElementType(PropertyInfo collectionProperty)
        {
            // We can only replace properties that are declared as ICollection<T>
            // and have a setter.
            var propertyType = collectionProperty.PropertyType;
            if (propertyType.IsGenericType &&
                propertyType.GetGenericTypeDefinition() == typeof(ICollection<>) &&
                collectionProperty.GetSetMethod() != null)
            {
                return propertyType.GetGenericArguments().Single();
            }
            return null;
        }

        public virtual void InitializeCollections(DbContext context, object entity)
        {
            foreach (var property in entity.GetType().GetProperties())
            {
                var collectionEntry = context.Entry(entity)
                                          .Member(property.Name) as DbCollectionEntry;
                if (collectionEntry != null)
                {
                    var elementType = TryGetElementType(property);
                    if (elementType != null)
                    {
                        collectionEntry.CurrentValue
                            = CreateCollection(collectionEntry, elementType);
                    }
                }
            }
        }

        public abstract object CreateCollection(DbCollectionEntry collectionEntry,
                                                Type elementType);
    }
}

namespace LazyUnicorns
{
    using System;
    using System.Data.Entity.Infrastructure;

    public class LazyCountCollectionInitializer : CollectionInitializer
    {
        public override object CreateCollection(DbCollectionEntry collectionEntry,
                                                Type elementType)
        {
            var lazyCollectionType
                = typeof(LazyCountCollection<>).MakeGenericType(elementType);

            return Activator.CreateInstance(lazyCollectionType,
                                            collectionEntry.CurrentValue,
                                            collectionEntry);
        }
    }
}
```

<p>We’ll see how to use this code in a event handler in part 2. Here are some things to notice about this code:</p>

<ul>
  <li>The code is split into two classes. The abstract class is not specific to any type of collection. The concrete subclass is specific to our LazyCountCollection type. If we had another collection type we could re-use the abstract base class and just create a new concrete subclass to create instances of our new type. </li>

  <li>The code uses Reflection to find all public properties of the entity type and then uses the Member method of DbEntityEntry to get a DbMemberEntry for each property. This method can be called for any (non-indexed) property of the class, including unmapped properties, primitive properties, complex properties, reference navigation properties, and collection navigation properties. However, the actual runtime type of the returned object will only be DbCollectionEntry (which derives from DbMemberEntry) if the property represents a collection navigation property in our model. This means we can use the “as DbCollectionEntry” construct to filter for navigation properties. (If you didn’t follow all that don’t worry—it’s just a way of getting all the collections that we want to replace.) </li>

  <li>Once we have found a collection navigation property we use more Reflection to check that it is defined as ICollection<T> and has a setter, since we need both these things to be true to wrap and replace the collection. </li>

  <li>If we can replace the collection then we determine the element type and again use Reflection to create an instance of our LazyCountCollection. We then use DbCollectonEntry.CurrentValue to set this onto our entity. </li>
</ul>

<p>You may be looking at this and saying, “Wow! That’s a lot of Reflection to do for every queried entity!” I agree and in an upcoming post I’ll show how to cache this work such that we take much less of a performance hit from Reflection.</p>

<p>So that’s all the infrastructure we need. In <a href="/2011/03/28/extra-lazy-collection-count-with-ef-4-1-part-2/">part 2</a> we’ll see how to use this LazyCountCollection class.</p>
