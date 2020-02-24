---
layout: default
title: "Many-to-many relationships in EF Core 2.0 - Part 4: A more general abstraction"
date: 2017-09-25 09:50
day: 25th
month: September
year: 2017
author: ajcvickers
permalink: 2017/09/25/many-to-many-relationships-in-ef-core-2-0-part-4-a-more-general-abstraction/
---

# EF Core 2.0
# Many-to-many relationships
# Part 4: A more general abstraction

In the <a href="/2017/09/25/many-to-many-relationships-in-ef-core-2-0-part-3-hiding-as-icollection/">last few posts</a> we saw how to hide use of the join entity from two entities with a many-to-many relationship. This post doesn't add any additional functionality, it just abstracts some of what we saw so it can be re-used more easily.

<!--more-->

To start with we define an interface for join entities:

``` c#
public interface IJoinEntity<TEntity>
{
    TEntity Navigation { get; set; }
}
```

Any join entity will implement this interface twice; once for each side:

``` c#
public class PostTag : IJoinEntity<Post>, IJoinEntity<Tag>
{
    public int PostId { get; set; }
    public Post Post { get; set; }
    Post IJoinEntity<Post>.Navigation
    {
        get => Post;
        set => Post = value;
    }

    public int TagId { get; set; }
    public Tag Tag { get; set; }
    Tag IJoinEntity<Tag>.Navigation
    {
        get => Tag;
        set => Tag = value;
    }
}
```

We can now re-write our facade collection to use any types that implement this interface:

``` c#
public class JoinCollectionFacade<TEntity, TOtherEntity, TJoinEntity> 
    : ICollection<TEntity>
    where TJoinEntity : IJoinEntity<TEntity>, IJoinEntity<TOtherEntity>, new()
{
    private readonly TOtherEntity _ownerEntity;
    private readonly ICollection<TJoinEntity> _collection;

    public JoinCollectionFacade(
        TOtherEntity ownerEntity,
        ICollection<TJoinEntity> collection)
    {
        _ownerEntity = ownerEntity;
        _collection = collection;
    }

    public IEnumerator<TEntity> GetEnumerator()
        => _collection.Select(e => ((IJoinEntity<TEntity>)e).Navigation).GetEnumerator();

    IEnumerator IEnumerable.GetEnumerator()
        => GetEnumerator();

    public void Add(TEntity item)
    {
        var entity = new TJoinEntity();
        ((IJoinEntity<TEntity>)entity).Navigation = item;
        ((IJoinEntity<TOtherEntity>)entity).Navigation = _ownerEntity;
        _collection.Add(entity);
    }

    public void Clear()
        => _collection.Clear();

    public bool Contains(TEntity item)
        => _collection.Any(e => Equals(item, e));

    public void CopyTo(TEntity[] array, int arrayIndex)
        => this.ToList().CopyTo(array, arrayIndex);

    public bool Remove(TEntity item)
        => _collection.Remove(
            _collection.FirstOrDefault(e => Equals(item, e)));

    public int Count
        => _collection.Count;

    public bool IsReadOnly
        => _collection.IsReadOnly;

    private static bool Equals(TEntity item, TJoinEntity e)
        => Equals(((IJoinEntity<TEntity>)e).Navigation, item);
}
```

The main advantage of this new abstraction is that specific delegates to select target entities and create join entities are not needed anymore. So now in our entities we can create collections like so:

``` c#
public class Post
{
    public Post() => Tags = new JoinCollectionFacade<Tag, Post, PostTag>(this, PostTags);

    public int PostId { get; set; }
    public string Title { get; set; }

    private ICollection<PostTag> PostTags { get; } = new List<PostTag>();

    [NotMapped]
    public ICollection<Tag> Tags { get; }
}

public class Tag
{
    public Tag() => Posts = new JoinCollectionFacade<Post, Tag, PostTag>(this, PostTags);

    public int TagId { get; set; }
    public string Text { get; set; }

    private ICollection<PostTag> PostTags { get; } = new List<PostTag>();

    [NotMapped]
    public IEnumerable<Post> Posts { get; }
}
```

Everything else, including the little test application, is unchanged from the previous post.
