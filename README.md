<p align="center">
    <a href="#fluentscheduler">
        <img alt="logo" src="Assets/logo-200x200.png">
    </a>
</p>

# FluentScheduler

[![][build-img]][build]
[![][nuget-img]][nuget]

Automated job scheduler with fluent interface.

[build]:     https://ci.appveyor.com/project/TallesL/fluentscheduler
[build-img]: https://ci.appveyor.com/api/projects/status/github/fluentscheduler/fluentscheduler?svg=true
[nuget]:     https://www.nuget.org/packages/FluentScheduler
[nuget-img]: https://badge.fury.io/nu/fluentscheduler.svg

## Usage

The job configuration is handled in a [Registry] class.
A job is either an [Action] or a class that inherits [IJob]:

```cs
using FluentScheduler;

public class MyRegistry : Registry
{
    public MyRegistry()
    {
        // Schedule an IJob to run at an interval
        Schedule<MyJob>().ToRunNow().AndEvery(2).Seconds();

        // Schedule an IJob to run once, delayed by a specific time interval
        Schedule<MyJob>().ToRunOnceIn(5).Seconds();

        // Schedule a simple job to run at a specific time
        Schedule(() => Console.WriteLine("It's 9:15 PM now."))
            .ToRunEvery(1).Days().At(21, 15);

        // Schedule a more complex action to run immediately and on an monthly interval
        Schedule(() =>
        {
            Console.WriteLine("Complex job started at " + DateTime.Now);
            Thread.Sleep(10000);
            Console.WriteLine("Complex job ended at" + DateTime.Now);
        }).ToRunNow().AndEvery(1).Months().OnTheFirst(DayOfWeek.Monday).At(3, 0);
        
        // Schedule multiple jobs to be run in a single schedule
        Schedule<MyJob>().AndThen<MyOtherJob>().ToRunNow().AndEvery(5).Minutes();
    }
} 
```

To check all possible options of scheduling you can use IntelliSense on the go or check this daunting call graph (click to enlarge):

<p align="center">
    <a href="https://github.com/fluentscheduler/FluentScheduler/raw/master/Assets/call-graph.png">
        <img alt="call graph" src="Assets/call-graph.png" width="300">
    </a>
</p>

With the registry ready you then need to initialize the [JobManager].
This is usually done as soon as your application is loaded (in the [Application_Start] method of a web application for example):

```cs
protected void Application_Start()
{
    JobManager.Initialize(new MyRegistry()); 
} 
```

It's also possible to schedule jobs after initialization:

```cs
JobManager.AddJob(() => Console.WriteLine("Late job!"), (s) => s.ToRunEvery(5).Seconds());
```

[Registry]:          Library/Registry.cs
[IJob]:              Library/IJob.cs
[Action]:            https://msdn.microsoft.com/library/System.Action
[JobManager]:        Library/JobManager.cs
[Application_Start]: https://msdn.microsoft.com/library/ms178473

## Using it with ASP.NET

When using it with ASP.NET consider implementing [IRegisteredObject] in your job and registering it itself on
[HostingEnvironment]&nbsp;([here's a great explanation on it]), like:

```cs
public class SampleJob : IJob, IRegisteredObject
{
    private readonly object _lock = new object();

    private bool _shuttingDown;

    public SampleJob()
    {
        // Register this job with the hosting environment.
        // Allows for a more graceful stop of the job, in the case of IIS shutting down.
        HostingEnvironment.RegisterObject(this);
    }

    public void Execute()
    {
        lock (_lock)
        {
            if (_shuttingDown)
                return;

            // Do work, son!
        }
    }

    public void Stop(bool immediate)
    {
        // Locking here will wait for the lock in Execute to be released until this code can continue.
        lock (_lock)
        {
            _shuttingDown = true;
        }

        HostingEnvironment.UnregisterObject(this);
    }
}
```

[IRegisteredObject]:                https://msdn.microsoft.com/library/System.Web.Hosting.IRegisteredObject
[HostingEnvironment]:               https://msdn.microsoft.com/library/System.Web.Hosting.HostingEnvironment
[here's a great explanation on it]: http://haacked.com/archive/2011/10/16/the-dangers-of-implementing-recurring-background-tasks-in-asp-net.aspx

## Dependency Injection

FluentScheduler makes it easy to use your IoC tool of choice to create job instances.
Simply implement [IJobFactory].

An example using [StructureMap]:

```cs
using FluentScheduler;
using StructureMap;

public class StructureMapJobFactory : IJobFactory
{
    public IJob GetJobInstance<T>() where T : IJob
    {
        return ObjectFactory.Container.GetInstance<T>();
    }
}

public class MyRegistry : Registry
{
    public MyRegistry()
    {
        // Schedule an IJob to run at an interval
        Schedule<MyJob>().ToRunNow().AndEvery(2).Seconds();
    }
} 
```

Register the new job factory with the [JobManager]:

```cs
protected void Application_Start()
{
    JobManager.JobFactory = new StructureMapJobFactory();
    JobManager.Initialize(new MyRegistry()); 
}
```

[IJobFactory]:  Library/JobFactory.cs
[StructureMap]: http://structuremap.github.io

## Unexpected exceptions

To observe unhandled exceptions from your scheduled jobs, you will need to hook the JobException event on
[JobManager].
That event will give you access to the underlying [System.Threading.Tasks.Task] and thrown exception details.

```cs
JobManager.JobException += (info) => Log.Fatal("An error just happened with a scheduled job: " + info.Exception);
```

[System.Threading.Tasks.Task]: https://msdn.microsoft.com/library/System.Threading.Tasks.Task

## Daylight Saving Time

Unfortunately, not unlike many schedulers, there is no Daylight Saving Time support yet.

If you are worried about your jobs not running or running twice due to that, the suggestion is to avoid troublesome time
ranges or just `UseUtcTime()` in your registry.

## Major changes

**Version 4 to 5**

* `AndEvery` only starts after `ToRunOnceAt` has been fired;
* Simplified event data types and signatures.

**Version 3 to 4**

* Renamed `ITask`, `TaskManager` and `ITaskFactory` to `IJob`, `JobManager` and `IJobFactory`.

## Contributing

Feel free to [open an issue] or [submit a pull request].

When sending a patch remember to [Run All Tests (Ctrl + R, A)] and [Run Code Analysis on Solution (Alt + F11)] if
possible.
And, of course, be consistent with the existing code!

[open an issue]:                             https://github.com/fluentscheduler/FluentScheduler/issues
[submit a pull request]:                     https://github.com/fluentscheduler/FluentScheduler/pulls
[Run All Tests (Ctrl + R, A)]:               https://msdn.microsoft.com/library/ms182470
[Run Code Analysis on Solution (Alt + F11)]: https://msdn.microsoft.com/library/bb907198
