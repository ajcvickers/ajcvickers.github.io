---
layout: post
title: Code First Data Annotations on non-public properties
date: 2012-03-26 22:18
author: ajcvickers
comments: true
categories: [Code First, Data Annotations, Entity Framework]
---
The Entity Framework supports mapping to public, protected, internal, and private properties. However, there are a few restrictions when mapping to non-public properties. and there are also some things you need to know when attempting to configure such mappings using Code First. In particular, mapping to non-public properties cannot be done using data annotations alone.
<h3>
Mapping to non-public members using Code First</h3>
We'll get to data annotations in a minute, but first let's look at how to actually get the non-public members mapped. Consider an entity class like this:

``` c#
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }

    [MaxLength(8)]
    private string Alias { get; set; }
}
```

Code First only maps public properties by default. This means that it will map Id and Name, but it will not map Alias. In other words, the value for Alias will not be written to or read from the database.

To map the Alias property you need to use the Code First fluent API. What you want to do is do something like this in the OnModelCreating method of your context:

``` c#
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    modelBuilder
        .Entity()
        .Property(p => p.Alias);
}
```

The problem is that this won't compile because the Alias property is not visible from outside the Person class. There are several ways to overcome this problem, probably the easiest of which is to embed an EntityTypeConfiguration for the Person entity inside the Person class itself. For example:

``` c#
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }

    [MaxLength(8)]
    private string Alias { get; set; }

    public class PersonConfiguration : EntityTypeConfiguration<Person>
    {
        public PersonConfiguration()
        {
            Property(p => p.Alias);
        }
    }
}
```

Then in OnModelCreating you pass this EntityTypeConfiguration to Code First:

``` c#
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    modelBuilder
        .Configurations.Add(new Person.PersonConfiguration());
}
```
<h3>The less intrusive way to do it</h3>
Embedding an EntityTypeConfiguration in your entity class isn't very clean and also means that your entity is no longer POCO, both conceptually and practically since it requires a reference to EntityFramework.dll to compile.

Jiri Cincura <a href="http://blog.cincura.net/232731-mapping-private-protected-properties-in-entity-framework-4-x-code-first/">blogged a very nice solution</a> to the problem that retains most of the POCO-ness of your entity classes. It is based on understanding that the Property fluent method expects to be passed an expression tree that describes the property. So we just need to find a way to create that expression tree. We can do that with a simple static Expression property embedded inside the entity class:

``` c#
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }

    [MaxLength(8)]
    private string Alias { get; set; }

    public static readonly Expression<Func<Person, string>> AliasExpression = p => p.Alias;
}
```

The value of this property can now be used in OnModelCreating:

``` c#
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    modelBuilder
        .Entity()
        .Property(Person.AliasExpression);
}
```
<h3>Is this really POCO?</h3>
The entity shown above is not persistence ignorant in the strictest sense since it contains code that is used to expose information about itself for use in configuring persistence. You could eliminate this code entirely if you're willing to use Reflection and expression builders  (or equivalent utility libraries) to create the needed expression manually.

To me, the added complexity of doing this is not worth it over adding a static read-only property to the entity that does not reference any persistence types and does not change the behavior or interface of the entity in any way. Also, once you use Reflection it means using strings and the loss of Intelisense and refactoring support that this entails.
<h3>So what about data annotations</h3>
Before EF 4.3 any data annotation on a non-public property would be ignored by Code First. This was fixed in 4.3 so that data annotations are respected on any mapped property. The key phrase here is “any <em>mapped </em>property”. In other words, the property must still be mapped for the data annotation to be used. And putting a data annotation on a property <em>does not cause it to be mapped.</em>

So in the example above just putting MaxLength on the property did not cause Code First to map it. However, once the property was mapped using the fluent API then Code First read the data annotation and set the maximum length for the property to 8.

I understand that this is kind of confusing, especially when you read that Code First now supports data annotations on non-public properties without any clear explanation of what that means. We're not sure yet how to make this less confusing, but we're thinking about it.
<h3>Restrictions on mapping to non-public members</h3>
I said at the top of the post that there were a few restrictions on mapping to non-public properties. Those restrictions fall into three categories:
<ul>
	<li>EF currently blocks the use of non-public properties when your app is running under partial trust (without ReflectionPermission) such as in a shared hosting environment. This is something we are hoping to address in a future release.</li>
	<li>A private or internal property cannot be overridden by a type in another assembly, which means that the services of proxies such as lazy-loading and change-tracking will not work with private or internal properties.</li>
	<li>There are some bugs around mapping private properties that are defined in unmapped base classes of the entity.</li>
</ul>
Hopefully this post clears up some of the confusion around data annotations on non-public properties.
