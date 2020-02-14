---
layout: post
title: Secrets of DetectChanges Part 4: Binary properties and complex types
date: 2012-03-13 10:04
author: ajcvickers
comments: true
categories: [Change Tracking, DbContext API, DetectChanges, EF4, EF4.1, EF4.2, EF4.3, EF5, Entity Framework, Foreign Keys; Complex Types, POCO, Proxies, SaveChanges]
---
In parts <a href="http://blog.oneunicorn.com/2012/03/10/secrets-of-detectchanges-part-1-what-does-detectchanges-do/">1</a>, <a href="http://blog.oneunicorn.com/2012/03/11/secrets-of-detectchanges-part-2-when-is-detectchanges-called-automatically/">2</a>, and <a href="http://blog.oneunicorn.com/2012/03/12/secrets-of-detectchanges-part-3-switching-off-automatic-detectchanges/">3</a> of this series we looked at fairly normal, if occasionally advanced, uses of DetectChanges. In this post we’re going to look at some corner cases around complex types and binary properties. While these are corner cases its still worth knowing about them so they don’t catch you out if you ever run into them.<!--more-->
<h3>DetectChanges and binary properties</h3>
EF supports byte array properties for storing binary data such as images. For example, we could store the image for a banner on a blog by adding a byte array property to our Blog class:

[sourcecode language="csharp"]
public class Post
{
    // Just showing added byte array property
    public byte[] BannerImage { get; set; }
}
[/sourcecode]

If you want to change this image you must do it through setting a new byte[] instance. Don’t try to change the contents of the existing byte array. DetectChanges doesn’t go down into the contents of the byte array and attempt to see if it has changed. It assumes that if the same instance is still present on the entity that was there when it took the snapshot, then the property has not changed, and so no update for the property will be sent to the database.

In other words, treat your byte arrays as if they are immutable.
<h3>Binary keys</h3>
EF also allows you to use byte array properties as keys. (This is rarely a good idea, but you <em>might</em> have a reason to legitimately do it.)

When a binary property is a key (primary or foreign) then DetectChanges behaves differently—it checks the contents of the byte array, not just the instance. This is because it is unlikely that a foreign key property on one entity and a primary key property on another entity will be set to the same byte array instance <em>even when they represent matching keys</em>. EF must still be able to detect that these two byte[] objects represent the same key in order to correctly do fixup and order changes sent to the database.
<h3>Complex types</h3>
EF <a href="http://msdn.microsoft.com/en-us/library/bb738472.aspx">complex types</a> allow properties to be gathered into non-entity classes that do not have keys. An entity with a property that is of a complex type is said to have a complex property and the object set on that property is called a complex object.

EF allows complex objects to be mutated. That is, you can change the values of the properties inside the complex object and EF will detect these changes and send appropriate updates to the database.

For example, consider a Person entity with an Address complex property, which itself contains a PhoneNumbers complex property:

[sourcecode language="csharp"]
public class Person
{
    public virtual int Id { get; set; }
    public virtual string Name { get; set; }
    public virtual Address Address { get; set; }
}

public class Address
{
    public virtual string Street { get; set; }
    public virtual string City { get; set; }
    public virtual string State { get; set; }
    public virtual PhoneNumbers PhoneNumbers { get; set; }
}

public class PhoneNumbers
{
    public virtual string Home { get; set; }
    public virtual string Work { get; set; }
}
[/sourcecode]

Notice that Person fulfills the <a href="http://msdn.microsoft.com/en-us/library/dd468057.aspx">requirements</a> for being a <a href="http://blog.oneunicorn.com/2011/12/05/should-you-use-entity-framework-change-tracking-proxies/">change-tracking proxy</a>. So you might think that DetectChanges is not needed and that this code will work:

[sourcecode language="csharp"]
using (var context = new PeopleContext())
{
    context.Configuration.AutoDetectChangesEnabled = false;

    var person = context.People
        .Single(p =&amp;gt; p.Name == &quot;Frans&quot;);

    person.Address.Street = &quot;1 Tall Street&quot;;
    person.Address.City = &quot;Fairbanks&quot;;
    person.Address.State = &quot;AK&quot;;
    person.Address.PhoneNumbers.Home = &quot;555-555-5555&quot;;
    person.Address.PhoneNumbers.Work = &quot;555-555-5556&quot;;

    context.SaveChanges();
}
[/sourcecode]

Running this code will result in <strong>no</strong> changes being sent to the database! The reason is that even though Person is proxied, the Address and PhoneNumbers types are not proxied, even though they have virtual properties. EF never creates change-tracking proxies for complex types. Instead, snapshot change tracking and DetectChanges is always used for complex types.
<h3>Treat complex objects as immutable</h3>
All is not lost. You can still avoid the need for DetectChanges when using complex types and change-tracking proxies so long as you treat the objects as immutable. In practice, this means always setting a new instance of the complex type rather than changing the properties of an existing instance. For example, this will work fine:

[sourcecode language="csharp"]
using (var context = new PeopleContext())
{
    context.Configuration.AutoDetectChangesEnabled = false;

    var person = context.People
        .Single(p =&amp;gt; p.Name == &quot;Frans&quot;);

    person.Address =
        new Address
        {
            Street = &quot;1 Tall Street&quot;,
            City = &quot;Fairbanks&quot;,
            State = &quot;AK&quot;,
            PhoneNumbers =
                new PhoneNumbers
                {
                    Home = &quot;555-555-5555&quot;,
                    Work = &quot;555-555-5556&quot;
                }

        };

    context.SaveChanges();
}
[/sourcecode]

I prefer to always treat complex objects as immutable—they then become a good analogue for value types in DDD.
<h3>Rule 1 to the rescue</h3>
If you do want to mutate a complex object and do it in a way that won’t require DetectChanges, then <a href="http://blog.oneunicorn.com/2012/03/12/secrets-of-detectchanges-part-3-switching-off-automatic-detectchanges/">Rule 1</a> can come to the rescue again:

[sourcecode language="csharp"]
using (var context = new PeopleContext())
{
    context.Configuration.AutoDetectChangesEnabled = false;

    var person = context.People
        .Single(p =&amp;gt; p.Name == &quot;Frans&quot;);

    var addressEntry = context.Entry(person)
        .ComplexProperty(p =&amp;gt; p.Address);

    addressEntry
        .Property(a =&amp;gt; a.Street)
        .CurrentValue = &quot;1 Tall Street&quot;;

    addressEntry
        .Property(a =&amp;gt; a.City)
        .CurrentValue = &quot;Fairbanks&quot;;

    addressEntry
        .Property(a =&amp;gt; a.State)
        .CurrentValue = &quot;AK&quot;;

    addressEntry
        .ComplexProperty(a =&amp;gt; a.PhoneNumbers)
        .Property(p =&amp;gt; p.Home)
        .CurrentValue = &quot;555-555-5555&quot;;

    addressEntry
        .ComplexProperty(a =&amp;gt; a.PhoneNumbers)
        .Property(p =&amp;gt; p.Work)
        .CurrentValue = &quot;555-555-5556&quot;;

    context.SaveChanges();
}
[/sourcecode]

This code uses calls into EF code to mutate the complex objects and hence, by Rule 1, doesn’t require DetectChanges to be called.
<h3>And that’s DetectChanges!</h3>
If you’ve read all four parts of this series then thank you! You are now a certified DetectChanges expert. So how about going off to <a href="http://stackoverflow.com/">Stack Overflow</a> and using your knowledge to the betterment of the EF community. :-)
