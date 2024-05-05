---
layout: post
title: "A template for Dependency Injection and Configuration in .NET Core console apps"
categories: blog
---

One of the nice things about working with ASP.NET Core is that dependency injection (DI) and configuration is wired up right out of the box. Working with console apps, you don't have this luxury. However, it turns out it's very easy to set this up yourself. In this post, I'll set up a minimal template for using DI and appsettings.json configuration from a .NET Core console app. I'll be using .NET Core 3.1, which is the current latest LTS version.

The first thing we'll need are a few NuGet packages, specifically these three:

```xml
<PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="3.1.1" />
<PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="3.1.1" />
<PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="3.1.1" />
```

Other than that, no other external dependencies are needed.

We'll add an appsettings.json file with the following sample content:

```json
{
    "SampleSettings": {
        "Foo": 42,
        "Bar": "a string"
    }
}
```

This file gets added to the project root, and we'll set it to copy to the output directory:

```xml
<None Update="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
</None>
```

We'll add the following class to represent this configuration element in our application:

```csharp
namespace SomeConsoleApp.Configuration
{
    public class SampleSettings
    {
        public int Foo { get; set; }

        public string Bar { get; set; }
    }
}
```

We'll also add a simple sample service that depends on these configuration settings using an options monitor:

```csharp
namespace SomeConsoleApp.Services
{
    using System;
    using System.Threading.Tasks;

    using Microsoft.Extensions.Options;

    using SomeConsoleApp.Configuration;

    public class SampleService : ISampleService
    {
        private readonly IOptionsMonitor<SampleSettings> _settingsProvider;

        public SampleService(IOptionsMonitor<SampleSettings> settingsProvider)
        {
            _settingsProvider = settingsProvider ?? throw new ArgumentNullException(nameof(settingsProvider));
        }

        public async Task<int> ComputeAnswerToVeryDifficultQuestion()
        {
            await Task.Delay(1500); // think about it

            return _settingsProvider.CurrentValue.Foo;
        }
    }
}
```

These will act as our examples that we'll use to demonstrate how to wire up DI in our application. The actual content of what this code does is not important; it's just a sample service and a sample configuration section.

One major difference that we have with respect to "normal" console applications is that our application won't really start from Program.Main. Instead, we'll create a separate class which will function as our application's entry point:

```csharp
namespace SomeConsoleApp
{
    using System;
    using System.Threading.Tasks;

    using SomeConsoleApp.Services;

    public class MainApp
    {
        private readonly ISampleService _sampleService;

        public MainApp(ISampleService sampleService)
        {
            _sampleService = sampleService ?? throw new ArgumentNullException(nameof(sampleService));
        }

        public async Task Run(string[] args)
        {
            Console.WriteLine("Main App running...");
            Console.WriteLine($"The answer is: {await _sampleService.ComputeAnswerToVeryDifficultQuestion()}");
        }
    }
}
```

This class has a Run function that basically acts as our application's Program.Main entry point. You'll also notice that this class has a constructor which takes our sample service as a parameter. This is because starting with this class, we are able to use dependency injection the way that we're used to from ASP.NET.

So how does it work?

The last piece of the puzzle is found in our Program class:

```csharp
namespace SomeConsoleApp
{
    using System;
    using System.IO;
    using System.Threading.Tasks;

    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.DependencyInjection;

    using SomeConsoleApp.Configuration;
    using SomeConsoleApp.Services;

    class Program
    {
        static async Task Main(string[] args)
            => await BuildApplication().Run(args);

        static MainApp BuildApplication()
            => BuildServiceProvider().GetService<MainApp>();

        static IServiceProvider BuildServiceProvider()
        {
            var services = new ServiceCollection();
            var config = BuildConfigurationHierarchy();

            // register services
            services.AddScoped<ISampleService, SampleService>();

            // register configuraion elements
            services.Configure<SampleSettings>(config.GetSection("SampleSettings"));

            // register main application
            services.AddSingleton<MainApp>();

            return services.BuildServiceProvider();
        }

        static IConfigurationRoot BuildConfigurationHierarchy()
            => new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.json", false)
                .Build();
    }
}
```

Here, all of our services and configuration elements are registered, including our MainApp class. We then use the service provider to instantiate our MainApp (thus allowing the use of constructor injection), and call its Run function. When we compile and run, we unsurprisingly get the following output:

```
Main App running...
The answer is: 42
```

And that's it! It's a very simple and small template, but it's a clean solution and using it will quickly get you up and running with DI and configuration in your console app.
