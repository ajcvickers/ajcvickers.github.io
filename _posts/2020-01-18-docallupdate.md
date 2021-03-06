---
layout: default
title: "EF Core Tips: Make sure to call Update when it is needed!"
date: 2020-01-18 13:30
day: 18th
month: January
year: 2020
author: ajcvickers
permalink: 2020/01/18/docallupdate/
---

# EF Core Tips
# Make sure to call Update when it is needed!

My <a href="/2020/01/17/dontcallupdate/">last post</a> talked about not calling <code>DbContext.Update</code> or <code>DbSet.Update</code> when it isn't needed. This post presents the opposite: a place where it is not obvious but necessary to call <code>Update</code>.

---

<h2>Getting it wrong...</h2>

Here's an example of the kind of code I see which makes this mistake. Can you spot it?

``` c#
public async Task<IActionResult> OnPostAsync(int id)
{
    // Warning: Don't copy-paste this code. It is wrong.

    var user = new User { Id = id };

    _context.Attach(user);

    if (await TryUpdateModelAsync<User>(
        user,
        "user",
        s => s.Name, s => s.Email))
    {
        await _context.SaveChangesAsync();
        return RedirectToPage("./Index");
    }

    return Page();
}
```

This code is intended to update an existing entity in the database. The sequence of events is supposed to be:

<ol>
<li>Create a new, empty User instance. (This is sometimes called a "stub".)</li>
<li>Attach it so EF will start tracking changes.</li>
<li>Use <code>TryUpdateModelAsync</code> from ASP.NET Core to set values into the entity instance.</li>
<li>EF will detect that these values have been set and mark the properties as modified.</li>
<li><code>SaveChanges</code> updates the database with these changes.</li>
</ol>

Sounds good, right? But what happens if the email coming back from the client is null? In this case setting <code>Email</code> to null in <code>TryUpdateModelAsync</code> has no affect--it's already null. This means there is no change for EF to detect, which means the <code>Email</code> property is <em>not marked as modified</em> and will <em>not be updated in the database</em>.

So, with this code, the <code>Email</code> will never be updated to NULL in the database. Instead it will retain its current value.

---

<h2>Fixing it...</h2>

Just changing <code>Attach</code> to <code>Update</code> in the above code will fix the issue. <code>Update</code> works the same as <code>Attach</code> except that it sets all properties as modified instead of unchanged.

``` c#
public async Task<IActionResult> OnPostAsync(int id)
{
    var user = new User { Id = id };

    _context.Update(user); // Use Update here instead of Attach

    if (await TryUpdateModelAsync<User>(
        user,
        "user",
        s => s.Name, s => s.Email))
    {
        await _context.SaveChangesAsync();
        return RedirectToPage("./Index");
    }

    return Page();
}
```

Email will now always be updated even if it is set to null, since <code>Update</code> marks all properties as modified.

---

<h2>Slightly better still...</h2>

So why isn't this the code that I showed in the <a href="/2020/01/17/dontcallupdate/">previous post</a>? Notice in the code above that <code>Id</code> is set before calling <code>Update</code>. This is because EF needs to know the primary key value in order to track the entity correctly. This can also be true for some other property types--for example, alternate keys.

If we instead use the code from the previous post, then <code>TryUpdateModelAsync</code> can set every required property before EF starts tracking the entity. Ultimately all properties are marked as modified by the Update call, and so all values will be saved.

``` c#
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

Of course, you should still consider the trade-offs from the previous post when using this code.
