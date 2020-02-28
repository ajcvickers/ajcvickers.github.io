---
layout: post
title: Secrets of DetectChanges Part 3: Switching off automatic DetectChanges
date: 2012-03-12 09:26
author: ajcvickers
comments: true
categories: [Change Tracking, DbContext API, DetectChanges, EF4, EF4.1, EF4.2, EF4.3, EF5, Entity Framework, Foreign Keys, POCO, Proxies, SaveChanges]
---
In parts <a href="/2012/03/10/secrets-of-detectchanges-part-1-what-does-detectchanges-do/">1</a> and <a href="/2012/03/11/secrets-of-detectchanges-part-2-when-is-detectchanges-called-automatically/">2</a> of this series we looked at what DetectChanges does and why the context calls DetectChanges automatically. In this part we’ll look at how automatic calls to DetectChanges can be switched off and what you then need to do differently in your code.
<h3>
Turning off automatic DetectChanges</h3>
If your context is not tracking a large number of entities you pretty much never need to switch off automatic DetectChanges. This holds true for a great many apps, especially when the app makes use of the best practice of having a short-lived context—for example, a context-per-request in a web app.

If your context is tracking thousands of entities you often still don’t need to turn off automatic DetectChanges as long as the app doesn’t call methods that use DetectChanges many, many times. The time you might want to switch off automatic DetectChanges is if your app is tracking many entities and repeatedly calling one of the methods that calls DetectChanges. The usual example of this is looping through a collection of entities and calling Add or Attach for each. For example:

``` c#
public void AddPosts(List posts)
{
    using (var context = new AnotherBlogContext())
    {
        posts.ForEach(p => context.Posts.Add(p));
        context.SaveChanges();
    }
}
```

In this example every call to Add results in a call to DetectChanges. This makes the operation O(n<sup>2</sup>) where n is the number of posts in the list. If n is large then this operation obviously gets very expensive. To avoid this you can turn off automatic DetectChanges just for the duration of the potentially expensive operation:

``` c#
public void AddPosts(List posts)
{
    using (var context = new AnotherBlogContext())
    {
        try
        {
            context.Configuration.AutoDetectChangesEnabled = false;
            posts.ForEach(p => context.Posts.Add(p));
        }
        finally
        {
            context.Configuration.AutoDetectChangesEnabled = true;
        }
        context.SaveChanges();
    }
}
```

Using try/finally ensures that automatic DetectChanges is always switched back on even if an exception is thrown when looping through the entities and calling Add.
<h3>Rules for when DetectChanges is needed</h3>
So how do you know that it’s okay for DetectChanges not to be called in the loop above? Taking this further, if you turn DetectChanges off for a longer period, then how do you know if this is okay or not?

The answer is that there are two rules that EF adheres to:
<ol>
	<li>No call to EF code will leave the context in a state where DetectChanges needs to be called if it didn’t need to be called before.</li>
	<li>Any time that non-EF code changes any property value of an entity or complex object then DetectChanges may need to be called.</li>
</ol>
So the calls to Add above will not result in DetectChanges needing to be called (rule 1) and since the code also doesn’t make any changes to the post entities (rule 2) the end result is that DetectChanges is not needed. In fact, this means that in this case the SaveChanges call could also safely been performed before automatic DetectChanges is switched back on.
<h3>Making effective use of Rule 1</h3>
If the code makes change changes to the properties of the entities instead of just calling Add or Attach, then, by Rule 2, DetectChanges will need to be called, at least as part of SaveChanges and possibly also before then.

However, this can be avoided by taking notice of Rule 1. Rule 1 can be very powerful when used in conjunction with the DbContext property APIs. For example, if you want to set the value of a property but need to do it without requiring a call to DetectChanges you can use the property API to do this. For example:

``` c#
public void AttachAndMovePosts(Blog efBlog, List posts)
{
    using (var context = new AnotherBlogContext())
    {
        try
        {
            context.Configuration.AutoDetectChangesEnabled = false;

            context.Blogs.Attach(efBlog);

            posts.ForEach(
                p =>
                {
                    context.Posts.Attach(p);
                    if (p.Title.StartsWith("Entity Framework:"))
                    {
                        context.Entry(p)
                            .Property(p2 => p2.Title)
                            .CurrentValue = p.Title.Replace("Entity Framework:", "EF:");

                        context.Entry(p)
                            .Reference(p2 => p2.Blog)
                            .CurrentValue = efBlog;
                    }
                });

            context.SaveChanges();
        }
        finally
        {
            context.Configuration.AutoDetectChangesEnabled = true;
        }
    }
}
```

This method attaches all posts in the given list. In addition, it replaces any title that starts with “Entity Framework:” with a title that starts with “EF:” and moves that post to a different blog.

If the code had done this by changing the properties of the Post entity directly, then a call to DetectChanges would have been required for EF to know about these changes and perform fixup, etc., and to ensure the changes are saved to the database correctly.

Instead, the code uses .Property and .Reference methods and then sets both the Title scalar property and the Blog navigation property through use of CurrentValue. Since this is a call into EF code it means that, in accordance with Rule 1, EF will ensure that everything is taken care of without needing a call to DetectChanges. This means that the code can call SaveChanges before switching on automatic DetectChanges with the confidence that everything will be saved correctly.
<h3>What about change-tracking proxies?</h3>
Another way to ensure that DetectChanges is not needed is to make use of change-tracking proxies. This is certainly a valid approach, but it also has its limitations and drawbacks, as described in my <a href="/2011/12/05/should-you-use-entity-framework-change-tracking-proxies/">previous post on the subject</a>.
<h3>Summary</h3>
Summarizing the advice in this post on DetectChanges:
<ul>
	<li>Don’t turn off automatic DetectChanges unless you really need to; it will just cause you pain.</li>
	<li>If you need to turn off DetectChanges, then do it locally using a try/finally.</li>
	<li>Use the DbContext property APIs to make changes to your entities without needing a call to DetectChanges</li>
</ul>
Next time we’ll look at some corner cases around DetectChanges with complex types and binary properties.
