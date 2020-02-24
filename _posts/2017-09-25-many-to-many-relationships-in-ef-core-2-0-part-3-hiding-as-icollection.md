---
layout: default
title: "Many-to-many relationships in EF Core 2.0 - Part 3: Hiding as ICollection"
date: 2017-09-25 09:50
day: 25th
month: September
year: 2017
author: ajcvickers
permalink: 2017/09/25/many-to-many-relationships-in-ef-core-2-0-part-3-hiding-as-icollection/
---

# EF Core 2.0
# Many-to-many relationships
# Part 3: Hiding 'as ICollection'

In the <a href="/2017/09/25/many-to-many-relationships-in-ef-core-2-0-part-2-hiding-as-ienumerable/">previous post</a> we ended up with entities that hide the join entity from the public surface. However, it was not possible to add or removed entities through this public surface. To enable this we need an ICollection implementation that acts as a true facade over the real join entity collection and delegates all responsibilities to that collection.

<h2>The collection implementation</h2>

Here's one possible implementation of such a collection:

``` c#
public class JoinCollectionFacade<T, TJoin> : ICollection<T>
{
    private readonly ICollection<TJoin> _collection;
    private readonly Func<TJoin, T> _selector;
    private readonly Func<T, TJoin> _creator;

    public JoinCollectionFacade(
        ICollection<TJoin> collection,
        Func<TJoin, T> selector,
        Func<T, TJoin> creator)
    {
        _collection = collection;
        _selector = selector;
        _creator = creator;
    }

    public IEnumerator<T> GetEnumerator()
        => _collection.Select(e => _selector(e)).GetEnumerator();

    IEnumerator IEnumerable.GetEnumerator()
        => GetEnumerator();

    public void Add(T item)
        => _collection.Add(_creator(item));

    public void Clear()
        => _collection.Clear();

    public bool Contains(T item)
        => _collection.Any(e => Equals(_selector(e), item));

    public void CopyTo(T[] array, int arrayIndex)
        => this.ToList().CopyTo(array, arrayIndex);

    public bool Remove(T item)
        => _collection.Remove(
            _collection.FirstOrDefault(e => Equals(_selector(e), item)));

    public int Count
        => _collection.Count;

    public bool IsReadOnly
        => _collection.IsReadOnly;
}
```

The idea is pretty simple--operations on the facade are translated into operations on the underlying collection. Where needed, a "selector" delegate is used to extract the desired target entity from the join entity. Likewise, a "creator" delegate creates a new join entity instance from the target entity when a new relationship is added.

<h2>Updating the model</h2>

We need to initialize instances of this collection in our entities:

``` c#
public class Post
{
    public Post()
        => Tags = new JoinCollectionFacade<Tag, PostTag>(
            PostTags,
            pt => pt.Tag,
            t => new PostTag { Post = this, Tag = t });

    public int PostId { get; set; }
    public string Title { get; set; }

    private ICollection<PostTag> PostTags { get; } = new List<PostTag>();

    [NotMapped]
    public ICollection<Tag> Tags { get; }
}

public class Tag
{
    public Tag()
        => Posts = new JoinCollectionFacade<Post, PostTag>(
            PostTags,
            pt => pt.Post,
            p => new PostTag { Post = p, Tag = this });

    public int TagId { get; set; }
    public string Text { get; set; }

    private ICollection<PostTag> PostTags { get; } = new List<PostTag>();

    [NotMapped]
    public ICollection<Post> Posts { get; }
}
```

<h2>Using the ICollection navigations</h2>

Notice how Tags and Posts are now ICollection properties instead of IEnumerable properties. This means we can add and remove entities from the many-to-many collections without using the join entity directly. Here's the test application updated to show this:

``` c#
public class Program
{
    public static void Main()
    {
        using (var context = new MyContext())
        {
            context.Database.EnsureDeleted();
            context.Database.EnsureCreated();

            var tags = new[]
            {
                new Tag { Text = "Golden" },
                new Tag { Text = "Pineapple" },
                new Tag { Text = "Girlscout" },
                new Tag { Text = "Cookies" }
            };

            var posts = new[]
            {
                new Post { Title = "Best Boutiques on the Eastside" },
                new Post { Title = "Avoiding over-priced Hipster joints" },
                new Post { Title = "Where to buy Mars Bars" }
            };

            posts[0].Tags.Add(tags[0]);
            posts[0].Tags.Add(tags[1]);
            posts[1].Tags.Add(tags[2]);
            posts[1].Tags.Add(tags[3]);
            posts[2].Tags.Add(tags[0]);
            posts[2].Tags.Add(tags[1]);
            posts[2].Tags.Add(tags[2]);
            posts[2].Tags.Add(tags[3]);

            context.AddRange(tags);
            context.AddRange(posts);

            context.SaveChanges();
        }

        using (var context = new MyContext())
        {
            var posts = LoadAndDisplayPosts(context, "as added");

            posts.Add(context.Add(new Post { Title = "Going to Red Robin" }).Entity);

            var newTag1 = new Tag { Text = "Sweet" };
            var newTag2 = new Tag { Text = "Buzz" };

            foreach (var post in posts)
            {
                var oldTag = post.Tags.FirstOrDefault(e => e.Text == "Pineapple");
                if (oldTag != null)
                {
                    post.Tags.Remove(oldTag);
                    post.Tags.Add(newTag1);
                }
                post.Tags.Add(newTag2);
            }

            context.SaveChanges();
        }

        using (var context = new MyContext())
        {
            LoadAndDisplayPosts(context, "after manipulation");
        }
    }

    private static List<Post> LoadAndDisplayPosts(MyContext context, string message)
    {
        Console.WriteLine($"Dumping posts {message}:");

        var posts = context.Posts
            .Include("PostTags.Tag")
            .ToList();

        foreach (var post in posts)
        {
            Console.WriteLine($"  Post {post.Title}");
            foreach (var tag in post.Tags)
            {
                Console.WriteLine($"    Tag {tag.Text}");
            }
        }

        Console.WriteLine();

        return posts;
    }
}
```

Notice that:

<ul>
<li>When seeding the database, we add Tags directly to the Tags collection on Post.</li>
<li>When finding and removing existing tags, we can search directly for the Tag and remove it from the Post.Tags collection without needing to use the join entity.</li>
</ul>

It's worth calling out again that, just like in the previous post, we still can't use Tags directly in any query. For example, using it for Include won't work:

``` c#
var posts = context.Posts
    .Include(e => e.Tags) // Won't work
    .ToList();
```

EF has no knowledge of "Tags"--it is not mapped. EF only knows about the private PostTags navigation property.

Functionally, this is about as far as we can go without starting to mess with the internals of EF. However, in one last post I'll show how to abstract out the collection and join entity a bit more so that it is easier reuse for different types.
