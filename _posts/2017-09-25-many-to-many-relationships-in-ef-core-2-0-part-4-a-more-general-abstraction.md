---
layout: post
title: Many-to-many relationships in EF Core 2.0 - Part 4: A more general abstraction
date: 2017-09-25 09:50
author: ajcvickers
comments: true
categories: [Code First, DbContext, DbContext API, EF Core, Entity Framework, Foreign Keys, Many-to-Many]
---
In the <a href="http://wp.me/p1rP25-7p">last few posts</a> we saw how to hide use of the join entity from two entities with a many-to-many relationship. This post doesn't add any additional functionality, it just abstracts some of what we saw so it can be re-used more easily.

<!--more-->

To start with we define an interface for join entities:

[code lang=csharp]
public interface IJoinEntity&lt;TEntity&gt;
{
    TEntity Navigation { get; set; }
}
[/code]

Any join entity will implement this interface twice; once for each side:

[code lang=csharp]
public class PostTag : IJoinEntity&lt;Post&gt;, IJoinEntity&lt;Tag&gt;
{
    public int PostId { get; set; }
    public Post Post { get; set; }
    Post IJoinEntity&lt;Post&gt;.Navigation
    {
        get =&gt; Post;
        set =&gt; Post = value;
    }

    public int TagId { get; set; }
    public Tag Tag { get; set; }
    Tag IJoinEntity&lt;Tag&gt;.Navigation
    {
        get =&gt; Tag;
        set =&gt; Tag = value;
    }
}
[/code]

We can now re-write our facade colection to use any types that implement this interface:

[code lang=csharp]
public class JoinCollectionFacade&lt;TEntity, TOtherEntity, TJoinEntity&gt; 
    : ICollection&lt;TEntity&gt;
    where TJoinEntity : IJoinEntity&lt;TEntity&gt;, IJoinEntity&lt;TOtherEntity&gt;, new()
{
    private readonly TOtherEntity _ownerEntity;
    private readonly ICollection&lt;TJoinEntity&gt; _collection;

    public JoinCollectionFacade(
        TOtherEntity ownerEntity,
        ICollection&lt;TJoinEntity&gt; collection)
    {
        _ownerEntity = ownerEntity;
        _collection = collection;
    }

    public IEnumerator&lt;TEntity&gt; GetEnumerator()
        =&gt; _collection.Select(e =&gt; ((IJoinEntity&lt;TEntity&gt;)e).Navigation).GetEnumerator();

    IEnumerator IEnumerable.GetEnumerator()
        =&gt; GetEnumerator();

    public void Add(TEntity item)
    {
        var entity = new TJoinEntity();
        ((IJoinEntity&lt;TEntity&gt;)entity).Navigation = item;
        ((IJoinEntity&lt;TOtherEntity&gt;)entity).Navigation = _ownerEntity;
        _collection.Add(entity);
    }

    public void Clear()
        =&gt; _collection.Clear();

    public bool Contains(TEntity item)
        =&gt; _collection.Any(e =&gt; Equals(item, e));

    public void CopyTo(TEntity[] array, int arrayIndex)
        =&gt; this.ToList().CopyTo(array, arrayIndex);

    public bool Remove(TEntity item)
        =&gt; _collection.Remove(
            _collection.FirstOrDefault(e =&gt; Equals(item, e)));

    public int Count
        =&gt; _collection.Count;

    public bool IsReadOnly
        =&gt; _collection.IsReadOnly;

    private static bool Equals(TEntity item, TJoinEntity e)
        =&gt; Equals(((IJoinEntity&lt;TEntity&gt;)e).Navigation, item);
}
[/code]

The main advantage of this new abstraction is that specific delegates to select target entities and create join entities are not needed anymore. So now in our entities we can create collections like so:

[code lang=csharp]
public class Post
{
    public Post() =&gt; Tags = new JoinCollectionFacade&lt;Tag, Post, PostTag&gt;(this, PostTags);

    public int PostId { get; set; }
    public string Title { get; set; }

    private ICollection&lt;PostTag&gt; PostTags { get; } = new List&lt;PostTag&gt;();

    [NotMapped]
    public ICollection&lt;Tag&gt; Tags { get; }
}

public class Tag
{
    public Tag() =&gt; Posts = new JoinCollectionFacade&lt;Post, Tag, PostTag&gt;(this, PostTags);

    public int TagId { get; set; }
    public string Text { get; set; }

    private ICollection&lt;PostTag&gt; PostTags { get; } = new List&lt;PostTag&gt;();

    [NotMapped]
    public IEnumerable&lt;Post&gt; Posts { get; }
}
[/code]

Everything else, including the little test application, is unchanged from the previous post.
