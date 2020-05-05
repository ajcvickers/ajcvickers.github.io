---
layout: default
title: "EF Core Tips: Don't call Update when you don't need to!"
date: 2020-01-17 22:14
day: 17th
month: January
year: 2020
author: ajcvickers
permalink: 2020/01/17/dontcallupdate/
---

# EF Core Tips
# Don't call Update when you don't need to!

I see all kinds of code while investigating issues and answering questions on <a href="https://github.com/dotnet/efcore">GitHub</a>. One thing that frequently crops up is calling <code>DbContext.Update</code> or <code>DbSet.Update</code> when it is not needed.

---

<h2>Change tracking</h2>

A brief bit of background. This is the typical way to update an entity in the database when using a single context instance:

```c#
using (var context = new UserContext())
{
    // Query for the entity.
    var user = context.Users.Single(e => e.Name == "Arthur");

    // Entity is now tracked. Make a change to it.
    user.Email = "arth@example.com";

    // EF will detect the change and update only the column that has changed.
    context.SaveChanges();
}
```

Notice that there is no need to explicitly tell EF that the <code>Email</code> property value has changed. EF detects this automatically and marks the property as modified. <code>SaveChanges</code> therefore knows to send an update for only the Email column.

```sql
Executed DbCommand (1ms) [Parameters=[@p1='1', @p0='arth@example.com' (Size = 4000)], CommandType='Text', CommandTimeout='30']
SET NOCOUNT ON;
UPDATE [Users] SET [Email] = @p0
WHERE [Id] = @p1;
SELECT @@ROWCOUNT;
```

---

<h2>What Update does</h2>

So what happens if we stick in a call to <code>Update</code>?

```c#
using (var context = new UserContext())
{
    // Query for the entity.
    var user = context.Users.Single(e => e.Name == "Arthur");

    // Entity is now tracked. Make a change to it.
    user.Email = "arth@example.com";

    // Mark all property values as modified.
    context.Update(user);

    // EF will update all columns because all properties are modified
    context.SaveChanges();
}
```

The <code>Update</code> method marks the entity and <em>all its properties</em> as modified. This means <code>SaveChanges</code> will send updates for both <code>Name</code> and <code>Email</code>, even though the value of <code>Name</code> has not changed:

```sql
Executed DbCommand (2ms) [Parameters=[@p2='1', @p0='arth@example.com' (Size = 4000), @p1='Arthur' (Size = 4000)], CommandType='Text', CommandTimeout='30']
SET NOCOUNT ON;
UPDATE [Users] SET [Email] = @p0, [Name] = @p1
WHERE [Id] = @p2;
SELECT @@ROWCOUNT;
```

---

<h2>So use Update here...</h2>

The entity in the examples above is tracked by the context when changes are made. This is often not the case when updating a disconnected entity such as in an ASP.NET Core application.

For example, consider this example modified from the <a href="https://docs.microsoft.com/en-us/aspnet/core/data/ef-rp/crud?view=aspnetcore-3.1">Razor Pages with EF Core in ASP.NET Core tutorial</a>:

```c#
public async Task<IActionResult> OnPostAsync(int id)
{
    var user = new User();

    if (await TryUpdateModelAsync<User>(
        user,
        "user",
        s => s.Id, s => s.Name, s => s.Email))
    {
        _context.Update(user);

        await _context.SaveChangesAsync();
        return RedirectToPage("./Index");
    }

    return Page();
}
```

In this case the call to <code>Update</code> is needed because the entity is created in the client, not queried from the database. This means it is not known which property values have changed, so marking all properties as modified ensures updates are made to any column that might have changed.

---

<h2>But don't use Update here!</h2>

But there is another common approach to updating disconnected entities: query for it before applying changes from the client. For example, this is closer to the example from the <a href="https://docs.microsoft.com/en-us/aspnet/core/data/ef-rp/crud?view=aspnetcore-3.1">Razor Pages with EF Core in ASP.NET Core tutorial</a>:

``` c#
public async Task<IActionResult> OnPostAsync(int id)
{
    // Query for the entity.
    var user = await _context.Users.FindAsync(id);

    if (user == null)
    {
        return NotFound();
    }

    // Entity is now tracked. Make changes to it.
    if (await TryUpdateModelAsync<User>(
        user,
        "user",
        s => s.Name, s => s.Email))
    {
        // EF will detect the change and update only the column that has changed.
        await _context.SaveChangesAsync();
        return RedirectToPage("./Index");
    }

    return Page();
}
```

Notice that the logic here is <em>exactly the same as the first example we looked at</em>. This means that if only the <code>Email</code> value has changed, then only the <code>Email</code> column is updated.

Even better, if no values have changed, then the database will noEmailt be updated at all.

---

<h2>The trade-off</h2>

There is a trade-off between these two approaches to disconnected entities. The first requires only one round trip to the database, but all columns will be updated. The second has two database round trips, but missing entities are handled more cleanly (it is easy to return <code>NotFound()</code>) and only values actually changed are updated. This trade-off is discussed more in the <a href="https://docs.microsoft.com/en-us/ef/core/saving/disconnected-entities">disconnected entities documentation</a>.

The point here is that if you choose the second approach, then <strong><em>don't also call <code>Update</code></em></strong>. Its not needed and can cause more columns to be updated than is necessary.
