---
layout: default
title: LazyCountCollection with Better Performance
date: 2011-03-29 20:28
day: 29th
month: March
year: 2011
author: ajcvickers
permalink: 2011/03/29/lazycountcollection-with-better-performance/
---

# Entity Framework 4.1
# LazyCountCollection with better performance

<p>In my previous two posts I showed how to implement an extra-lazy Count property using EF 4.1. However, the code required that a lot of .NET Reflection happen for every entity returned by a query. The real-world performance impact of this can vary greatly, but for an application that queries for a lot of entities it <em>could</em> be significant.</p>  <p>Before going any further I should say that I haven't profiled any of this, but we frequently write code that does this kind of work inside the Entity Framework and experience has shown that it often becomes a performance bottleneck. That's why we always look for ways to make it faster and this usually comes down to doing the work once and then caching the results for every subsequent use. That's what I am going to show here.</p><p>All the Reflection code is contained in the collection initializer classes—the actual LazyCountCollection will remain unchanged. Here's the new code for the initializer classes:</p>  

``` c#
namespace LazyUnicorns
{
    using System;
    using System.Collections.Concurrent;
    using System.Collections.Generic;
    using System.Data.Entity;
    using System.Data.Entity.Infrastructure;
    using System.Linq;
    using System.Reflection;

    public abstract class CachingCollectionInitializer
    {
        private static readonly
            ConcurrentDictionary<Type, IList<Tuple<string, Func<DbCollectionEntry, object>>>> Factories
            = new ConcurrentDictionary<Type, IList<Tuple<string, Func<DbCollectionEntry, object>>>>();

        private static readonly MethodInfo FactoryMethodInfo
            = typeof(CachingCollectionInitializer).GetMethod("CreateCollection");

        public virtual Type TryGetElementType(PropertyInfo collectionProperty)
        {
            // We can only replace properties that are declared as ICollection<T> and have a setter.
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
            var factories = Factories.GetOrAdd(entity.GetType(), t =>
            {
                var list = new List<Tuple<string, Func<DbCollectionEntry, object>>>();

                foreach (var property in t.GetProperties())
                {
                    var collectionEntry = context.Entry(entity)
                        .Member(property.Name) as DbCollectionEntry;
                    
                    if (collectionEntry != null)
                    {
                        var elementType = TryGetElementType(property);
                        if (elementType != null)
                        {
                            list.Add(Tuple.Create(property.Name,
                                     CreateCollectionFactory(elementType)));
                        }
                    }
                }

                return list;
            });

            foreach (var factory in factories)
            {
                var collectionEntry = context.Entry(entity).Collection(factory.Item1);
                collectionEntry.CurrentValue = factory.Item2(collectionEntry);
            }

        }

        public virtual Func<DbCollectionEntry, object> CreateCollectionFactory(Type elementType)
        {
            return (Func<DbCollectionEntry, object>)Delegate.CreateDelegate(
                typeof(Func<DbCollectionEntry, object>), this,
                FactoryMethodInfo.MakeGenericMethod(elementType));
        }

        public abstract object CreateCollection<TElement>(DbCollectionEntry collectionEntry);
    }
}

namespace LazyUnicorns
{
    using System.Collections.Generic;
    using System.Data.Entity.Infrastructure;

    public class CachingLazyCountCollectionInitializer : CachingCollectionInitializer
    {
        public override object CreateCollection<TElement>(DbCollectionEntry collectionEntry)
        {
            return new LazyCountCollection<TElement>(
                (ICollection<TElement>)collectionEntry.CurrentValue, collectionEntry);
        }
    }
}
```

<h3>App-domain cache with ConcurrentDictionary</h3>

<p>So what's going one here? The first thing to notice is the static ConcurrentDictionary. This is the app-domain cache that maps from an entity type to the delegates that we'll use to create the LazyCountCollection instances. It is a ConcurrentDictionary because this code must be thread safe and ConcurrentDictionary (used correctly) is one of the best ways to get thread safety.</p>

<p>Next look at the InitializeCollections method. This method uses GetOrAdd on the ConcurrentDictionary to either get the cached list of delegates for the entity type or create it if it doesn't already exist. (Technically, GetOrAdd could result in the list being created more than once, but only one instance will make it into the dictionary and be cached. This doesn't compromise the thread safety and in practice doesn't impact performance—see the ConcurrentDictionary documentation for more information.)</p>

<h3>Creating delegates with CreateDelegate</h3>

<p>When creating the delegates we use pretty much the same code as before to figure out whether or not the type has a property that we want to wrap. Where the code changes again is how we create the instance of LazyCountCollection. We now don't want to create the instance immediately but rather create a delegate that can be used to create an instance whenever we need one. The CreateCollectionFactory method is responsible for creating this delegate.</p>

<p>The .NET Framework has a bunch of ways to create delegates at runtime. You can use Lightweight Code Gen, but this requires manipulating IL and is very easy to get wrong. You can get away from IL by creating expression trees and calling CompileDelegate. This is great for more complex dynamic code. However, for a simple delegate like we need here it is hard to beat the good old CreateDelegate method. It's fast, doesn't require writing IL or expression trees, and has less chance of causing partial trust issues.</p>

<p>What we really want to do here is create a delegate for the LazyCountCollection constructor. CreateDelegate doesn't allow that, but it does allow creation of a delegate for a simple factory method that will call the constructor. That's what the abstract CreateCollection method is for. CreateCollection is overridden in CachingLazyCountCollectionInitializer to return an instance of LazyCountCollection for the appropriate element type.</p>

<h3>Using the delegates</h3>

<p>Once InitializeCollections has a list of delegates (either because we just created them or because we got them from the cache) it can loop over each collection in the entity and use the delegate to replace it without using any Reflection, which is exactly what we need.</p>

<p>Note that the model and tests from the previous post can be used with this new code simply by replacing references to LazyCountCollectionInitializer with reference to CachingLazyCountCollectionInitializer.</p>

<h3>Summary</h3>

<p>In this post I showed how to use ConcurrentDictionary and CreateDelegate to create an app-domain cache that eliminates most of the performance issues that come from using Reflection to create and set LazyCountCollection instances.</p>
