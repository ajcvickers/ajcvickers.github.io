---
layout: post
title: Notification entities in EF Core 1.1
date: 2016-11-16 15:06
author: ajcvickers
comments: true
categories: [Change Tracking, DetectChanges, EF, EF Core, Entity Framework, INotifyCollectionChanged, INotifyPropertyChanged, INotifyPropertyChanging, Notification Entities, Snapshot Change Tracking]
---
By default, EF Core uses snapshot change tracking. However, if entity types implement INotifyPropertyChanged and optionally INotifyPropertyChanging, then EF can use these notifications to avoid the overhead of creating snapshots and detecting changes.



<h2>Snapshot change tracking</h2>

When tracking entities EF needs to know which properties have changed in order to send appropriate updates to the database. By default this is done by creating a snapshot of the property values when the entity is either queried or attached to the context. Each entity is then scanned by a method called DetectChanges which compares the snapshot values to the current property values and marks as modified any property where the value has changed. DetectChanges is called automatically as part of SaveChanges, ensuring that the correct updates are sent to the database.

<h2>INotifyPropertyChanged</h2>

The need for DetectChanges to scan every entity can be removed if the entity itself notifies EF whenever a property has changed. This is done by having the entity type implement the INotifyPropertyChanged interface. For example:

[code lang=csharp]
public class Blog : INotifyPropertyChanged
{
    private int _id;
    private string _title;

    public int Id
    {
        get { return _id; }
        set
        {
            if (_id != value)
            {
                _id = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Id)));
            }
        }
    }

    public string Title
    {
        get { return _title; }
        set
        {
            if (_title != value)
            {
                _title = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Title)));
            }
        }
    }

    public event PropertyChangedEventHandler PropertyChanged;
}
[/code]

INotifyPropertyChanged contains one member: the PropertyChanged event. Whenever a property changes it is the responsibly of the entity type to call this event passing in the name of the property that has changed.

<h3>Using a base class</h3>

Rather than repeat the notification logic for every property it is common to use a base class. For example:

[code lang=csharp]
public class Blog : NotificationEntity
{
    private int _id;
    private string _title;

    public int Id
    {
        get { return _id; }
        set { SetWithNotify(value, ref _id); }
    }

    public string Title
    {
        get { return _title; }
        set { SetWithNotify(value, ref _title); }
    }
}

public class NotificationEntity : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;

    protected void SetWithNotify<T>(T value, ref T field, [CallerMemberName] string propertyName = "")
    {
        if (!Equals(field, value))
        {
            field = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }
    }
}
[/code]

Notice the use of the CallerMemberName attribute. This tells the compiler to automatically set the value of propertyName to the name of the property that is calling the method.

<h3>Notifications from collections</h3>

When entity types have collection navigation properties it is not sufficient, and indeed usually not useful, to know when the collection property has changed. Instead EF needs to know when entities are added or removed from the collection. The collection does this by implementing INotifyCollectionChanged.

The most common way to do this is to use ObservableCollection<T>. For example:

[code lang=csharp]
    public ICollection<Post> Posts { get; } = new ObservableCollection<Post>();
[/code]

This works fine, but it is backed by a list data structure which can be slow when using Contains on a large collection. For this reason EF Core ships with ObservableHashSet<T>. For example:

[code lang=csharp]
    public ICollection<Post> Posts { get; } = new ObservableHashSet<Post>();
[/code]

This lives in <code>Microsoft.EntityFrameworkCore.ChangeTracking</code>, so make sure you are using that namespace.

Collection navigation properties can be initialized automatically during query or fixup if they have a setter. Consider this:

[code lang=csharp]
    public ICollection<Post> Posts { get; set; }
[/code]

If EF needs to add entities to this collection and it is null, then EF will create and set a collection. Normally EF will use a HashSet<T> for this. However, if the entity type implements INotifyPropertyChanged, then EF will use an ObservableHashSet<T> instead.

<h3>EF requirements for INotifyPropertyChanged</h3>

To have EF use INotifyPropertyChanged the entity type must:

<ul>
<li>Ensure that all mapped properties send notifications. Don't have some properties send notifications and some not or EF will miss changes.</li>
<li>Ensure that any collection navigation properties implement INotifyCollectionChanged, as described above.</li>
</ul>

Sometimes entity types may need to implement INotifyPropertyChanged for reasons other than EF change tracking. In these cases it may not be necessary to follow the rules above. Indeed, the developer may not even know about the rules or be aware that EF can use INotifyPropertyChanged for change tracking. For this reason, just implementing INotifyPropertyChanged is not sufficient to have EF Core use the event for change tracking. Instead, some explicit code must be added in OnModelCreating:

[code lang=csharp]
modelBuilder.Entity<Blog>()
    .HasChangeTrackingStrategy(ChangeTrackingStrategy.ChangedNotifications);
[/code]

Or set this at the model level to use INotifyPropertyChanged for all entity types:

[code lang=csharp]
modelBuilder
    .HasChangeTrackingStrategy(ChangeTrackingStrategy.ChangedNotifications);
[/code]

If this option is set at the model level, then all entity types must implement INotifyPropertyChanged and follow the rules above.

<h2>INotifyPropertyChanging</h2>

Using INotifyPropertyChanging doesn't avoid snapshots completely because sometimes EF needs to know what the property was set to before it was changed. In EF parlance, this is called the property's <em>original value</em>. INotifyPropertyChanged doesn't provide this information, but another interface, INotifyPropertyChanging, can be used to get it. Adding this interface to the NotificationEntity base class is pretty straightforward:

[code lang=csharp]
public class Blog : NotificationEntity
{
    private int _id;
    private string _title;

    public int Id
    {
        get { return _id; }
        set { SetWithNotify(value, ref _id); }
    }

    public string Title
    {
        get { return _title; }
        set { SetWithNotify(value, ref _title); }
    }

    public ICollection<Post> Posts { get; } = new ObservableHashSet<Post>();
}

public class NotificationEntity : INotifyPropertyChanged, INotifyPropertyChanging
{
    public event PropertyChangedEventHandler PropertyChanged;
    public event PropertyChangingEventHandler PropertyChanging;

    protected void SetWithNotify<T>(T value, ref T field, [CallerMemberName] string propertyName = "")
    {
        if (!Equals(field, value))
        {
            PropertyChanging?.Invoke(this, new PropertyChangingEventArgs(propertyName));
            field = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }
    }
}
[/code]

Tell EF to start using both interfaces with:

[code lang=csharp]
modelBuilder.Entity<Blog>()
    .HasChangeTrackingStrategy(ChangeTrackingStrategy.ChangingAndChangedNotifications);
[/code]

Or:

[code lang=csharp]
modelBuilder
    .HasChangeTrackingStrategy(ChangeTrackingStrategy.ChangingAndChangedNotifications);
[/code]

<h3>Getting original values anyway</h3>

EF has no need to snapshot original values when INotifyPropertyChanged and INotifyPropertyChanging are both being used. However, sometimes application code needs original values. For example, the application may need to reset a property to its original value when the user hits a <em>Cancel</em> button. For cases where full notifications and original values are both needed, <code>ChangeTrackingStrategy.ChangingAndChangedNotificationsWithOriginalValues</code> can be used.

<h2>Summary</h2>

EF Core uses snapshot change tracking by default. This has the least burden on entity type implementation and is sufficient for most applications. EF also supports INotifyPropertyChanged, INotifyPropertyChanging, and INotifyCollectionChanged for cases where the snapshot and or DetectChanges overhead is significant. EF must be explicitly configured to use these interfaces to avoid subtle bugs when entity types implement these interfaces for other reasons.
