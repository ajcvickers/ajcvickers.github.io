---
layout: default
title: "Code First model builder versions"
date: 2012-04-22 15:41
day: 22nd
month: April
year: 2012
author: ajcvickers
permalink: 2012/04/22/code-first-model-builder-versions/
---

# Entity Framework 5.0
# Code First model builder versions

Code First's convention-based approach to building a model has some interesting implications for breaking changes between different versions of EF. This post describes those implications and shows how model builder versions can be used to deal with potential breaking changes while still retaining a forward-moving developer experience.
<h3>Starting out with a new version</h3>
Code First builds the mapping between your code and the database by examining your code and running a set of conventions. For given input code and set of conventions this will always produce the same mapping. All is well.

But what happens if a new version of EF supports new features? Let's take enums as an example since these are now supported in EF5 on .NET 4.5. Developers using EF5 with Code First will expect enums to be mapped to the database correctly…and indeed they are. All is still well.
<h3>Updating from an old version</h3>
But let's say somebody has a working EF 4.3.1 application and there is an enum somewhere in the model. In EF 4.3.1 this enum is not mapped, which presumably was fine since the app is working—maybe because it is using a pre-EF5 workaround for enums. For example:

``` c#
public enum Gender
{
    Male,
    Female,
    Unknown
}

public class Turtle
{
    public int Id { get; set; }

    private int GenderValue { get; set; }
    public Gender Gender
    {
        get { return (Gender)GenderValue; }
        set { GenderValue = (int)value; }
    }

    public static readonly Expression<Func> GenderExpression = t => t.GenderValue;
}

public class TurtleContext : DbContext
{
    public DbSet Turtles { get; set; }

    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder
            .Entity()
            .Property(Turtle.GenderExpression)
            .HasColumnName("Gender");
    }
}
```

Now when the application is rebuilt and run against EF5 Code First will try to map the enum. In other words, the mapping for the given input code has now changed. This is essentially because the conventions in EF5 have been updated to include enums.

For the example above the result is that EF now maps Gender to a column in the Turtles table. Running the app will either give a “the model has changed” exception or will fail when attempting to query/save due to the expected column being missing. All is not well.
<h3>Should this be allowed?</h3>
This is clearly a breaking change—the results after updating your app to use the new version are not the same as when using the old version. The traditional Microsoft response to this is to not allow the breaking change to happen. But what this means is that every version of Code First must continue by default to build the same model as was built by the first version. In other words, the conventions can never be updated to take advantages of new features.

If we took this approach, then in our example the developer coming to EF5 and expecting to be able to use enums is going to be quite disappointed because they appear to still not work. So avoiding making any breaking changes does not allow the developer experience to continue to improve. All would not be well.
<h3>What to do?</h3>
The solution we settled on has several parts to it:
<ul>
	<li>Make the default experience be that all new features work without each feature having to be explicitly enabled. In other words, continue to improve the default experience.</li>
	<li>Allow the version of the conventions used by Code First to be explicitly specified so that a developer can opt-in to using specific conventions regardless of which version of EF the code is built with.</li>
	<li>Commit to not releasing EntityFramework.dll as an in-place update so that an update to a new version always requires some explicit step to be taken. This is, a running app won't break just because some new version of EntityFramework.dll makes it onto the box.</li>
</ul>
<h3>What it looks like</h3>
To force your context to use an older version of the conventions just add an attribute to your context class:

``` c#
[DbModelBuilderVersion(DbModelBuilderVersion.V4_1)]
public class TurtleContext : DbContext
{
    public DbSet Turtles { get; set; }

    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder
            .Entity()
            .Property(Turtle.GenderExpression)
            .HasColumnName("Gender");
    }
}
```

All is well again.

You could also just add a [NotMapped] annotation to the Gender property or just remove the workaround and start using the enum directly:

``` c#
public class Turtle
{
    public int Id { get; set; }
    public Gender Gender { get; set; }
}

public class TurtleContext : DbContext
{
    public DbSet Turtles { get; set; }
}
```

All is even better.
<h3>Will everyone be happy?</h3>
Of course not. Some developers feel very strongly that there should be no breaking changes between library versions. On the other hand, other developers feel very strongly that being strict about breaking changes perpetuates bad experiences.

If you fall into the former camp then you may want to consider adding a model builder version attribute to all your contexts. That way if your app is rebuilt against a new version of EF it will not break because of new features detected by the conventions.

Hopefully the solution we have creates a good balance between continuous improvement of the developer experience and maintaining backward compatibility.
