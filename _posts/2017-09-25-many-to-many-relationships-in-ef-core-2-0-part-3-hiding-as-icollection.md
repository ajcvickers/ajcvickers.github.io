---
layout: post
title: Many-to-many relationships in EF Core 2.0 - Part 3: Hiding as ICollection
date: 2017-09-25 09:50
author: ajcvickers
comments: true
categories: [Code First, DbContext, DbContext API, EF Core, Entity Framework, Foreign Keys, Many-to-Many]
---
In the <a href="http://wp.me/p1rP25-7p">previous post</a> we ended up with entities that hide the join entity from the public surface. However, it was not possible to add or removed entities through this public surface. To enable this we need an ICollection implementation that acts as a true facade over the real join entity collection and delegates all responsibilities to that collection.

<!--more-->

<h2>The collection implementation</h2>

Here's one possible implementation of such a collection:

[code lang=csharp]
public class JoinCollectionFacade&lt;T, TJoin&gt; : ICollection&lt;T&gt;
{
    private readonly ICollection&lt;TJoin&gt; _collection;
    private readonly Func&lt;TJoin, T&gt; _selector;
    private readonly Func&lt;T, TJoin&gt; _creator;

    public JoinCollectionFacade(
        ICollection&lt;TJoin&gt; collection,
        Func&lt;TJoin, T&gt; selector,
        Func&lt;T, TJoin&gt; creator)
    {
        _collection = collection;
        _selector = selector;
        _creator = creator;
    }

    public IEnumerator&lt;T&gt; GetEnumerator()
        =&gt; _collection.Select(e =&gt; _selector(e)).GetEnumerator();

    IEnumerator IEnumerable.GetEnumerator()
        =&gt; GetEnumerator();

    public void Add(T item)
        =&gt; _collection.Add(_creator(item));

    public void Clear()
        =&gt; _collection.Clear();

    public bool Contains(T item)
        =&gt; _collection.Any(e =&gt; Equals(_selector(e), item));

    public void CopyTo(T[] array, int arrayIndex)
        =&gt; this.ToList().CopyTo(array, arrayIndex);

    public bool Remove(T item)
        =&gt; _collection.Remove(
            _collection.FirstOrDefault(e =&gt; Equals(_selector(e), item)));

    public int Count
        =&gt; _collection.Count;

    public bool IsReadOnly
        =&gt; _collection.IsReadOnly;
}
[/code]

The idea is pretty simple--operations on the facade are translated into operations on the underlying collection. Where needed, a "selector" delegate is used to extract the desired target entity from the join entity. Likewise, a "creator" delegate creates a new join entity instance from the target entity when a new relationship is added.

<h2>Updating the model</h2>

We need to initialize instances of this collection in our entities:

[code lang=csharp]
public class Post
{
    public Post()
        =&gt; Tags = new JoinCollectionFacade&lt;Tag, PostTag&gt;(
            PostTags,
            pt =&gt; pt.Tag,
            t =&gt; new PostTag { Post = this, Tag = t });

    public int PostId { get; set; }
    public string Title { get; set; }

    private ICollection&lt;PostTag&gt; PostTags { get; } = new List&lt;PostTag&gt;();

    [NotMapped]
    public ICollection&lt;Tag&gt; Tags { get; }
}

public class Tag
{
    public Tag()
        =&gt; Posts = new JoinCollectionFacade&lt;Post, PostTag&gt;(
            PostTags,
            pt =&gt; pt.Post,
            p =&gt; new PostTag { Post = p, Tag = this });

    public int TagId { get; set; }
    public string Text { get; set; }

    private ICollection&lt;PostTag&gt; PostTags { get; } = new List&lt;PostTag&gt;();

    [NotMapped]
    public ICollection&lt;Post&gt; Posts { get; }
}
[/code]

<h2>Using the ICollection navigations</h2>

Notice how Tags and Posts are now ICollection properties instead of IEnumerable properties. This means we can add and remove entities from the many-to-many collections without using the join entity directly. Here's the test application updated to show this:

[code lang=csharp]
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
                new Tag { Text = &quot;Golden&quot; },
                new Tag { Text = &quot;Pineapple&quot; },
                new Tag { Text = &quot;Girlscout&quot; },
                new Tag { Text = &quot;Cookies&quot; }
            };

            var posts = new[]
            {
                new Post { Title = &quot;Best Boutiques on the Eastside&quot; },
                new Post { Title = &quot;Avoiding over-priced Hipster joints&quot; },
                new Post { Title = &quot;Where to buy Mars Bars&quot; }
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
            var posts = LoadAndDisplayPosts(context, &quot;as added&quot;);

            posts.Add(context.Add(new Post { Title = &quot;Going to Red Robin&quot; }).Entity);

            var newTag1 = new Tag { Text = &quot;Sweet&quot; };
            var newTag2 = new Tag { Text = &quot;Buzz&quot; };

            foreach (var post in posts)
            {
                var oldTag = post.Tags.FirstOrDefault(e =&gt; e.Text == &quot;Pineapple&quot;);
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
            LoadAndDisplayPosts(context, &quot;after manipulation&quot;);
        }
    }

    private static List&lt;Post&gt; LoadAndDisplayPosts(MyContext context, string message)
    {
        Console.WriteLine($&quot;Dumping posts {message}:&quot;);

        var posts = context.Posts
            .Include(&quot;PostTags.Tag&quot;)
            .ToList();

        foreach (var post in posts)
        {
            Console.WriteLine($&quot;  Post {post.Title}&quot;);
            foreach (var tag in post.Tags)
            {
                Console.WriteLine($&quot;    Tag {tag.Text}&quot;);
            }
        }

        Console.WriteLine();

        return posts;
    }
}
[/code]

Notice that:

<ul>
<li>When seeding the database, we add Tags directly to the Tags collection on Post.</li>
<li>When finding and removing existing tags, we can search directly for the Tag and remove it from the Post.Tags collection without needing to use the join entity.</li>
</ul>

It's worth calling out again that, just like in the previous post, we still can't use Tags directly in any query. For example, using it for Include won't work:

[code lang=csharp]
var posts = context.Posts
    .Include(e =&gt; e.Tags) // Won&#039;t work
    .ToList();
[/code]

EF has no knowledge of "Tags"--it is not mapped. EF only knows about the private PostTags navigation property.

Functionally, this is about as far as we can go without starting to mess with the internals of EF. However, in one last post I'll show how to abstract out the collection and join entity a bit more so that it is easier reuse for different types.
