---
layout: post
title: EF5: Rejecting changes to a single property
date: 2012-03-24 16:14
author: ajcvickers
comments: true
categories: [Change Tracking, Code First, DbContext, DbContext API, Entity Framework, IsModified]
---
You have probably heard about the high-profile features of <a href="http://blogs.msdn.com/b/adonet/archive/2012/03/22/ef5-beta-2-available-on-nuget.aspx">EF5</a>—things like <a href="http://msdn.microsoft.com/en-us/hh859576">enums</a>, <a href="http://msdn.microsoft.com/en-us/hh859721">spatial types</a>, <a href="http://msdn.microsoft.com/en-us/hh859577">TVF support</a>, and the <a href="http://blogs.msdn.com/b/adonet/archive/2012/02/14/sneak-preview-entity-framework-5-0-performance-improvements.aspx">perf improvements</a>. But there is one little feature you may not have heard about—the ability to reject changes to an individual property. That is, to reset the IsModified flag of a property such that the value of that property will not be sent to the database.

<!--more-->

I <a href="http://blogs.msdn.com/b/adonet/archive/2011/01/30/using-dbcontext-in-ef-feature-ctp5-part-5-working-with-property-values.aspx">previously blogged</a> on the EF Team blog about working with properties in DbContext. That post described how to check whether or not a property is marked as modified, and how to mark a property as modified. In that post I stated, “It is not currently possible to reset an individual property to be not modified after it has been marked as modified. This is something we plan to support in a future release.” Well, EF5 is that future release!
<h3>What is a modified property?</h3>
A property is marked as modified if <a href="http://blog.oneunicorn.com/2012/03/10/secrets-of-detectchanges-part-1-what-does-detectchanges-do/">DetectChanges</a> determines that its current value is different from the value it had when the entity was queried or attached. Also, if you set the state of an entity to Modified, then all the entity’s properties are marked as modified.

Being marked as modified is important because only values of properties that are marked as modified will be sent to the database when SaveChanges is called.
<h3>Checking for modified properties</h3>
Consider a User entity like so:

[sourcecode language="csharp"]
public class User
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public byte[] ProfilePicture { get; set; }
    public DateTime? MemberSince { get; set; }
}
[/sourcecode]

You can check if a property of an entity is marked as modified using code like this:

[sourcecode language="csharp"]
bool emailIsModified = context.Entry(user)
    .Property(u =&gt; u.Email)
    .IsModified;
[/sourcecode]
<h3>Marking properties as modified</h3>
You can also mark a property as modified using code like this:

[sourcecode language="csharp"]
context.Entry(user)
    .Property(u =&gt; u.Email)
    .IsModified = true;
[/sourcecode]


This forces the property value to be sent to the database even if its current value is the same as its original value. This can be useful in an n-tier application such as a web app where original values are not known because the entity has traveled to and from the client and then re-attached to a new context.

Most commonly this is done at the entity level (instead of the property level) by changing the state the entity to Modified, thereby marking all properties as modified.
<h3>Rejecting changes to modified properties</h3>
When using EF5 on .NET 4.5 you can now mark a modified property as not modified again using code like this:

[sourcecode language="csharp"]
context.Entry(user)
    .Property(u =&gt; u.Email)
    .IsModified = false;
[/sourcecode]


(You need to be running EF5 on .NET 4.5 for this to work because it makes use of changes to the core EF libraries in the .NET Framework.)

When you tell EF that a property is no longer modified the context will do two things:
<ul>
	<li>It will reset the current value to the original value stored when the entity was queried or attached. (If the original value is not known then this will have no observable effect.)</li>
	<li>It will reset the modified flag.</li>
</ul>
Because the modified flag is reset it means that when SaveChanges is called the value for the property will not be sent to the database.
<h3>An example</h3>
One place where this behavior can be useful is when you want to mark all the properties of an entity as modified but then selectively choose some that you don’t want sent to the database.

For example, imagine you want to let a user modify his or her contact information and for simplicity you don’t want to keep track of exactly which properties the user edited. But you know that there are some properties that cannot be edited. For example, maybe the MemberSince property will never change and the ProfilePicture property is changed through a different UI with a different Update method. Saving the changes to the user entity without saving MemberSince or ProfilePicture could then be done like so:

[sourcecode language="csharp"]
public void Update(User user)
{
    var entry = _context.Entry(user);

    // Attach user entity and set all properties as modified
    entry.State = EntityState.Modified;

    // Reset modified state on just two properties
    entry.Property(u =&gt; u.ProfilePicture).IsModified = false;
    entry.Property(u =&gt; u.MemberSince).IsModified = false;

    // Values for ProfilePicture and MemberSince will not be
    // included in UPDATE statement sent to database
    _context.SaveChanges();
}
[/sourcecode]

In this case ProfilePicture might be quite large so avoiding sending data (that hasn’t changed) to the database can be quite advantageous.

So there you have it. A small feature of EF5 (on .NET 4.5), but hopefully a useful one.
