---
layout: post
title: Many-to-many relationships in EF Core 2.0 - Part 2: Hiding as IEnumerable
date: 2017-09-25 09:50
author: ajcvickers
comments: true
categories: [Code First, DbContext, DbContext API, EF Core, Entity Framework, Foreign Keys]
---
In the <a href="http://wp.me/p1rP25-7p">previous post</a> we looked at how many-to-many relationships can be mapped using a join entity. In this post we'll make the navigation properties to the join entity private so that they don't appear in the public surface of our entity types. We'll then add public IEnumerable properties that expose the relationship for reading without reference to the join entity.

<!--more-->

<h2>Updating the model</h2>

In the first post our entity types that look like this:

[code lang=csharp]
public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }

    public ICollection&lt;PostTag&gt; PostTags { get; } = new List&lt;PostTag&gt;();
}

public class Tag
{
    public int TagId { get; set; }
    public string Text { get; set; }

    public ICollection&lt;PostTag&gt; PostTags { get; } = new List&lt;PostTag&gt;();
}
[/code]

But really we want our entity types to look more like this:

[code lang=csharp]
public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }

    public ICollection&lt;Tag&gt; Tags { get; } = new List&lt;Tag&gt;();
}

public class Tag
{
    public int TagId { get; set; }
    public string Text { get; set; }

    public ICollection&lt;Post&gt; Posts { get; } = new List&lt;Post&gt;();
}
[/code]

One way to do this is to make the PostTags navigation properties private and add public IEnumerable projections for their contents. For example:

[code lang=csharp]
public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }

    private ICollection&lt;PostTag&gt; PostTags { get; } = new List&lt;PostTag&gt;();

    [NotMapped]
    public IEnumerable&lt;Tag&gt; Tags =&gt; PostTags.Select(e =&gt; e.Tag);
}

public class Tag
{
    public int TagId { get; set; }
    public string Text { get; set; }

    private ICollection&lt;PostTag&gt; PostTags { get; } = new List&lt;PostTag&gt;();

    [NotMapped]
    public IEnumerable&lt;Post&gt; Posts =&gt; PostTags.Select(e =&gt; e.Post);
}
[/code]

<h2>Configuring the relationship</h2>

Making the navigation properties private presents a few problems. First, EF Core doesn't pick up private navigations by convention, so they need to be explicitly configured:

[code lang=csharp]
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity&lt;PostTag&gt;()
        .HasKey(t =&gt; new { t.PostId, t.TagId });

    modelBuilder.Entity&lt;PostTag&gt;()
        .HasOne(pt =&gt; pt.Post)
        .WithMany(&quot;PostTags&quot;);

    modelBuilder.Entity&lt;PostTag&gt;()
        .HasOne(pt =&gt; pt.Tag)
        .WithMany(&quot;PostTags&quot;);
}
[/code]

<h1>Using Include</h1>

Next, the Include call can no longer easily access to the private properties using an expression, so we use the string-based API instead:

[code lang=csharp]
var posts = context.Posts
    .Include(&quot;PostTags.Tag&quot;)
    .ToList();
[/code]

Notice here that we can't just Include tags like this:

[code lang=csharp]
var posts = context.Posts
    .Include(e =&gt; e.Tags) // Won&#039;t work
    .ToList();
[/code]

This is because EF has no knowledge of "Tags"--it is not mapped. EF only knows about the private PostTags navigation property. This is one of the limitations I called out in Part 1. It would currently require messing with EF internals to be able to use Tags directly in queries.

<h2>Using the projected navigation properties</h2>

Reading the many-to-many relationship can now use the public properties directly. For example:

[code lang=csharp]
foreach (var tag in post.Tags)
{
    Console.WriteLine($&quot;Tag {tag.Text}&quot;);
}
[/code]

But if we want to add and remove Tags we still need to do it using the PostTag join entity. We will address this in Part 3, but for now we can add a simple helper that gets PostTags by Reflection. Updating our test application to use this we get:

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

            context.AddRange(
                new PostTag { Post = posts[0], Tag = tags[0] },
                new PostTag { Post = posts[0], Tag = tags[1] },
                new PostTag { Post = posts[1], Tag = tags[2] },
                new PostTag { Post = posts[1], Tag = tags[3] },
                new PostTag { Post = posts[2], Tag = tags[0] },
                new PostTag { Post = posts[2], Tag = tags[1] },
                new PostTag { Post = posts[2], Tag = tags[2] },
                new PostTag { Post = posts[2], Tag = tags[3] });

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
                var oldPostTag = GetPostTags(post).FirstOrDefault(e =&gt; e.Tag.Text == &quot;Pineapple&quot;);
                if (oldPostTag != null)
                {
                    GetPostTags(post).Remove(oldPostTag);
                    GetPostTags(post).Add(new PostTag { Post = post, Tag = newTag1 });
                }
                GetPostTags(post).Add(new PostTag { Post = post, Tag = newTag2 });
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

    private static ICollection&lt;PostTag&gt; GetPostTags(object entity)
        =&gt; (ICollection&lt;PostTag&gt;)entity
            .GetType()
            .GetRuntimeProperties()
            .Single(e =&gt; e.Name == &quot;PostTags&quot;)
            .GetValue(entity);
}
[/code]

This test code will be simplified significantly in the next post where we show how to make the projected navigations updatable.
