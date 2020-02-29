---
layout: post
title: EF 6.1: Turning on logging without recompiling
date: 2014-02-09 13:53
author: ajcvickers
comments: true
categories: [DbContext, DbContext API, EF6.1, Entity Framework, Extensibility, Interception, Logging]
---
I already blogged about <a href="/2013/05/08/ef6-sql-logging-part-1-simple-logging/">SQL logging in EF6</a>. <a href="/2013/05/14/ef6-sql-logging-part-3-interception-building-blocks/">Part 3</a> of that series shows how to use EF with a logging framework such as NLog. If you do this then you can easily switch logging on and off using NLog or equivalent without any changes to the app. This is the approach I would use if I wanted to log SQL from EF. But what if logging was not considered at all when the app was created? Now with EF 6.1 you can switch logging on for any app without access to the source or recompiling.
<h2>How to do it</h2>
To log SQL to the console add the following entry to the entityFramework section in your application's web.config or app.config:

[code language="xml"]
<interceptors>
  <interceptor type="System.Data.Entity.Infrastructure.Interception.DatabaseLogger, EntityFramework"/>
</interceptors>
[/code]

(If you are just working with the compiled application then look for a file like “MyApplication.exe.config” since this is what app.config is compiled into.)

To log to a filename instead use something like:

[code language="xml"]
<interceptors>
  <interceptor type="System.Data.Entity.Infrastructure.Interception.DatabaseLogger, EntityFramework">
    <parameters>
      <parameter value="C:\Stuff\LogOutput.txt"/>
    </parameters>
  </interceptor>
</interceptors>
[/code]

By default this will cause the log file to be overwritten with a new file each time the app starts. To instead append to the log file if it already exists use something like:

[code language="xml"]
<interceptors>
  <interceptor type="System.Data.Entity.Infrastructure.Interception.DatabaseLogger, EntityFramework">
    <parameters>
      <parameter value="C:\Stuff\LogOutput.txt"/>
      <parameter value="true" type="System.Boolean"/>
    </parameters>
  </interceptor>
</interceptors>
[/code]

<h2>What's going on</h2>
This is the part of the post where I describe what's going on in EF. You don't need to read any of this to use logging, but for those interested here it is.

The built-in SQL logging uses a class called DatabaseLogFormatter.  DatabaseLogFormatter is an EF interceptor that implements various interfaces which inform it of operations on commands, connections, transactions, etc. It creates log output based on these operations and sends this log output to a registered delegate—effectively sending the formatted log output to a stream. Changing what is logged can be customized as described in <a href="/2013/05/09/ef6-sql-logging-part-2-changing-the-contentformatting/">part 2</a> of the logging series.

EF 6.1 introduced IDbConfigurationInterceptor. This is an interception interface which lets code examine and/or modify EF configuration when the app starts. Using this hook we can write a simple version of DatabaseLogger:

[code language="csharp"]
public class ExampleDatabaseLogger : IDbConfigurationInterceptor
{
    public void Loaded(
        DbConfigurationLoadedEventArgs loadedEventArgs,
        DbConfigurationInterceptionContext interceptionContext)
    {
        var formatterFactory = loadedEventArgs
                .DependencyResolver
                .GetService<Func<DbContext, Action<string>, DatabaseLogFormatter>>();

        var formatter = formatterFactory(null, Console.Write);

        DbInterception.Add(formatter);
    }
}
[/code]

This is basically the same as DatabaseLogger in the EntityFramework assembly except that I have simplified it a bit by only including the code that logs to the Console. The Loaded method is called when EF configuration (as represented by the DbConfiguration class) has been loaded. Let's look at the three lines of code in Loaded.
<ul>
	<li>The first line asks EF for the registered DatabaseLogFormatter factory. We could just new up a new DatabaseLogFormatter here instead, but if the app has customized log formatting then this line will get that customized logger instead of the default.</li>
	<li>The first line didn't actually get the DatabaseLogFormatter. It instead got a factory that can be used to create DatabaseLogFormatter instances. The second line invokes that factory to get an actual formatter instance. Null is passed for the context since we are going to log for all context instances. The second argument to the factory is a delegate for the Console output stream—in this case, just a pointer to the Console.Write method. If we were logging to a file we would pass the Write method from our StreamWriter instance instead.</li>
	<li>The third line registers the DatabaseLogFormatter as an interceptor, which is how logging in EF works as described above.</li>
</ul>
Since this class doesn't define a constructor the C# compiler will generate a parameterless default constructor for it. It is this constructor that is called when the interceptor is registered in config with no parameters. We could also add constructors with parameters, such as the ones used in the real DatabaseLogger to register logging to a file. Keep in mind that the parameters must accept constant values that can be written into the config file.
