---
layout: docs.hbs
title: Use case and Deployment Scenarios
---
# Deployment Scenarios

### Console Application

```csharp
PM> install-package Akka
PM> install-package Akka.Remote
```

```csharp
using Akka;
using Akka.Actor;
using Akka.Configuration;

namespace Foo.Bar
{
    class Program
    {
        static void Main(string[] args)
        {
            //configure remoting for localhost:8081
            var fluentConfig = FluentConfig.Begin()
                .StartRemotingOn("localhost", 8081)
                .Build();

            using (var system = ActorSystem.Create("my-actor-server", fluentConfig))
            {
                //start two services
                var service1= system.ActorOf<Service1>("service1");
                var service2 = system.ActorOf<Service2>("service2");
                Console.ReadKey();
            }
        }
    }
}
```

### Windows Service

For windows service deployment its recommended to use [TopShelf](http://topshelf.readthedocs.org/en/latest/index.html) to build your Windows Services. It radically simplifies hosting Windows Services.

The quickest way to get started with TopShelf is by creating a Console Application. Which would look like this:

#### Program.cs
```csharp
using Akka.Actor;
using Topshelf;
```
```csharp
class Program {
    static void Main(string[] args) {

        HostFactory.Run(x => {
            x.Service<MyActorService>(s =>
            {
                s.ConstructUsing(n => new MyActorService());
                s.WhenStarted(service => service.Start());
                s.WhenStopped(service => service.Stop());
                //continue and restart directives are also available
            });

            x.RunAsLocalSystem();
            x.UseAssemblyInfoForServiceInfo();
        });            
    }
}

/// <summary>
/// This class acts as an interface between your application and TopShelf
/// </summary>
public class MyActorService {
    private ActorSystem mySystem;

    public void Start() {
        
        //this is where you setup your actor system and other things
        mySystem = ActorSystem.Create("MySystem");
    }

    public void Stop() {
        
        //this is where you stop your actor system
        mySystem.Shutdown();
    }
}
```
The above example is the simplest way imaginable. However there are also other styles of integration with TopShelf that give you more control.

Installing with Topshelf is as easy as calling `myConsoleApp.exe install` on the command line. 

For all the options and settings check out their [docs](http://topshelf.readthedocs.org/en/latest/index.html).

### ASP.NET

Hosting inside an ASP.NET application is easy. The Global.asax would be the designated place to start.

```csharp
public class MvcApplication : System.Web.HttpApplication
{
    protected static ActorSystem ActorSystem;

    protected void Application_Start()
    {
        //your mvc config. Does not really matter if you initialise
        //your actor system before or after

        ActorSystem = ActorSystem.Create("app");
        //here you would register your toplevel actors
        
    }
}
```

As you can see the main point here is keeping a static reference to your `ActorSystem` . This ensures it won't be accidentally garbage collected and gets disposed and created with the start and stop events of your web application. 

>**Warning**<br>Although hosting inside an ASP.NET Application is easy. A **word of caution**: When you are hosting inside of `IIS` the applicationpool your app lives in could be stopped and started at the whim of `IIS`. This in turn means your `ActorSystem` could be stopped at any given time.

Typically you use a very lightweight `ActorSystem` inside ASP.NET applications, and offload heavy-duty work to a seperate Windows Service via Akka.Remote / Akka.Cluster
