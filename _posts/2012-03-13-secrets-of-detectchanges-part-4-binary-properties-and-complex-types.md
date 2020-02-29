---
layout: default
title: "Secrets of DetectChanges Part 4: Binary properties and complex types"
date: 2012-03-13 10:04
day: 13th
month: March
year: 2012
author: ajcvickers
permalink: 2012/03/13/secrets-of-detectchanges-part-4-binary-properties-and-complex-types/
---

# Secrets of DetectChanges
# Part 4: Binary properties and complex types

### Relevance

These posts were written in 2012 for Entity Framework 4.3.
However, the information is fundamentally correct for all versions up to and including EF6.

The general concepts are also relevant for EF Core.

As always, [use your noggin](/noggin/).

---

In parts <a href="/2012/03/10/secrets-of-detectchanges-part-1-what-does-detectchanges-do/">1</a>, <a href="/2012/03/11/secrets-of-detectchanges-part-2-when-is-detectchanges-called-automatically/">2</a>, and <a href="/2012/03/12/secrets-of-detectchanges-part-3-switching-off-automatic-detectchanges/">3</a> of this series we looked at fairly normal, if occasionally advanced, uses of DetectChanges. In this post we're going to look at some corner cases around complex types and binary properties. While these are corner cases its still worth knowing about them so they don't catch you out if you ever run into them.
<h3>DetectChanges and binary properties</h3>
EF supports byte array properties for storing binary data such as images. For example, we could store the image for a banner on a blog by adding a byte array property to our Blog class:

``` c#
public class Post
{
    // Just showing added byte array property
    public byte[] BannerImage { get; set; }
}
```

If you want to change this image you must do it through setting a new byte[] instance. Don't try to change the contents of the existing byte array. DetectChanges doesn't go down into the contents of the byte array and attempt to see if it has changed. It assumes that if the same instance is still present on the entity that was there when it took the snapshot, then the property has not changed, and so no update for the property will be sent to the database.

In other words, treat your byte arrays as if they are immutable.
<h3>Binary keys</h3>
EF also allows you to use byte array properties as keys. (This is rarely a good idea, but you <em>might</em> have a reason to legitimately do it.)

When a binary property is a key (primary or foreign) then DetectChanges behaves differently—it checks the contents of the byte array, not just the instance. This is because it is unlikely that a foreign key property on one entity and a primary key property on another entity will be set to the same byte array instance <em>even when they represent matching keys</em>. EF must still be able to detect that these two byte[] objects represent the same key in order to correctly do fixup and order changes sent to the database.
<h3>Complex types</h3>
EF <a href="http://msdn.microsoft.com/en-us/library/bb738472.aspx">complex types</a> allow properties to be gathered into non-entity classes that do not have keys. An entity with a property that is of a complex type is said to have a complex property and the object set on that property is called a complex object.

EF allows complex objects to be mutated. That is, you can change the values of the properties inside the complex object and EF will detect these changes and send appropriate updates to the database.

For example, consider a Person entity with an Address complex property, which itself contains a PhoneNumbers complex property:

``` c#
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
```

Notice that Person fulfills the <a href="http://msdn.microsoft.com/en-us/library/dd468057.aspx">requirements</a> for being a <a href="/2011/12/05/should-you-use-entity-framework-change-tracking-proxies/">change-tracking proxy</a>. So you might think that DetectChanges is not needed and that this code will work:

``` c#
using (var context = new PeopleContext())
{
    context.Configuration.AutoDetectChangesEnabled = false;

    var person = context.People
        .Single(p => p.Name == "Frans");

    person.Address.Street = "1 Tall Street";
    person.Address.City = "Fairbanks";
    person.Address.State = "AK";
    person.Address.PhoneNumbers.Home = "555-555-5555";
    person.Address.PhoneNumbers.Work = "555-555-5556";

    context.SaveChanges();
}
```

Running this code will result in <strong>no</strong> changes being sent to the database! The reason is that even though Person is proxied, the Address and PhoneNumbers types are not proxied, even though they have virtual properties. EF never creates change-tracking proxies for complex types. Instead, snapshot change tracking and DetectChanges is always used for complex types.
<h3>Treat complex objects as immutable</h3>
All is not lost. You can still avoid the need for DetectChanges when using complex types and change-tracking proxies so long as you treat the objects as immutable. In practice, this means always setting a new instance of the complex type rather than changing the properties of an existing instance. For example, this will work fine:

``` c#
using (var context = new PeopleContext())
{
    context.Configuration.AutoDetectChangesEnabled = false;

    var person = context.People
        .Single(p => p.Name == "Frans");

    person.Address =
        new Address
        {
            Street = "1 Tall Street",
            City = "Fairbanks",
            State = "AK",
            PhoneNumbers =
                new PhoneNumbers
                {
                    Home = "555-555-5555",
                    Work = "555-555-5556"
                }

        };

    context.SaveChanges();
}
```

I prefer to always treat complex objects as immutable—they then become a good analogue for value types in DDD.
<h3>Rule 1 to the rescue</h3>
If you do want to mutate a complex object and do it in a way that won't require DetectChanges, then <a href="/2012/03/12/secrets-of-detectchanges-part-3-switching-off-automatic-detectchanges/">Rule 1</a> can come to the rescue again:

``` c#
using (var context = new PeopleContext())
{
    context.Configuration.AutoDetectChangesEnabled = false;

    var person = context.People
        .Single(p => p.Name == "Frans");

    var addressEntry = context.Entry(person)
        .ComplexProperty(p => p.Address);

    addressEntry
        .Property(a => a.Street)
        .CurrentValue = "1 Tall Street";

    addressEntry
        .Property(a => a.City)
        .CurrentValue = "Fairbanks";

    addressEntry
        .Property(a => a.State)
        .CurrentValue = "AK";

    addressEntry
        .ComplexProperty(a => a.PhoneNumbers)
        .Property(p => p.Home)
        .CurrentValue = "555-555-5555";

    addressEntry
        .ComplexProperty(a => a.PhoneNumbers)
        .Property(p => p.Work)
        .CurrentValue = "555-555-5556";

    context.SaveChanges();
}
```

This code uses calls into EF code to mutate the complex objects and hence, by Rule 1, doesn't require DetectChanges to be called.
<h3>And that's DetectChanges!</h3>
If you've read all four parts of this series then thank you! You are now a certified DetectChanges expert. So how about going off to <a href="http://stackoverflow.com/">Stack Overflow</a> and using your knowledge to the betterment of the EF community. :-)
