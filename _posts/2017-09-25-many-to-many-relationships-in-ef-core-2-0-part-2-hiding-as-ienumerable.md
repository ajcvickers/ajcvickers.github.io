---
layout: default
title: "Many-to-many relationships in EF Core 2.0 - Part 2: Hiding as IEnumerable"
date: 2017-09-25 09:50
day: 25th
month: September
year: 2017
author: ajcvickers
permalink: 2017/09/25/many-to-many-relationships-in-ef-core-2-0-part-2-hiding-as-ienumerable/
---

# EF Core 2.0
# Many-to-many relationships
# Part 2: Hiding 'as IEnumerable'

In the <a href="/2017/09/25/many-to-many-relationships-in-ef-core-2-0-part-1-the-basics/">previous post</a> we looked at how many-to-many relationships can be mapped using a join entity. In this post we'll make the navigation properties to the join entity private so that they don't appear in the public surface of our entity types. We'll then add public IEnumerable properties that expose the relationship for reading without reference to the join entity.

<h2>Updating the model</h2>

In the first post our entity types that look like this:

``` c#
public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }

    public ICollection<PostTag> PostTags { get; } = new List<PostTag>();
}

public class Tag
{
    public int TagId { get; set; }
    public string Text { get; set; }

    public ICollection<PostTag> PostTags { get; } = new List<PostTag>();
}
```

But really we want our entity types to look more like this:

``` c#
public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }

    public ICollection<Tag> Tags { get; } = new List<Tag>();
}

public class Tag
{
    public int TagId { get; set; }
    public string Text { get; set; }

    public ICollection<Post> Posts { get; } = new List<Post>();
}
```

One way to do this is to make the PostTags navigation properties private and add public IEnumerable projections for their contents. For example:

``` c#
public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }

    private ICollection<PostTag> PostTags { get; } = new List<PostTag>();

    [NotMapped]
    public IEnumerable<Tag> Tags => PostTags.Select(e => e.Tag);
}

public class Tag
{
    public int TagId { get; set; }
    public string Text { get; set; }

    private ICollection<PostTag> PostTags { get; } = new List<PostTag>();

    [NotMapped]
    public IEnumerable<Post> Posts => PostTags.Select(e => e.Post);
}
```

<h2>Configuring the relationship</h2>

Making the navigation properties private presents a few problems. First, EF Core doesn't pick up private navigations by convention, so they need to be explicitly configured:

``` c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<PostTag>()
        .HasKey(t => new { t.PostId, t.TagId });

    modelBuilder.Entity<PostTag>()
        .HasOne(pt => pt.Post)
        .WithMany("PostTags");

    modelBuilder.Entity<PostTag>()
        .HasOne(pt => pt.Tag)
        .WithMany("PostTags");
}
```

<h1>Using Include</h1>

Next, the Include call can no longer easily access to the private properties using an expression, so we use the string-based API instead:

``` c#
var posts = context.Posts
    .Include("PostTags.Tag")
    .ToList();
```

Notice here that we can't just Include tags like this:

``` c#
var posts = context.Posts
    .Include(e => e.Tags) // Won't work
    .ToList();
```

This is because EF has no knowledge of "Tags"--it is not mapped. EF only knows about the private PostTags navigation property. This is one of the limitations I called out in Part 1. It would currently require messing with EF internals to be able to use Tags directly in queries.

<h2>Using the projected navigation properties</h2>

Reading the many-to-many relationship can now use the public properties directly. For example:

``` c#
foreach (var tag in post.Tags)
{
    Console.WriteLine($"Tag {tag.Text}");
}
```

But if we want to add and remove Tags we still need to do it using the PostTag join entity. We will address this in Part 3, but for now we can add a simple helper that gets PostTags by Reflection. Updating our test application to use this we get:

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
            var posts = LoadAndDisplayPosts(context, "as added");

            posts.Add(context.Add(new Post { Title = "Going to Red Robin" }).Entity);

            var newTag1 = new Tag { Text = "Sweet" };
            var newTag2 = new Tag { Text = "Buzz" };

            foreach (var post in posts)
            {
                var oldPostTag = GetPostTags(post).FirstOrDefault(e => e.Tag.Text == "Pineapple");
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

    private static ICollection<PostTag> GetPostTags(object entity)
        => (ICollection<PostTag>)entity
            .GetType()
            .GetRuntimeProperties()
            .Single(e => e.Name == "PostTags")
            .GetValue(entity);
}
```

This test code will be simplified significantly in the next post where we show how to make the projected navigations updatable.
