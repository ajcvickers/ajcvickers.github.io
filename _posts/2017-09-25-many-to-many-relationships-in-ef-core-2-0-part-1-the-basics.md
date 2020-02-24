---
layout: default
title: "Many-to-many relationships in EF Core 2.0 - Part 1: The basics"
date: 2017-09-25 09:49
day: 25th
month: September
year: 2017
author: ajcvickers
permalink: 2017/09/25/many-to-many-relationships-in-ef-core-2-0-part-1-the-basics/
---

# EF Core 2.0
# Many-to-many relationships
# Part 1: The basics

As of EF Core 2.0, many-to-many relationships without an explicitly mapped join table are not supported. However, all is not lost. In this series of posts I will show:

<ul>
<li>Mapping many-to-many relationships with a join entity/table</li>
<li>Abstracting/hiding the join entity

<ul>
<li>In a simple way for read-only access to the relationship</li>
<li>In a more involved way that allows entities to be added and removed from each end</li>
</ul></li>
</ul>

<h2>Limitations</h2>

Before going any further I want to be clear about two limitations with the approach used in all these posts:

<ul>
<li>EF Core doesn't know about un-mapped properties. This means that queries must still be written in terms of the join entity.</li>
<li>The join entity is not gone; it's still in application code and still mapped. It's just that normal interactions with the model, such as those in your application, do not use it.</li>
</ul>

Addressing these limitations requires changes to the internals of EF Core. I don't expect the limitations to go away until at least some parts of <a href="https://github.com/aspnet/EntityFrameworkCore/issues/1368">GitHub issue 1368</a> are implemented.

<h2>The model</h2>

A good example of a many-to-many relationship is blogs and tags. Every blog can have many tags, and every tag can be associated with many blogs. A typical way to model this is:

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

<h2>Modeling with a join entity</h2>

This cannot be mapped directly using foreign keys--each post would need multiple FK values for each tag, and vice-versa. Instead, another entity type is needed to bridge the gap and hold all the FK pairs. In a relational database this is often called a "join table", and we will map it to a join entity:

``` c#
public class PostTag
{
    public int PostId { get; set; }
    public Post Post { get; set; }

    public int TagId { get; set; }
    public Tag Tag { get; set; }
}
```

This entity type has two FKs each associated with a navigation property one pointing to one side of the relationship (Post) and the other to the other side of the relationship (Tag). When we want to associate a Post with a Tag, we create a new PostTag instance and set the navigation properties to point to the Post and the Tag. Equivalently, we could set the PostId FK to the PK value of the Post and the TagId FK to the PK value to the Tag.

To use this join entity we need to update the original entity types to map through it:

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

<h2>Configuring the join entity type</h2>

The join entity type needs to have a key defined since there is no key that EF can figure out by convention. Since pairs of PostId and TagId values are unique, we can use these pairs as a composite key for the entity:

``` c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<PostTag>()
        .HasKey(t => new { t.PostId, t.TagId });
}
```

The actual relationships don't need to be configured explicitly in this case because they can be figured out by convention.

<h2>Using the many-to-many relationship</h2>

Let's write a little console application to show this working. First, we need a DbContext:

``` c#
public class MyContext : DbContext
{
    public DbSet<Post> Posts { get; set; }
    public DbSet<Tag> Tags { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder.UseSqlServer(
            @"Server=(localdb)\mssqllocaldb;Database=Test;ConnectRetryCount=0");

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<PostTag>()
            .HasKey(t => new { t.PostId, t.TagId });
    }
}
```

I'm using SQL Server LocalDb as a database provider, but this code should work the same with any database provider.

Next, here is a little test application that manipulates the many-to-many relationship in various ways:

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
                var oldPostTag = post.PostTags.FirstOrDefault(e => e.Tag.Text == "Pineapple");
                if (oldPostTag != null)
                {
                    post.PostTags.Remove(oldPostTag);
                    post.PostTags.Add(new PostTag { Post = post, Tag = newTag1 });
                }
                post.PostTags.Add(new PostTag { Post = post, Tag = newTag2 });
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
            .Include(e => e.PostTags)
            .ThenInclude(e => e.Tag)
            .ToList();

        foreach (var post in posts)
        {
            Console.WriteLine($"  Post {post.Title}");
            foreach (var tag in post.PostTags.Select(e => e.Tag))
            {
                Console.WriteLine($"    Tag {tag.Text}");
            }
        }

        Console.WriteLine();

        return posts;
    }
}
```

Summarizing this application, it:

<ul>
<li>Deletes any stale test database and creates a new one</li>
<li>Creates some Posts and some Tags and then creates join entities to associate them</li>
<li>Saves all the Posts and Tags and their relationships to the database</li>
<li>Loads the entities in a new context and displays them</li>
<li>Manipulates the relationships:

<ul>
<li>A new Post is created and tracked</li>
<li>Every "Pineapple" Tag is removed and replaced with a "Sweet" Tag</li>
<li>Every Post gets a new "Buzz" Tag</li>
</ul></li>
<li>These are again saved, re-read, and displayed.</li>
</ul>

On my machine, this results in the following output:

```
Dumping posts as added:
  Post Best Boutiques on the Eastside
    Tag Golden
    Tag Pineapple
  Post Avoiding over-priced Hipster joints
    Tag Girlscout
    Tag Cookies
  Post Where to buy Mars Bars
    Tag Golden
    Tag Pineapple
    Tag Girlscout
    Tag Cookies

Dumping posts after manipulation:
  Post Best Boutiques on the Eastside
    Tag Golden
    Tag Sweet
    Tag Buzz
  Post Avoiding over-priced Hipster joints
    Tag Girlscout
    Tag Cookies
    Tag Buzz
  Post Where to buy Mars Bars
    Tag Golden
    Tag Girlscout
    Tag Cookies
    Tag Sweet
    Tag Buzz
  Post Going to Red Robin
    Tag Buzz

Press any key to continue . . .
```

So that's simple many-to-many relationship with a join entity. In the next post we'll start hiding aspects of the join entity.
