---
layout: post
title: "Hello World"
categories: blog
---

The evolution of software development, as illustrated by Hello World.

## Hello World (beginner)

```csharp
namespace HelloWorldApp
{
    using System;

    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
        }
    }
}
```

## Hello World (student)

```csharp
namespace HelloWorldApp
{
    using System;

    class Program
    {
        const string HelloFormat = "Hello {0}!";

        static void Main(string[] args)
        {
            PrintHello("World");
        }

        static void PrintHello(string name)
        {
            if (string.IsNullOrWhiteSpace(name))
            {
                throw new ArgumentException(nameof(name));
            }

            Console.WriteLine(string.Format(HelloFormat, name));
        }
    }
}
```

## Hello World (junior developer)

```csharp
namespace HelloWorldApp
{
    using System;

    class Program
    {
        static void Main(string[] args)
        {
            try
            {
                PrintHello("World");
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
            }
        }

        static void PrintHello(string name)
        {
            if (string.IsNullOrWhiteSpace(name))
            {
                throw new ArgumentException(nameof(name));
            }

            var helloProvider = new HelloProvider();

            Console.WriteLine(helloProvider.ProvideHello(name));
        }
    }
}

namespace HelloWorldApp
{
    using System;

    public class HelloProvider
    {
        const string HelloFormat = "Hello {0}!";

        public string ProvideHello(string name)
        {
            if (string.IsNullOrWhiteSpace(name))
            {
                throw new ArgumentException(nameof(name));
            }

            return string.Format(HelloFormat, name);
        }
    }
}
```

## Hello World (medior developer)

```csharp
namespace HelloWorldApp
{
    using System;

    using Microsoft.Extensions.DependencyInjection;

    class Program
    {
        static void Main(string[] args)
        {
            var serviceProvider = BuildServiceProvider();
            var mainApp = serviceProvider.GetService<MainApp>();

            mainApp.Run(args);
        }

        static IServiceProvider BuildServiceProvider()
        {
            var services = new ServiceCollection();

            // register services
            services.AddScoped<IHelloProvider, HelloProvider>();

            // register main application
            services.AddSingleton<MainApp>();

            return services.BuildServiceProvider();
        }
    }
}

namespace HelloWorldApp
{
    using System;

    public class MainApp
    {
        private const string World = "World";
        private readonly IHelloProvider _helloProvider;

        public MainApp(IHelloProvider helloProvider)
        {
            _helloProvider = helloProvider ?? throw new ArgumentNullException(nameof(helloProvider));
        }

        public void Run(string[] args)
        {
            try
            {
                Console.WriteLine(_helloProvider.ProvideHello(World));
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
            }
        }
    }
}

namespace HelloWorldApp
{
    public interface IHelloProvider
    {
        /// <summary>
        /// Provides a "hello" greeting for the given subject.
        /// </summary>
        /// <param name="name">The subject to greet.</param>
        /// <returns>A "hello" greeting.</returns>
        string ProvideHello(string name);
    }
}

namespace HelloWorldApp
{
    using System;

    public class HelloProvider : IHelloProvider
    {
        private const string HelloFormat = "Hello {0}!";

        public string ProvideHello(string name)
        {
            if (string.IsNullOrWhiteSpace(name))
            {
                throw new ArgumentException(nameof(name));
            }

            return string.Format(HelloFormat, name);
        }
    }
}
```

## Hello World (senior developer)

```csharp
namespace HelloWorldApp
{
    using System;
    using System.IO;
    using System.Threading.Tasks;

    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.DependencyInjection;

    using Serilog;

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

            // register logging
            services.AddLogging(x =>
            {
                var seriLogger = new LoggerConfiguration()
                    .ReadFrom.Configuration(config)
                    .CreateLogger();

                x.AddSerilog(seriLogger, true);
            });

            // register services
            services.AddScoped<IHelloConfigurationProvider, HelloConfigurationProvider>();
            services.AddScoped<IHelloProvider, HelloProvider>();

            // register configuraion elements
            services.Configure<HelloWorldConfig>(config.GetSection("HelloWorld"));

            // register main application
            services.AddSingleton<MainApp>();

            return services.BuildServiceProvider();
        }

        static IConfigurationRoot BuildConfigurationHierarchy()
            => new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.json", false)
                .AddJsonFile("appsettings.development.json", true)
                .Build();
    }
}

namespace HelloWorldApp
{
    using System;
    using System.Threading.Tasks;

    using Microsoft.Extensions.Logging;

    public class MainApp
    {
        private readonly ILogger<MainApp> _logger;
        private readonly IHelloConfigurationProvider _configurationProvider;
        private readonly IHelloProvider _helloProvider;

        public MainApp(
            ILogger<MainApp> logger,
            IHelloConfigurationProvider configurationProvider,
            IHelloProvider helloProvider)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _configurationProvider = configurationProvider ?? throw new ArgumentNullException(nameof(configurationProvider));
            _helloProvider = helloProvider ?? throw new ArgumentNullException(nameof(helloProvider));
        }

        public async Task Run(string[] args)
        {
            _logger.LogDebug("application startup");

            var config = _configurationProvider.ProvideCurrentCultureConfig();
            if (config == null)
            {
                _logger.LogError("configuration was not found");
                throw new InvalidOperationException("configuration was not found");
            }

            try
            {
                var greeting = _helloProvider.ProvideHello(config.WorldName);
                _logger.LogInformation("priting greeting: {0}", greeting);
                Console.WriteLine(greeting);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "fatal application error!");
                Console.WriteLine(ex);
            }

            _logger.LogDebug("application shutdown");
        }
    }
}

namespace HelloWorldApp
{
    public interface IHelloProvider
    {
        /// <summary>
        /// Provides a "hello" greeting for the given subject.
        /// </summary>
        /// <param name="name">The subject to greet.</param>
        /// <returns>A "hello" greeting.</returns>
        string ProvideHello(string name);
    }
}

namespace HelloWorldApp
{
    using System;

    using Microsoft.Extensions.Logging;

    public class HelloProvider : IHelloProvider
    {
        private readonly ILogger<HelloProvider> _logger;
        private readonly IHelloConfigurationProvider _configurationProvider;

        public HelloProvider(
            ILogger<HelloProvider> logger,
            IHelloConfigurationProvider configurationProvider)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _configurationProvider = configurationProvider ?? throw new ArgumentNullException(nameof(configurationProvider));
        }

        public string ProvideHello(string name)
        {
            if (string.IsNullOrWhiteSpace(name))
            {
                _logger.LogError("name was null or whitespace");
                throw new ArgumentException(nameof(name));
            }

            var config = _configurationProvider.ProvideCurrentCultureConfig();
            if (config == null)
            {
                _logger.LogError("configuration was not found");
                throw new InvalidOperationException("configuration was not found");
            }

            _logger.LogInformation("proving greeting for name: {0}", name);

            return string.Format(config.GreetingFormat, name);
        }
    }
}

namespace HelloWorldApp
{
    public interface IHelloConfigurationProvider
    {
        /// <summary>
        /// Provides "hello world" configuration for the current culture.
        /// Uses <code>Thread.CurrentThread.CurrentUICulture</code> for culture info.
        /// If no config exists for the thread culture, the default config is returned.
        /// </summary>
        /// <returns>A "hello world" configuration element.</returns>
        HelloWorldCultureConfig ProvideCurrentCultureConfig();
    }
}

namespace HelloWorldApp
{
    using System;
    using System.Threading;

    using Microsoft.Extensions.Logging;
    using Microsoft.Extensions.Options;

    public class HelloConfigurationProvider : IHelloConfigurationProvider
    {
        private readonly ILogger<HelloConfigurationProvider> _logger;
        private readonly IOptionsMonitor<HelloWorldConfig> _configMonitor;

        public HelloConfigurationProvider(
            ILogger<HelloConfigurationProvider> logger,
            IOptionsMonitor<HelloWorldConfig> configMonitor)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _configMonitor = configMonitor ?? throw new ArgumentNullException(nameof(configMonitor));
        }

        public HelloWorldCultureConfig ProvideCurrentCultureConfig()
        {
            var languageCode = Thread.CurrentThread.CurrentUICulture.TwoLetterISOLanguageName;
            _logger.LogInformation("detected language code: {0}", languageCode);

            if (_configMonitor.CurrentValue.CultureSettings.TryGetValue(
                    languageCode,
                    out var cultureSpecificConfig))
            {
                _logger.LogInformation("using culture-specific config");
                return cultureSpecificConfig;
            }
            else
            {
                _logger.LogInformation("using default config");
                return _configMonitor.CurrentValue.DefaultSettings;
            }
        }
    }
}

namespace HelloWorldApp
{
    using System.Collections.Generic;

    public class HelloWorldConfig
    {
        public HelloWorldCultureConfig DefaultSettings { get; set; }

        public Dictionary<string, HelloWorldCultureConfig> CultureSettings { get; set; }
    }
}

namespace HelloWorldApp
{
    public class HelloWorldCultureConfig
    {
        public string GreetingFormat { get; set; }

        public string WorldName { get; set; }
    }
}

{
    "Logging": {
        "LogLevel": {
            "Default": "Warning"
        }
    },
    "Serilog": {
        "Using": [ "Serilog.Sinks.RollingFile" ],
        "MinimumLevel": "Warning",
        "WriteTo": [
            {
                "Name": "RollingFile",
                "Args": {
                    "pathFormat": "HelloWorld-{Date}.txt",
                    "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level}] ({SourceContext}) {Message}{NewLine}{Exception}"
                }
            }
        ]
    },
    "HelloWorld": {
        "DefaultSettings": {
            "GreetingFormat": "Hello {0}!",
            "WorldName": "World"
        },
        "CultureSettings": {
            "en": {
                "GreetingFormat": "Hello {0}!",
                "WorldName": "World"
            },
            "nl": {
                "GreetingFormat": "Hallo {0}!",
                "WorldName": "Wereld"
            }
        }
    }
}

{
    "Logging": {
        "LogLevel": {
            "Default": "Debug"
        }
    },
    "Serilog": {
        "MinimumLevel": "Debug"
    }
}
```

## Hello World (principal developer)

```csharp
namespace HelloWorldApp
{
    using System;

    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
        }
    }
}
```
