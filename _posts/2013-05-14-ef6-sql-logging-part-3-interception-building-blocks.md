---
layout: default
title: "EF6 SQL Logging – Part 3: Interception building blocks"
date: 2013-05-14 09:25
day: 14th
month: May
year: 2013
author: ajcvickers
permalink: 2013/05/14/ef6-sql-logging-part-3-interception-building-blocks/
---

# Entity Framework 6.0
# SQL Logging – Part 3: Interception building blocks

In <a href="/2013/05/08/ef6-sql-logging-part-1-simple-logging/">parts 1</a> and <a href="/2013/05/09/ef6-sql-logging-part-2-changing-the-contentformatting/">2</a> of this series we looked at how to use DbContext.Database.Log to log the SQL generated by EF. But this code is actually a relatively thin façade over some low-level building blocks for interception in general and, in this case, DbCommand interception in particular.
<h2>Interception interfaces</h2>
The interception code is built around the concept of interception interfaces. These interfaces inherit from IDbInterceptor and define methods that are called when EF performs some action. The intent is to have one interface per type of object being intercepted. For example, the IDbCommandInterceptor interface defines methods that are called before EF makes a call to ExecuteNonQuery, ExecuteScalar, ExecuteReader, and related methods. Likewise, the interface defines methods that are called when each of these operations completes. The DatabaseLogFormatter class that we looked at in part 2 implements this interface to log commands.
<h3>What interfaces exist?</h3>
This feature is being added relatively late in the development cycle for EF6 and is being added primarily for two reasons:
<ul>
	<li>To provide command logging</li>
	<li>To support the implementation of some other features</li>
</ul>
For this reason there are, at the time of writing, only two public IDbInterceptor interfaces: IDbCommandInterceptor and IDbCommandTreeInterceptor. In the future we plan to add other interfaces using the same pattern.
<h2>The interception context</h2>
Looking at the methods defined on any of the interceptor interfaces it is apparent that every call is given an object of type DbInterceptionContext or some type derived from this such as DbCommandInterceptionContext<>. This object contains contextual information about the action that EF is taking. For example, if the action is being taken on behalf of a DbContext, then the DbContext is included in the DbInterceptionContext. Similarly, for commands that are being executed asynchronously, the IsAsync flag is set on DbCommandInterceptionContext.

Collecting this information together into an object keeps the interface methods relatively simple and allows new contextual information to be added in the future without it being a breaking change on the interface. In the same way that we plan to add more interception types, we also plan to add more information to the interception context in the future.
<h3>Caveat</h3>
It's worth noting that the interception context is a best effort to provide contextual information. However, in some corner cases some information that you would expect to be there may not be there. This is because EF has code paths that cannot easily be changed and do not include information that might be expected. For example, when EF makes a call into a provider, the provider has no knowledge of the DbContext being used. If that provider, outside of EF, decides to call ExecuteNonQuery, then two things might happen:
<ul>
	<li>First the provider may just make the call directly, avoiding EF interception completely. (This is a consequence of having interception at the EF level rather than lower in the stack. It would be great if interception were lower in the stack, but this is unfortunately outside of the control of the EF team.)</li>
	<li>If the provider is aware of EF interception then it can dispatch the ExecuteNonQuery call through EF interceptors. This means that any registered interceptor will be notified and can act appropriately. This is what the SQL Server and SQL Server Compact providers do. However, even when a provider does this it is likely that the DbContext being used will not be included in the interception context because the provider has no knowledge of it, and a change to allow this would break the well-defined provider APIs.</li>
</ul>
Luckily this kind of situation is rare and will likely not be an issue for most applications.
<h2>Result handling</h2>
The generic DbCommandInterceptionContext<> class contains a properties called Result, OriginalResult, Exception, and OriginalException. These properties are set to null/zero for calls to the interception methods that are called before the operation is executed—i.e. the …Executing methods. If the operation is executed and succeeds, then Result and OriginalResult are set to the result of the operation. These values can then be observed in the interception methods that are called after the operation has executed—i.e. the …Executed methods. Likewise, if the operation throws, then the Exception and OriginalException properties will be set.
<h3>Suppressing execution</h3>
If an interceptor sets the Result property before the command has executed (in one of the …Executing methods) then EF will not attempt to actually execute the command, but will instead just use the result set. In other words, the interceptor can suppress execution of the command but have EF continue as if the command had been executed.

An example of how this might be used is the command batching that has traditionally been done with a wrapping provider. The interceptor would store the command for later execution as a batch but would “pretend” to EF that the command had executed as normal. Note that it requires more than this to implement batching, but this is an example of how changing the interception result might be used.

Execution can also be suppressed by setting the Exception property in one of the …Executing methods. This causes EF to continue as if execution of the operation had failed by throwing the given exception. This may, of course, cause the application to crash, but it may also be a transient exception or some other exception that is handled by EF. For example, this could be used in test environments to test the behavior of an application when command execution fails.
<h3>Changing the result after execution</h3>
If an interceptor sets the Result property after the command has executed (in one of the …Executed methods) then EF will use the changed result instead of the result that was actually returned from the operation. Similarly, if an interceptor sets the Exception property after the command has executed, then EF will throw the set exception as if the operation had thrown the exception.

An interceptor can also set the Exception property to null to indicate that no exception should be thrown. This can be useful if execution of the operation failed but the interceptor wishes EF to continue as if the operation had succeeded. This usually also involves setting the Result so that EF has some result value to work with as it continues.
<h3>OriginalResult and OriginalException</h3>
After EF has executed an operation it will set either the Result and OriginalResult properties if execution did not fail, or the Exception and OriginalException properties if execution failed with an exception.

The OriginalResult and OriginalException properties are read-only and are only set by EF after actually executing an operation. These properties cannot be set by interceptors. This means that any interceptor can distinguish between an exception or result that has been set by some other interceptor as opposed to the real exception or result that occurred when the operation was executed.
<h2>Registering interceptors</h2>
Once a class that implements one or more of the interception interfaces has been created it can be registered with EF using the DbInterception class. For example:

``` c#
Interception.AddInterceptor(new NLogCommandInterceptor());
```

Interceptors can also be registered at the app-domain level using the DbConfiguration code-based configuration mechanism.
<h2>Example: Logging to NLog</h2>
Let's put all this together into an example that using IDbCommandInterceptor and <a href="https://nlog.codeplex.com/">NLog</a> to:
<ul>
	<li>Log a warning for any command that is executed non-asynchronously</li>
	<li>Log an error for any command that throws when executed</li>
</ul>
Here's the class that does the logging, which should be registered as shown above:

``` c#
public class NLogCommandInterceptor : IDbCommandInterceptor
{
    private static readonly Logger Logger = LogManager.GetCurrentClassLogger();

    public void NonQueryExecuting(
        DbCommand command, DbCommandInterceptionContext<int> interceptionContext)
    {
        LogIfNonAsync(command, interceptionContext);
    }

    public void NonQueryExecuted(
        DbCommand command, DbCommandInterceptionContext<int> interceptionContext)
    {
        LogIfError(command, interceptionContext);
    }

    public void ReaderExecuting(
        DbCommand command, DbCommandInterceptionContext<DbDataReader> interceptionContext)
    {
        LogIfNonAsync(command, interceptionContext);
    }

    public void ReaderExecuted(
        DbCommand command, DbCommandInterceptionContext<DbDataReader> interceptionContext)
    {
        LogIfError(command, interceptionContext);
    }

    public void ScalarExecuting(
        DbCommand command, DbCommandInterceptionContext<object> interceptionContext)
    {
        LogIfNonAsync(command, interceptionContext);
    }

    public void ScalarExecuted(
        DbCommand command, DbCommandInterceptionContext<object> interceptionContext)
    {
        LogIfError(command, interceptionContext);
    }

    private void LogIfNonAsync<TResult>(
        DbCommand command, DbCommandInterceptionContext<TResult> interceptionContext)
    {
        if (!interceptionContext.IsAsync)
        {
            Logger.Warn("Non-async command used: {0}", command.CommandText);
        }
    }

    private void LogIfError<TResult>(
        DbCommand command, DbCommandInterceptionContext<TResult> interceptionContext)
    {
        if (interceptionContext.Exception != null)
        {
            Logger.Error("Command {0} failed with exception {1}",
                command.CommandText, interceptionContext.Exception);
        }
    }
}
```

Notice how this code uses the interception context to discover when a command is being executed non-asynchronously and to discover when there was an error executing a command.
<h2>Dispatching</h2>
In addition to methods for the registration of interceptors, the DbInterception class also has a Dispatch method. This method allows code that is not part of EF to dispatch notifications to interceptors on behalf of EF. This is the mechanism mentioned above that allows providers to let interceptors know that that a command is being executed outside of the control of EF. It would be rare for an application developer to ever need to use the Dispatch API, but in sure rare cases the calls would look like this:

``` c#
DbInterception.Dispatch.Command.NonQueryAsync(myCommand, new DbCommandInterceptionContext());
```

This line of code will do the following:
<ul>
	<li>Make sure that IsAsync is set on the interception context</li>
	<li>Call NonQueryExecuting on all registered IDbCommandInterceptors</li>
	<li>Call ExecuteNonQueryAsync on the given command, unless one of the NonQueryExecuting methods set the Result property as described above</li>
	<li>Setup continuations on the async task such that NonQueryExecuted is called on all registered IDbCommandInterceptors</li>
	<li>Make sure that the result task contains the correct value, which may have been changed by one of the interceptors</li>
</ul>
<h2>Conclusion</h2>
In the three posts of this series we have looked at simple command logging, customizing the log output, and the low-level building blocks for interception.