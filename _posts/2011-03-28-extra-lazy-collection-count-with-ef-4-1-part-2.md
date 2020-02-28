---
layout: default
title: Extra-Lazy Collection Count with EF 4.1 (Part 2)
date: 2011-03-28 21:50
day: 28th
month: March
year: 2011
author: ajcvickers
permalink: 2011/03/28/extra-lazy-collection-count-with-ef-4-1-part-2/
---

# Entity Framework 4.1
# Extra-lazy collection count with EF 4.1 (Part 2)

<h3>Using LazyCountCollection in a model</h3>  <p>In <a href="/2011/03/28/extra-lazy-collection-count-with-ef-4-1-part-1/">part 1</a> we setup all the infrastructure for implementing an extra-lazy Count property with EF 4.1—now let’s actually use it!</p><p>I’m going to be boring and just use the <em>new Northwind</em>—a blogging model. Here are my entities and context implementation:</p>  

``` c#
namespace LazyUnicornTests.Model
{
    using System.Data.Entity;
    using System.Data.Entity.Infrastructure;
    using LazyUnicorns;

    public class BlogContext : DbContext
    {
        public BlogContext()
        {
            Database.SetInitializer(new BlogsContextInitializer());
            Configuration.LazyLoadingEnabled = false;

            ((IObjectContextAdapter)this).ObjectContext.ObjectMaterialized +=
                (s, e) => new LazyCountCollectionInitializer()
                    .InitializeCollections(this, e.Entity);
        }

        public DbSet<Post> Posts { get; set; }
        public DbSet<Comment> Comments { get; set; }
    }
}

namespace LazyUnicornTests.Model
{
    public class Comment
    {
        public int Id { get; set; }
        public string Content { get; set; }
        public Post Post { get; set; }
    }
}

namespace LazyUnicornTests.Model
{
    using System.Collections.Generic;
    using System.ComponentModel.DataAnnotations;

    public class Post
    {
        [DatabaseGenerated(DatabaseGeneratedOption.None)]
        public int Id { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        public ICollection<Comment> Comments { get; set; }
    }
}
```

<p>The most interesting point to notice about the code above is that the constructor registers an instance of LazyCountCollectionInitializer with the ObjectMaterialized event of the underlying ObjectContext. Notice that the entities themselves just have simple ICollection<T> properties and don’t explicitly reference LazyCountCollection.</p>

<p>Everything else is pretty straightforward EF 4.1 code so I won’t go into more details here.</p>

<h3>Some tests…</h3>

<p>Rather than demonstrate the code through an application I have instead written some tests. The model above is for use with those tests and hence it is setup more as a test model than a real model, including doing things like calling SetInitializer in the context constructor.</p>

<p>The tests are mostly not real unit tests, but rather small functional tests for certain expected behaviors of the code. Real unit tests would not create a real database and execute real queries against it. Nevertheless, this type of small functional test is very easy to write with EF 4.1 and serves as an easy way to test behavior with little code.</p>

<p>The tests make use of a straightforward database initializer that adds some sample blogs and comments when the database is created. The assertions in the tests rely on this well-known data being present in the database. Here is the initializer code:</p>

``` c#
namespace LazyUnicornTests.Model
{
    using System.Collections.Generic;
    using System.Data.Entity;

    public class BlogsContextInitializer : DropCreateDatabaseIfModelChanges<BlogContext>
    {
        protected override void Seed(BlogContext context)
        {
            context.Posts.Add(new Post
            {
                Id = 1,
                Title = "Lazy Unicorns",
                Comments = new List<Comment>
                {
                    new Comment { Content = "Are enums supported?" },
                    new Comment { Content = "My unicorns are so lazy they fell asleep." },
                    new Comment { Content = "Is a unicorn without a horn just a horse?" },
                }
            });

            context.Posts.Add(new Post
            {
                Id = 2,
                Title = "Sleepy Horses",
                Comments = new List<Comment>
                {
                    new Comment { Content = "Are enums supported?" },
                }
            });
        }
    }
}
```
<p>And here are the tests:</p>

``` c#
namespace LazyUnicornTests
{
    using System.Collections.Generic;
    using System.Data.Entity;
    using System.Linq;
    using LazyUnicorns;
    using LazyUnicornTests.Model;
    using Microsoft.VisualStudio.TestTools.UnitTesting;

    [TestClass]
    public class LazyCountCollectionTests
    {
        [TestMethod]
        public void LazyCountCollection_Count_returns_count_without_loading_collection()
        {
            using (var context = new BlogContext())
            {
                var post = context.Posts.Find(1);

                Assert.AreEqual(3, post.Comments.Count);
                Assert.AreEqual(0, context.ChangeTracker.Entries<Comment>().Count());
            }
        }

        [TestMethod]
        public void LazyCountCollection_Count_returns_count_even_when_collection_is_loaded()
        {
            using (var context = new BlogContext())
            {
                var post = context.Posts.Find(1);
                context.Entry(post).Collection(p => p.Comments).Load();

                Assert.AreEqual(3, post.Comments.Count);
                Assert.AreEqual(3, context.ChangeTracker.Entries<Comment>().Count());
            }
        }

        [TestMethod]
        public void LazyCountCollection_Count_returns_database_count_not_collection_count()
        {
            using (var context = new BlogContext())
            {
                var post = context.Posts.Find(1);
                context.Entry(post).Collection(p => p.Comments).Load();
                post.Comments.Add(new Comment());

                Assert.AreEqual(3, post.Comments.Count);
                Assert.AreEqual(4, context.ChangeTracker.Entries<Comment>().Count());
            }
        }

        [TestMethod]
        public void Enumerating_the_LazyCountCollection_causes_it_to_be_lazy_loaded()
        {
            using (var context = new BlogContext())
            {
                context.Posts.Find(1).Comments.ToList();

                Assert.AreEqual(3, context.ChangeTracker.Entries<Comment>().Count());
            }
        }

        [TestMethod]
        public void Adding_to_the_LazyCountCollection_does_not_cause_it_to_be_lazy_loaded()
        {
            using (var context = new BlogContext())
            {
                context.Posts.Find(1).Comments.Add(new Comment());

                Assert.AreEqual(1, context.ChangeTracker.Entries<Comment>().Count());
            }
        }

        [TestMethod]
        public void LazyCountCollection_Count_returns_count_even_when_collection_is_eager_loaded()
        {
            using (var context = new BlogContext())
            {
                var post = context.Posts
                    .Where(p => p.Id == 1)
                    .Include(p => p.Comments)
                    .Single();

                Assert.AreEqual(3, post.Comments.Count);
                Assert.AreEqual(3, context.ChangeTracker.Entries<Comment>().Count());
            }
        }

        public class FakeEntityWithListCollection
        {
            public List<Post> Posts { get; set; }
        }

        [TestMethod]
        public void Collections_not_declared_as_ICollection_are_ignored()
        {
            Assert.IsNull(new LazyCountCollectionInitializer()
                .TryGetElementType(typeof(FakeEntityWithListCollection)
                .GetProperty("Posts")));
        }

        public class FakeEntityWithReadonlyCollection
        {
            public ICollection<Post> Posts { get { return null; } }
        }

        [TestMethod]
        public void Collections_without_setters_are_ignored()
        {
            Assert.IsNull(new LazyCountCollectionInitializer()
                .TryGetElementType(typeof(FakeEntityWithReadonlyCollection)
                .GetProperty("Posts")));
        }

    }
}

```

<h3>Potential problems with this code</h3>

<p>So that’s pretty much it…but before I end I’d like to address a few potential problems with the code above. First, this code is not production ready for two main reasons: first because it is not tested. I know there are some tests up there, but they are nowhere near comprehensive enough to say that this code really works. All we can say is that it <em>seems </em>to work for the simple cases that we have thrown at it. (In fact, I have spotted one bug already…)</p>

<p>The second reason I would not consider this code production-ready is all the uncached use of Reflection. I haven’t profiled it, but I have a hunch that this code will result in a performance bottleneck in some apps. In short, use at your own risk.</p>

<p>Another potential problem with this code is that it gives Count somewhat strange semantics. Let’s say you add three entities to the collection and then call Count. The result could be less than three, three, or more than three. This might be fine if you know that this is the way Count works for the collection, but it is pretty strange when compared to the semantics of Count on normal collections.</p>

<p>Replacing the collection that an entity thinks it has with an instance of a different type can also cause problems. The entity (and the rest of the app) must know not to reset the collection at any point or the wrapped collection will be replaced. Also, if you try to serialize the entity you are likely to run into problems because it will attempt to serialize a LazyLoadCollection instance, which probably won’t work. Even if it does work, it’s probably not what you want. I’m sure that there are other problems with replacing the collection that I haven’t thought of…</p>

<p>And finally, the collection is only replaced when the ObjectMaterialized event is fired. This means that, for example, if you were to create an entity and attach it to a context then it wouldn’t have a LazyLoadCollection. Of course, you could write an Attach method that would attach and then replace the collection if you really needed this to work.</p>

<h3>Summary</h3>

<p>In these posts we looked at a simple way to implement collections with extra-lazy behavior using EF 4.1. Specifically, we implemented a collection with an extra-lazy Count property.</p>

<p>In upcoming posts I’ll show how to address the potential performance issue from the Reflection code and how to generalize the wrapped collection to have more than just an extra-lazy Count property.</p>
