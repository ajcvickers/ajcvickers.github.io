---
layout: post
title: Deleting orphans with Entity Framework
date: 2012-06-02 18:18
author: ajcvickers
comments: true
categories: [Cascade delete, Code First, Data Annotations, DbContext, DbContext API, EF4.1, EF4.2, EF4.3, EF5, Entity Framework, Foreign Keys]
---
It is common for a single parent entity to be related to many child entities. This relationship may be <em>required </em>or <em>optional</em>. A  required relationship means that the child cannot exist without a parent, and if the parent is deleted or the relationship between the child and the parent is severed, then the child becomes orphaned. In such situations it is often useful to have the orphaned child automatically deleted.<!--more-->
<h3>An example model</h3>
Consider the following model representing students, report cards, and honors advisors.

[sourcecode language="csharp"]
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }

    public int? HonorsAdvisorId { get; set; }
    public virtual HonorsAdvisor HonorsAdvisor { get; set; }

    public virtual ICollection ReportCards { get; set; }
}

public class ReportCard
{
    public int Id { get; set; }
    public decimal Gpa { get; set; }
    public string Remarks { get; set; }

    public int StudentId { get; set; }
    public virtual Student Student { get; set; }
}

public class HonorsAdvisor
{
    public int Id { get; set; }
    public string Name { get; set; }

    public virtual ICollection Students { get; set; }
}
[/sourcecode]

Each student has many report cards and it doesn’t make sense for a report card to exist without it belonging to a student, so the relationship is required. Note that this doesn’t mean that the student must have report cards—she could have none. What it does mean is that any report card that exists must be associated with a student.

On the other hand, a student who is not enrolled in the honors program can happily exist without having an honors advisor, so this relationship is optional.
<h3>Relationships and foreign keys</h3>
When using foreign keys in your model a required relationship is usually represented by using a non-nullable foreign key. For example, the StudentId property of ReportCard.

Conversely, optional relationships are usually represented by nullable foreign keys. For example, the HonorsAdvisorId property of Student. If HonorsAdvisorId is set to null it means that the student does not have an honors advisor.

You can also use the [Required] attribute or the fluent API to force relationships to be required or optional regardless of FK nullability. This can be useful when using nullable types such as strings as keys. For example, here’s how the student/advisor relationship could be made required even though it has a nullable FK:

[sourcecode language="csharp"]
modelBuilder
    .Entity()
    .HasRequired(s =&gt; s.HonorsAdvisor)
    .WithMany(r =&gt; r.Students);
[/sourcecode]
<h3>Cascade delete for required relationships</h3>
Let’s say that a student leaves the school and our application handles that by deleting the student from the database:

[sourcecode language="csharp"]
public static void StudentLeaves(string name)
{
    using (var context = new SchoolContext())
    {
        context.Students.Remove(context.Students.Single(s =&gt; s.Name == name));
        context.SaveChanges();
    }
}
[/sourcecode]

What will happen to the student’s report cards? Dumping my test database contents before and after shows this:

Before:

[sourcecode language="text"]
Students:
  Student Pinky Pie with advisor Princess Celestia
  Student Rainbow Dash with advisor Princess Celestia
Report cards:
  Report card for Pinky Pie has GPA 4.00 and remarks 'Best student ever.'
  Report card for Pinky Pie has GPA 4.00 and remarks 'Still doing great.'
  Report card for Rainbow Dash has GPA 2.10 and remarks 'Spends too much time flying.'
  Report card for Rainbow Dash has GPA 2.20 and remarks 'Needs to sit still.'
[/sourcecode]

After:

[sourcecode language="text"]
Students:
  Student Rainbow Dash with advisor Princess Celestia
Report cards:
  Report card for Rainbow Dash has GPA 2.10 and remarks 'Spends too much time flying.'
  Report card for Rainbow Dash has GPA 2.20 and remarks 'Needs to sit still.'
[/sourcecode]

The answer is that they will be deleted automatically because Code First has setup a cascade delete between Student and ReportCard. A cascade delete means that if the parent is deleted then all the children will also be deleted. Code First did this because the relationship between students and report cards is required.

Code First not only placed the cascade delete in the model but also configured it in the database. This is important—it is expected that if an EF cascade delete exists then it must also exist in the database. If the two are not in sync then you risk getting constraint exceptions from the database. It is because there is a cascade delete in the database that the report cards were deleted without even loading them into the context.
<h3>No cascade delete for optional relationships</h3>
Let’s say that an honors advisor leaves the school:

[sourcecode language="csharp"]
public static void AdvisorLeaves(string name)
{
    using (var context = new SchoolContext())
    {
        context
            .HonorsAdvisors
            .Remove(context.HonorsAdvisors
                        .Include(a =&gt; a.Students)
                        .Single(a =&gt; a.Name == name));

        context.SaveChanges();
    }
}
[/sourcecode]

What will happen to the advisor’s students? Dumping students before and after shows this:

Before:

[sourcecode language="text"]
Students:
  Student Pinky Pie with advisor Princess Celestia
  Student Rainbow Dash with advisor Princess Celestia
[/sourcecode]

After:

[sourcecode language="text"]
Students:
  Student Pinky Pie with advisor
  Student Rainbow Dash with advisor
[/sourcecode]

The answer is that each HonorsAdvisorId FK property and HonorsAdvisor navigation property is set to null but the students are not deleted. This is because Code First did not setup a cascade delete for the optional relationship.

Note that in this case I needed to load the students into memory so that EF could set the FKs to null before saving.

If you do want to force a cascade delete on an optional relationship you can do so using the fluent API:

[sourcecode language="csharp"]
modelBuilder
    .Entity()
    .HasRequired(s =&gt; s.HonorsAdvisor)
    .WithMany(r =&gt; r.Students)
    .WillCascadeOnDelete();
[/sourcecode]
<h3>Severing relationships</h3>
Imagine a student complains about her report card and the teacher agrees to write a new one. (Maybe a parent agreed to donate a large sum to the school.) There might be some code like:

[sourcecode language="csharp"]
public static void DoctorReport(string name)
{
    using (var context = new SchoolContext())
    {
        var student = context.Students.Single(s =&gt; s.Name == name);

        student.ReportCards.Remove(
            student.ReportCards.OrderBy(r =&gt; r.Id).Last(r =&gt; r.Student.Name == name));

        student.ReportCards.Add(new ReportCard { Gpa = 3.5m, Remarks = &quot;Doing better at staying still.&quot; });

        context.SaveChanges();
    }
}
[/sourcecode]

What will happen when SaveChanges is called? The answer is that you get an exception reading:
<blockquote>System.InvalidOperationException: The operation failed: The relationship could not be changed because one or more of the foreign-key properties is non-nullable. When a change is made to a relationship, the related foreign-key property is set to a null value. If the foreign-key does not support null values, a new relationship must be defined, the foreign-key property must be assigned another non-null value, or the unrelated object must be deleted.</blockquote>
This is because EF cascade delete only kicks in when a parent is <em>deleted</em>. It doesn’t do anything when the parent still exists but the relationship has been severed. This is something that is on our backlog to fix.

You can solve this problem by directly deleting the orphaned child or by overriding SaveChanges to find and delete orphans:

[sourcecode language="csharp"]
public override int SaveChanges()
{
    ReportCards
        .Local
        .Where(r =&gt; r.Student == null)
        .ToList()
        .ForEach(r =&gt; ReportCards.Remove(r));

    return base.SaveChanges();
}
[/sourcecode]

This code does the following:
<ul>
	<li>Uses DbSet.Local to get access to non-deleted report card entities currently being tracked by the context without running any database query.</li>
	<li>Filters this list for any that do not reference a student.</li>
	<li>Makes a copy of this filtered list to avoid modifying a collection while enumerating it.</li>
	<li>Marks each orphaned report card as deleted.</li>
</ul>
<h3>Summary</h3>
By default, Code First makes an optional relationship when the FK is nullable and a required relationship when the FK is non-nullable. Required relationships are configured to cascade delete so that if the parent is deleted then all the children will also be deleted.

The optional/required nature of a relationship can be changed with the fluent API or data annotations and cascade delete can be configured with the fluent API.

A cascade delete will not delete orphans that have been severed from their parent, but this can be done by overriding SaveChanges.
<h3>EF Trivia</h3>
The exception message above is known on the team as the “conceptual null message.” This is because normally when a relationship is severed the FK for that relationship is set to null. However, if the property is non-nullable then EF instead conceptually sets it to null without actually doing so. Such “conceptual nulls” cannot be saved to the database, hence the exception.

The text itself is a message that describes in detail what the problem is…in a way that most people don’t understand. It is therefore not very helpful. We often use it as an example of a pitfall to avoid when writing exception messages. That is, the message needs to describe the problem and suggest a way to fix the problem <em>from the user’s perspective</em>. Writing something in terms of the implementation details often does not help much.

We should really write a new message that is more helpful, but better still would be to fix the delete orphans problems so that the exception is no longer needed. Now if I could only stop blogging long enough to do that…
