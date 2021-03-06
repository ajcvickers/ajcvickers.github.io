---
layout: default
title: "EF6 SQL Logging – Part 2: Changing the content/formatting"
date: 2013-05-09 09:34
day: 9th
month: May
year: 2013
author: ajcvickers
permalink: 2013/05/09/ef6-sql-logging-part-2-changing-the-contentformatting/
---

# Entity Framework 6.0
# SQL Logging – Part 2: Changing the content/formatting

<h2>DatabaseLogFormatter</h2>
Under the covers the Database.Log property makes use of a DatabaseLogFormatter object. This object effectively binds a IDbCommandInterceptor implementation (see part 3 of this series) to a delegate that accepts strings and a DbContext. This means that interception methods on DatabaseLogFormatter are called before and after the execution of commands by EF. These DatabaseLogFormatter methods gather and format log output and send it to the delegate.

Changing what is logged and how it is formatted can be achieved by creating a new class that derives from DatabaseLogFormatter and then overriding methods as appropriate. The most common methods to override are:
<ul>
	<li>LogCommand – Override this to change how commands are logged before they are executed. By default LogCommand calls LogParameter for each parameter; you may choose to do the same in your override or handle parameters differently instead.</li>
	<li>LogResult – Override this to change how the outcome from executing a command is logged.</li>
	<li>LogParameter – Override this to change the formatting and content of parameter logging.</li>
</ul>
For example, suppose we wanted to log just a single line before each command is sent to the database. This can be done with two overrides:
<ul>
	<li>Override LogCommand to format and write the single line of SQL</li>
	<li>Override LogResult to do nothing. (Executed could be overridden instead and this would probably be slightly more efficient, but functionally equivalent.)</li>
</ul>
The code would look something like this:

``` c#
public class OneLineFormatter : DatabaseLogFormatter
{
    public OneLineFormatter(DbContext context, Action<string> writeAction)
        : base(context, writeAction)
    {
    }

    public override void LogCommand<TResult>(
        DbCommand command, DbCommandInterceptionContext<TResult> interceptionContext)
    {
        Write(string.Format(
            "Context '{0}' is executing command '{1}'{2}",
            Context.GetType().Name,
            command.CommandText.Replace(Environment.NewLine, ""),
            Environment.NewLine));
    }

    public override void LogResult<TResult>(
        DbCommand command, object result, DbCommandInterceptionContext<TResult> interceptionContext)
    {
    }
}
```

To log output simply call the Write method which will send output to the configured write delegate.

(Note that this code does simplistic removal of line breaks just as an example. It will likely not work well for viewing complex SQL.)
<h2>Setting the DatabaseLogFormatter</h2>
Once a new DatabaseLogFormatter class has been created it needs to be registered with EF. This is done using <a href="http://msdn.microsoft.com/en-us/data/jj680699">code-based configuration</a>. In a nutshell this means creating a new class that derives from DbConfiguration in the same assembly as your DbContext class and then calling SetDatabaseLogFormatter in the constructor of this new class. For example:

``` c#
public class MyDbConfiguration : DbConfiguration
{
    public MyDbConfiguration()
    {
        SetDatabaseLogFormatter(
            (context, writeAction) => new OneLineFormatter(context, writeAction));
    }
}
```
<h2>Using the new DatabaseLogFormatter</h2>
This new DatabaseLogFormatter will now be used anytime Database.Log is set. So, running the code from <a href="/2013/05/08/ef6-sql-logging-part-1-simple-logging/">part 1</a> will now result in the following output:

```
Context 'BlogContext' is executing command 'SELECT TOP (1) [Extent1].[Id] AS [Id], [Extent1].[Title] AS [Title]FROM [dbo].[Blogs] AS [Extent1]WHERE (N'One Unicorn' = [Extent1].[Title]) AND ([Extent1].[Title] IS NOT NULL)'
Context 'BlogContext' is executing command 'SELECT [Extent1].[Id] AS [Id], [Extent1].[Title] AS [Title], [Extent1].[BlogId] AS [BlogId]FROM [dbo].[Posts] AS [Extent1]WHERE [Extent1].[BlogId] = @EntityKeyValue1'
Context 'BlogContext' is executing command 'update [dbo].[Posts]set [Title] = @0where ([Id] = @1)'
Context 'BlogContext' is executing command 'insert [dbo].[Posts]([Title], [BlogId])values (@0, @1)select [Id]from [dbo].[Posts]where @@rowcount > 0 and [Id] = scope_identity()'
```

<h2>More control?</h2>
In the next post we'll take a look at implementing IDbCommandInterceptor directly for even more control of command interception and show an example that integrates directly with NLog without using the Database.Log property.
