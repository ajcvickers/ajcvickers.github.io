---
layout: default
title: A More General Queryable Collection
date: 2011-03-30 20:58
day: 30th
month: March
year: 2011
author: ajcvickers
permalink: 2011/03/30/a-more-general-queryable-collection/
---

# Entity Framework 4.1
# A more general Queryable collection

<p>In the last three posts we looked at an implementation of <a href="/2011/03/28/extra-lazy-collection-count-with-ef-4-1-part-1/">extra-lazy Count for EF 4.1</a> and how to <a href="/2011/03/29/lazycountcollection-with-better-performance/">reduce the Reflection cost of this implementation</a>. However, when looking at LazyCountCollection it is fairly apparent that the same pattern can be used for more than just extra-lazy Count. In this we’ll look at a more general implementation of ICollection<T> that contains an underlying IQueryable<T> that can be used for more than just extra-lazy Count.</p><p>LazyCountCollection is an ICollection<T> implementation that delegates most operations to another underlying ICollection<T> but also contains a DbCollectionEntry that is occasionally used instead of the underlying collection. However, the only part of DbCollectionEntry that we really need is the Query method, which returns an IQueryable<T>. Therefore, we can make our implementation more general by working only with IQueryable<T> instead of the DbCollectionEntry. Here’s the code:</p>  

``` c#
namespace LazyUnicorns
{
    using System;
    using System.Collections;
    using System.Collections.Generic;
    using System.Data.Entity;
    using System.Linq;
    using System.Linq.Expressions;

    public class QueryableCollection<T>
        : ICollection<T>, IQueryable<T>, IHasIsLoaded
    {
        private readonly ICollection<T> _collection;
        private readonly IQueryable<T> _query;

        public QueryableCollection(ICollection<T> collection, IQueryable<T> query)
        {
            _collection = collection ?? new HashSet<T>();
            _query = query;
        }

        public IQueryable<T> Query
        {
            get { return _query; }
        }

        public bool IsLoaded { get; set; }

        public void Add(T item)
        {
            _collection.Add(item);
        }

        public void Clear()
        {
            LazyLoad();
            _collection.Clear();
        }

        public bool Contains(T item)
        {
            LazyLoad();
            return _collection.Contains(item);
        }

        public void CopyTo(T[] array, int arrayIndex)
        {
            LazyLoad();
            _collection.CopyTo(array, arrayIndex);
        }

        public int Count
        {
            get
            {
                return IsLoaded ? _collection.Count : _query.Count();
            }
        }

        public bool IsReadOnly
        {
            get
            {
                return _collection.IsReadOnly;
            }
        }

        public bool Remove(T item)
        {
            LazyLoad();
            return _collection.Remove(item);
        }

        public IEnumerator<T> GetEnumerator()
        {
            LazyLoad();
            return _collection.GetEnumerator();
        }

        IEnumerator IEnumerable.GetEnumerator()
        {
            // Can't call LazyLoad here due to bug in EF, but usually when writing
            // code the generic enumerator is called anyway.
            return ((IEnumerable)_collection).GetEnumerator();
        }

        private void LazyLoad()
        {
            if (!IsLoaded)
            {
                IsLoaded = true;
                _query.Load();
            }
        }

        Expression IQueryable.Expression
        {
            get { return _query.Expression; }
        }

        Type IQueryable.ElementType
        {
            get { return _query.ElementType; }
        }

        IQueryProvider IQueryable.Provider
        {
            get { return _query.Provider; }
        }
    }
}
```
<p>And here are an interface and extension methods for using IsLoaded:</p>

``` c#
namespace LazyUnicorns
{
    public interface IHasIsLoaded
    {
        bool IsLoaded { get; set; }
    }
}

namespace LazyUnicorns
{
    using System.Collections.Generic;

    public static class CollectionExtensions
    {
        public static bool IsLoaded<T>(this ICollection<T> collection)
        {
            var asHasIsLoaded = collection as IHasIsLoaded;
            return asHasIsLoaded != null ? asHasIsLoaded.IsLoaded : true;
        }

        public static void SetLoaded<T>(this ICollection<T> collection, bool isLoaded)
        {
            var asHasIsLoaded = collection as IHasIsLoaded;
            if (asHasIsLoaded != null)
            {
                asHasIsLoaded.IsLoaded = isLoaded;
            }
        }
    }
}
```

<p>For QueryableCollection, instead of passing a DbCollectionEntry to the constructor we now pass an IQueryable<T>. We can then use this query directly in the Count property.</p>

<p>In LazyCountCollection we were also using the Load method of DbCollectionEntry to do lazy loading of the collection. This has been replaced with a call to the Load extension method on the query. This extension method is defined in the System.Data.Entity namespace (and lives in EntityFramework.dll) but is not actually tied to the Entity Framework in any other way. It does is the same thing as ToList but without actually creating the list. If you wanted to remove all uses of EF from this class you could replace Load with ToList, or write your own Load method.</p>

<h3>Creating QueryableCollections</h3>

<p>We can use QueryableCollection in our entities by creating a new concrete implementation of CachingCollectionInitializer:</p>

``` c#
namespace LazyUnicorns
{
    using System.Collections.Generic;
    using System.Data.Entity.Infrastructure;
    using System.Linq;

    public class QueryableCollectionInitializer : CachingCollectionInitializer
    {
        public override object CreateCollection<TElement>(DbCollectionEntry collectionEntry)
        {
            return new QueryableCollection<TElement>(
                (ICollection<TElement>)collectionEntry.CurrentValue,
                collectionEntry.Query().Cast<TElement>());
        }
    }
}
```
We can create an instance of this initializer and use it with the ObjectMaterialized event just as in our previous posts.</p>

<h3>Some Tests…</h3>

<p>All of the existing tests we had for LazyCountCollection will still pass with this new, more general, implementation. In addition, here are a few more tests that demonstrate some of the things you can do with the new implementation:</p>

``` c#
[TestMethod]
public void QueryableCollection_can_be_used_for_First_without_loading_entire_collection()
{
    using (var context = new BlogContext())
    {
        var post = context.Posts.Find(1);

        var firstComment = post.Comments
            .AsQueryable()
            .OrderBy(c => c.Id)
            .FirstOrDefault();

        Assert.IsNotNull(firstComment);
        Assert.AreEqual(1, context.ChangeTracker.Entries<Comment>().Count());
    }
}

[TestMethod]
public void QueryableCollection_can_be_used_to_load_filtered_results()
{
    using (var context = new BlogContext())
    {
        var post = context.Posts.Find(1);

        var unicornComments = post.Comments
            .AsQueryable()
            .Where(c => c.Content.Contains("unicorn"))
            .ToList();

        Assert.AreEqual(2, unicornComments.Count());
        Assert.AreEqual(2, context.ChangeTracker.Entries<Comment>().Count());
    }
}

[TestMethod]
public void IHasIsLoaded_can_be_used_to_set_IsLoaded_after_a_filtered_query()
{
    using (var context = new BlogContext())
    {
        var post = context.Posts.Find(1);

        post.Comments.AsQueryable()
            .Where(c => c.Content.Contains("unicorn"))
            .Load();
        post.Comments.SetLoaded(true);

        Assert.AreEqual(2, post.Comments.Count); // Doesn't trigger further loading
        Assert.AreEqual(2, context.ChangeTracker.Entries<Comment>().Count());
    }
}
```

<h3>Using the IQueryable</h3>

<p>The first test shows how to use LINQ methods such as FirstOrDefault or Any on the collection. The key to this is the use of AsQueryable. Normally LINQ will always perform LINQ to Objects queries against anything statically typed as ICollection<T>. This ensures that LINQ to Objects is very efficient when working against in-memory collections. However, you can force explicit use of the IQueryable<T> implementation using the AsQueryable method.</p>

<p>Our implementation of IQueryable<T> simply delegates to the underlying query that we set in the constructor. This causes the query to treated as a LINQ to Entities query and it will be evaluated on the server without loading entities into the collection first. Will this work for everything? Probably not since the relationship between the query provider and the query is a bit more complicated that this implementation implies. But it works well enough and allows us to do things like load the first item of the collection without loading the entire collection into memory.</p>

<p>Note that if the collection navigation property on the entity is set to an instance of a regular ICollection<T> implementation (such as might be present when doing unit testing with mocks or fakes) then AsQueryable will perform LINQ to Objects against this collection in the normal way. This is usually exactly what you want for unit testing where the data is all in-memory anyway and should not be going through any data access layer.</p>

<h3>Load with filtering</h3>

<p>The other two tests shows how to load only a filtered set of entities into the collection. This is one of the common usages of DbCollectionEntry.Query in EF 4.1. However, putting the implementation in the collection class has two big advantages over using DbCollectionEntry directly:</p>

<ul>
  <li>The implementation becomes decoupled from the Entity Framework such that the code doing the filtering is now not explicitly aware that it is doing so through EF. This is a form of persistence ignorance.</li>

  <li>Since the IsLoaded flag is now part of the collection we can decide in app code that this collection now has everything loaded that needs to be loaded and lazy loading should therefore not be triggered again.</li>
</ul>

<p>In the first test we load a filtered set of entities using a simple LINQ query and use them immediately using the ToList method.</p>

<p>In the second test we use Load instead of ToList such that the entities are loaded into the context (and hence the collection) and then set the IsLoaded flag to let the collection know that everything we want is now loaded. (As mentioned above, ToList would work just as well as Load here although it would create a List object that we would immediately throw away.)</p>

<h3>IsLoaded and SetLoaded extensions</h3>

<p>ICollection<T> doesn’t have the concept of an IsLoaded flag so we can’t just set the flag directly on our collection. We could check the type, cast, and set, but that requires a bunch of code whenever we want to set IsLoaded. A cleaner approach is to define extension methods for IsLoaded and SetLoaded on ICollection<T>.</p>

<p>The extension methods check whether or not the collection implementation have an IsLoaded flag and use if if they have one, otherwise do nothing. This can be quite useful if you are creating mock or fake entities that are not instances of QueryableCollection but probably already have any in-memory data in the collection such that loading is not needed.</p>

<p>The check for whether or not a collection has an IsLoaded flag could use Reflection (or dynamic methods) if necessary, but to demonstrate the idea I chose just to have any collection that has IsLoaded implement an IHasIsLoaded interface.</p>


<h3>Summary</h3>

<p>In this post we saw how to generalize the extra-lazy collection class to do more with an underlying IQueryable. We showed how this could be used to perform LINQ on the underlying database query and how to use those LINQ queries and the IsLoaded flag to load filtered sets of entities into the collection.</p>
