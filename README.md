# Dotnet Deep Dive 

Contents:
- [Fundamentals](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/?view=aspnetcore-8.0)
  - [App start up](#app-startup)
  - [Dependency Injection (Services)](#dependency-injection-services)
  - [Middleware](#middleware)
    - [Custom middleware example](#example-of-writing-a-custom-middleware)
  - [Host](#host)
  - [Configuration](#configuration)
  - [Environments](#environments)
  - [Logging and monitoring](#logging-and-monitoring)
      - [Health checks](#health-checks)
  - [HttpContext](#httpcontext)
  - [Routing](#routing)
  - [Handle errors](#handle-errors)
  - [Make HTTP requests](#make-http-requests)
- [APIs](#apis) 
  - [Controller-based APIs](#controller-based-apis)
  - [Minimal APIs](#minimal-apis)
  - [OpenAPI](#openapi)
- [Best practices](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices?view=aspnetcore-8.0)
  - [Servers](#servers)
    - [Kestrel](#kestrel)
    - IIS (only windows, for dotnet 8 or higher use kestrel)
- [Security and Identity](#security-and-identity)
  - [Authentication](#authentication)
    - [Identity](#identity)
  - [Authorization](#authorisation)
  - [Secrets Management](#secrets-management)
- Performance
  - [Caching](#caching)
  - [Rate limiting middleware](#rate-limiting-middleware)

---

## Fundamentals 

### App Startup 
- `Program.cs`
  - application startup code is in Program.cs file 
    - services required by the app are configured here
    - app's request handling pipeline is defined as a series of middleware components 
  - There are different ways to do this, probably depends on the type of Host used
  - Generally there will be a services registration area, i.e. `builder.services.AddSingleton<IOrganisationProvider, OrganisationProvider>();` and a middleware registration area that follows afterward, i.e. `app.UseAuthorization();`


### Dependency Injection (services)
  - dependency injection or DI makes configured services available throughout the app
  - it is a technique intended to achieve [inversion of control](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion)
    - direction of dependency in the direction of abstraction, not implementation details
    - allows for easier testing and means that different implementations of these interfaces can easily be plugged in without modifying the code (open-closed principle)
  - a depencency is an object that another object depends on
  - services are added to the DI container, i.e.:
````c#
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorPages();
builder.Services.AddControllersWithViews();
builder.services.AddSingleton<IOrganisationProvider, OrganisationProvider>();

var app = builder.Build();
````
  - this uses the [builder pattern](https://refactoring.guru/design-patterns/builder)
  - services are typically resolved from DI using constructor injection. The DI framework provides an instance of this service at runtime can can be used as follows:
````c#
public class SomeClass 
{
  private readonly IOrganisationProvider _orgProvider
  public IList<Organisation> Organisation {get ; set;}

  public SomeClass(IOrganisationPrivider organisationProvider)
  {
    _orgProvider = organisationProvider
  }

  public async Task SetOrganisation()
  {
    Organisation = await _orgProvider.GetOrganisation("someId")
  }
}
````
- [Service lifetimes](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection#service-lifetimes)
  - Services can be registered with one of the following lifetimes:
    - Transient
    - Scoped
    - Singleton
  - Transient:
    - created each time they're requested from the service container
  - Scoped:
    - A scoped lifetime indicates that services are created once per client request (connection)
  - Singleton
    - singleton lifetime services are created either:
      - the first time they're requested
      - by the developer, when providing an implementation instance directly to the container (an approach rarely needed)
    - every subsequent request of the service implementation from the DI container uses the same instance.
- Keyed Services
  - in dotnet 8 there is support for registrations of services based on a key, so you can register multiple services with a different key and use it for the lookup. i.e.:
````c#
services.AddKeyedSingleton<IMessageWriter, MemoryMessageWriter>("memory");
services.AddKeyedSingleton<IMessageWriter, QueueMessageWriter>("queue");

// example of the constructor of the class that uses IMessageWriter: 
public class ExampleService
{
    public ExampleService(
        [FromKeyedServices("queue")] IMessageWriter writer)
    {
        // Omitted for brevity...
    }
}
````


### Middleware 
  - The request handling pipeline is composed as a series of middleware components. Each component performs operations on an HttpContext and either invokes the next middleware in the pipeline or terminates the request. 
    - Each middleware is responsible for invoking the next component or short-circuiting the pipeline (i.e. terminating the request) - if it short-circuits, its called a terminal middleware because it prevents further middleware from processing the request.
  - Request delegates are used to build the request pipeline - they handle each HTTP request
    - Configured using `Run`, `Map` and `Use` extension methods
      - specified either as an in-line anonymous method (in-line middleware) or defined in a reusable class

![alt text](middleware-execution.png)
  - each middleware delegate can perform operations before and after the next delegate - the order matters.
    - i.e. exception-handling delegates should be called early in the pipeline so they can catch exceptions that occur in later stages of the pipeline.
````c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// example of an "in-line anonymous middleware" 
app.Use(async (context, next) =>
{
    // Do work that can write to the Response => i.e. before the next delegate is called
    await next.Invoke(); // this calls the next delegate in the pipeline
    // Do logging or other work that doesn't write to the Response => i.e. after the above delegate is completed. 
});

app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from 2nd delegate.");
});

app.Run();
````

- `Run` delegates don't receive a `next` parameter. The first `Run` delegate is always terminal and terminates the pipeline. `Run` is a convention. Some middleware 

  - Middle ware uses the [chain of responsibility pattern](https://refactoring.guru/design-patterns/chain-of-responsibility)
  - by convention, a middleware component is added to the pipeline by invoking a `Use{Feature}` extension method. I.e.:
````c#
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllersWithViews();

var app = builder.Build();

// Middleware below
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseAuthorization();
// middleware above

app.MapDefaultControllerRoute();

app.Run();
````

- Middleware order
  - the below diagram shows a request processing pipeline for a dotnet core MVC / razor page app. 
  - existing middlewares are ordered and custom middlewares are added

  ![alt text](mvc-razor-middlewares.png)

  - this is when explicitly calling app.UseRouting - if you don't call this, the routing middleware runs at the beginning of the pipeline by default.

  - some of the existing pre-built middleware must be called in certain order, see: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-8.0
    - i.e. `UseCors` must be before `UseAuthentication` which must be before `UseAuthorisation` etc.


- Branch the middleware pipeline
  - `Map` extensions are used as a convention for branching the pipeline. Map branches the request pipeline based on the given request path. If the request path starts with the given path, the branch is executed. 
````c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Map("/map1", HandleMapTest1);

app.Map("/map2", HandleMapTest2);

app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from non-Map delegate.");
});

app.Run();

static void HandleMapTest1(IApplicationBuilder app)
{
    app.Run(async context =>
    {
        await context.Response.WriteAsync("Map Test 1");
    });
}

static void HandleMapTest2(IApplicationBuilder app)
{
    app.Run(async context =>
    {
        await context.Response.WriteAsync("Map Test 2");
    });
}
````
- in the above code, going to 'localhost:1234' responds with "Hello from non-Map delegate"
- going to 'localhost:1234/map1' responds with "Map test 1" etc


- Built-in middleware: 
  - info here: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-8.0#built-in-middleware

#### Example of writing a custom middleware
- https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write?view=aspnetcore-8.0

---

### Host
  - On startup, a dotnet app builds a host. The host encapsulates all of the app's resources, such as:
    - A HTTP server implementation
    - Middleware components
    - Logging
    - DI services
    - Configuration
  - There are 3 different hosts capable of running a dotnet app:
    - ASP.NET Core WebApplication, also known as the Minimal Host 
    - .NET Generic Host combined with ASP.NET core's ConfigureWebHostDefaults
    - ASP.NET Core WebHost
  - `WebApplication` and `WebApplicationBuilder`are recommended and used in all dotnet core templates. 
  - `WebHost` is available only for backward compatibility. 
  - `WebApplication` / `WebApplicationBuilder` example: 
````c#
var builder = WebApplication.CreateBuilder(args); // => this is an instance of 'WebApplicationBuilder'.

// Use the builder to configure services
builder.Services.AddControllers();
builder.Services.AddScoped<ITodoRepository, TodoRepository>(); // => DI injection example.

var app = builder.Build(); // => this is an instance of 'WebApplication'.

// Use the app to configure middleware and endpoints
app.UseRouting();
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers();
});

// Run the application
app.Run();
````
  - The `WebApplicationBuilder` configures a host with a set of default options, such as:
    - Use Kestrel as the web server and enable IIS integration
    - Load configuration from appsettings.json, environment variables, command line arguments, and other config sources
    - Send logging output to the console and debug providers
  - The Generic Host allows other types of apps for non-web scenarios like cruss-cutting framework extensions


#### WepApplication Deep Dive

````c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
````
- can also use this - `WebApplication.Create` initializes a new instance of WebApplication with preconfigured defaults:

````c#
var app = WebApplication.Create(args);

app.MapGet("/", () => "Hello World!");

app.Run();
````

- Working with Ports
  - when a web app is created with visual studio or `dotnet new`, a properties/launchSettings.json file is created that specifies the ports the app responds to
  - you can also set the port in the startup.cs, including multiple ports:

````c#
var app = WebApplication.Create(args);

app.Urls.Add("http://localhost:3000");
app.Urls.Add("http://localhost:4000");

app.MapGet("/", () => "Hello World");

app.Run();
````

  - You can also set the port from the command line: `dotnet run --urls="https://localhost:7777"`
  - You can also read the port from the environment:
    - the preferred way being setting the ASPNETCORE_URLS environment variable, i.e. `ASPNETCORE_URLS=http://localhost:3000`
    - or for multiple ports, `ASPNETCORE_URLS=http://localhost:3000;https://localhost:5000`

````c#
var app = WebApplication.Create(args);

var port = Environment.GetEnvironmentVariable("PORT") ?? "3000";

app.MapGet("/", () => "Hello World");

app.Run($"http://localhost:{port}");
````

- Read the environment example

````c#
var app = WebApplication.Create(args);

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/oops");
}

app.MapGet("/", () => "Hello World");
app.MapGet("/oops", () => "Oops! An error happened.");

app.Run();
````

- Read the configuration system: (appSettings.json)

````c#
var app = WebApplication.Create(args);

var message = app.Configuration["HelloKey"] ?? "Config failed!";

app.MapGet("/", () => message);

app.Run();
````

- Access the DI container to add custom services 
- Add middleware 

````c#
var builder = WebApplication.CreateBuilder(args);

// Add a custom scoped service.
builder.Services.AddScoped<ITodoRepository, TodoRepository>(); // NOTE: done before builder.Build() is called, then middleware added to app
var app = builder.Build();
app.UseFileServer(); // add middleware

app.Run()
````

---

### Configuration
- https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-8.0
  - dotnet provides a configuration framework that gets settings as name-value pairs from an ordered set of configuration providers. Built-in config providers are available as .json or .xml files, environment variables, and command-line arguments.
  - by default they're read from `appsettings.json`, environment variables and command line arguments
  - when the apps config is loaded, values from environment variables override values from `appsettings.json`

- Host configuration
  - dotnet templates create a `webApplicationBuilder` which contains the host
    - only configuration that is necessary for the host should be done in "host configuration", the rest should be done in "application configuration"
````c#
var builder = WebApplication.CreateBuilder(args);
````
  - the initialized `WebApplicationBuilder` (above is a builder) provides default configuration for the app in the following order, from highest to lowest priority:
    - command-line arguments
    - non-prefixed env variables
    - user secrets (when the app runs in the development environment) 
    - appsettings.{Environment}.json
    - appsettings.json using
    - a fallback
  - i.e. appsettings.json gets overriden by appsettings.Prod.json which gets overridden by command-line arguments

- application config
  - overrides host config 

````c#
{
  {
    "Position": {
      "Title": "Editor",
      "Name": "Joe Smith"
    },
    "MyKey": "My appsettings.json Value",
    "Logging": {
      "LogLevel": {
        "Default": "Information",
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information"
      }
    },
    "AllowedHosts": "*"
  }
}
````
  - naming of environment vaiables should reflect the structure of an appsettings.json file, like so: `Logging_LogLevel_Default`  will override whats in the appsettings.json, i.e. replacing "Information" when set
  - accessing them looks somehting like this:

````c#
public class TestModel : PageModel
{
    // requires using Microsoft.Extensions.Configuration;
    private readonly IConfiguration Configuration;

    public TestModel(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public ContentResult OnGet()
    {
        var myKeyValue = Configuration["StmpServer"];
        var title = Configuration["Position:Title"];
        var name = Configuration["Position:Name"];
        var defaultLogLevel = Configuration["Logging:LogLevel:Default"];


        return Content($"MyKey value: {myKeyValue} \n" +
                       $"Title: {title} \n" +
                       $"Name: {name} \n" +
                       $"Default Log Level: {defaultLogLevel}");
    }
````

- Configuration keys:
  - case insensitive
- Configuration values:
  - are strings
  - null values can't be stored

- ConfigurationBinder API: 
  - GetValue extracts a single value and converts it to the specified type
  - if `NumberKey` isn't available, the default value of 99 is used

````c#
public class TestNumModel : PageModel
{
    private readonly IConfiguration Configuration;

    public TestNumModel(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public ContentResult OnGet()
    {
        var number = Configuration.GetValue<int>("NumberKey", 99);
        return Content($"{number}");
    }
}
````

- access configuration with DI

````c#
public class Service
{
    private readonly IConfiguration _config;

    public Service(IConfiguration config) =>
        _config = config;

    public void DoSomething()
    {
        var configSettingValue = _config["ConfigSetting"];

        // ...
    }
}
````

- access configuration in `Program.cs`

````c#
// appsettings.json: 
{
  ...
  "KeyOne": "Key One Value",
  "KeyTwo": 1999,
  "KeyThree": true
}

// program.cs:
var builder = WebApplication.CreateBuilder(args);

var key1 = builder.Configuration.GetValue<string>("KeyOne");

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

var key2 = app.Configuration.GetValue<int>("KeyTwo");
var key3 = app.Configuration.GetValue<bool>("KeyThree");

app.Logger.LogInformation("KeyOne: {KeyOne}", key1);
app.Logger.LogInformation("KeyTwo: {KeyTwo}", key2);
app.Logger.LogInformation("KeyThree: {KeyThree}", key3);

app.Run();
````

---

### Environments
- dotnet configures app behavior based on the runtime environment using an environment variable:
  1. DOTNET_ENVIRONMENT (env variable)
  2. ASPNETCORE_ENVIRONMENT when `WebApplication.CreateBuilder` method is called - default app templates call this.
    - the DOTNET_ENVIRONMENT value overrides ASPNETCORE_ENVIRONMENT when `WebApplicationBuilder` is used. 
    - for other hosts i,e, `ConfigureWebHostDefaults` and `WebHost.CreateDefaultBuilder`, ASPNETCORE_ENVIRONMENT has higher precdence 
- by default the `IHostEnvironment.EnvironmentName` can be set to any value but by default its "Development", "Staging", "Production"
- 


- Execution environments such as Development, Staging and Production are available in dotnet core. Specify the environment an app is running in by setting the `ASPNETCORE_ENVIRONMENT` environment variable. It gets read at startup and stores the value in `IWebHostEnvironment` implementation.
  - `IWebHostEnvironment` is available anywhere in the app via dependency injection. 

````c#
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorPages();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
    app.UseHsts();
}
````

- use the environment flag to set the environment: `dotnet run --environment Production`


**Development**
- launchSettings.json has information about the "Development" environment

````json
{
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:59481",
      "sslPort": 44308
    }
  },
  "profiles": {
    "EnvironmentsSample": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "https://localhost:7152;http://localhost:5105",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
````

- the above example has two profiles: 
  - EnvironmentsSample is used by default as its the first listed, i.e. when you go `dotnet run`, since commandName is "Project", it uses kestrel web server.
  - IIS Express ises IISExpress.
- you can select the profiles in visual studio clicking the drop down or via terminal `dotnet run --launch-profile "EnvironmentsSample"`

**PRODUCTION**
- prod should be configured to maximise security, performance, and application robustness. common settings that differ from development include:
  - Caching
  - Client-side resources are bundled, minified and potentially served from a CDN
  - Diagnostic error pages disabled
  - friendly error pages enabled
  - production logging and monitoring enabled



### Logging and Monitoring
- dotnet supports a logging API that works with a variety of built-in and third-party logging providers. Available providers include:
  - Console
  - Debug
  - Event Tracing on Windows
  - Windows Event Log 
  - TraceSource
- `var builder = WebApplication.CreateBuilder(args);` adds a default set of logging providers
  - the above defaults can be overwritten by `builder.Logging.ClearProviders()` and `builder.Logging.AddConsole()`
- Creating Logs
  - to create logs, resolve an `ILogger<TCategoryName>` service from DI and call logging methods such as `LogInformation`, i.e.:

````c#
public class IndexModel : PageModel
{
    private readonly RazorPagesMovieContext _context;
    private readonly ILogger<IndexModel> _logger;

    public IndexModel(RazorPagesMovieContext context, ILogger<IndexModel> logger)
    {
        _context = context;
        _logger = logger;
    }

    public IList<Movie> Movie { get;set; }

    public async Task OnGetAsync()
    {
        _logger.LogInformation("IndexModel OnGetAsync.");
        Movie = await _context.Movie.ToListAsync();
    }
}
````

- Configuring logging is commonly provided by the `appsettings.{ENVIRONMENT}.json` files, i.e.

````json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Warning",
    },
    "Debug": { // Debug provider.
      "LogLevel": {
        "Default": "Information", // Overrides preceding LogLevel:Default setting.
        "Microsoft.Hosting": "Trace" // Debug:Microsoft.Hosting category.
      }
  }
}
````

- "Default" and "Microsoft.AspNetCore" are specified, where "Microsoft.AspNetCore" applies to any categories that start with "Microsoft.AspNetCore" 
- the LogLevel indicates the severity of the log and ranges from 0 to 6: Trace, Debug, Information, Warning, Error, Critical, and None (from more logs to less)
- If no LogLevel is specified, logging defaults to the Information level. 
- Logs below the minimum log level are not passed to the provider, logged or displayed. 
- This has all the log providers enabled by default:

````json
{
  "Logging": {
    "LogLevel": { // No provider, LogLevel applies to all the enabled providers.
      "Default": "Error",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Warning"
    },
    "Debug": { // Debug provider.
      "LogLevel": {
        "Default": "Information" // Overrides preceding LogLevel:Default setting.
      }
    },
    "Console": {
      "IncludeScopes": true,
      "LogLevel": {
        "Microsoft.AspNetCore.Mvc.Razor.Internal": "Warning",
        "Microsoft.AspNetCore.Mvc.Razor.Razor": "Debug",
        "Microsoft.AspNetCore.Mvc.Razor": "Error",
        "Default": "Information"
      }
    },
    "EventSource": {
      "LogLevel": {
        "Microsoft": "Information"
      }
    },
    "EventLog": {
      "LogLevel": {
        "Microsoft": "Information"
      }
    },
    "AzureAppServicesFile": {
      "IncludeScopes": true,
      "LogLevel": {
        "Default": "Warning"
      }
    },
    "AzureAppServicesBlob": {
      "IncludeScopes": true,
      "LogLevel": {
        "Microsoft": "Information"
      }
    },
    "ApplicationInsights": {
      "LogLevel": {
        "Default": "Information"
      }
    }
  }
}
````

- An example of adding a log to the /Test endpoint:
````c#
var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddConsole();

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.MapGet("/Test", async (ILogger<Program> logger, HttpResponse response) =>
{
    logger.LogInformation("Testing logging in Program.cs");
    await response.WriteAsync("Testing");
});

app.Run();
````

- LogLevel can also be set by command line, env variables and other config 
  - via command line: `set Logging__LogLevel__Microsoft=Information`

- Logs are displayed:
  - in visual studo, the debug output window when debugging 
  - in the console window when the app is run with `dotnet run`

- Log Category
  - when an `ILogger` object is created, a category is specified. The category is included with each log message created by that instance of ILogger.
  - the category string is arbitrary, but the convention is to use the fully qualified class name. i.e. in a controller, the name might be `ILogger<TodoApi.Controllers.TodoController> logger`

- Log Level
  - the Log method's first param, LogLevel, indicates the severity of the log. But most evelopers call Log{LOG LEVEL} extension methods, ie

````c#
[HttpGet]
public IActionResult Test1(int id)
{
    var routeInfo = ControllerContext.ToCtxString(id);

    _logger.Log(LogLevel.Information, MyLogEvents.TestItem, routeInfo);
    _logger.LogInformation(MyLogEvents.TestItem, routeInfo); // the extension method, log level is "Information".

    return ControllerContext.MyDisplayRouteInfo();
}

[HttpGet("{id}")]
public async Task<ActionResult<TodoItemDTO>> GetTodoItem(long id)
{
    _logger.LogInformation(MyLogEvents.GetItem, "Getting item {Id}", id);

    var todoItem = await _context.TodoItems.FindAsync(id);

    if (todoItem == null)
    {
        _logger.LogWarning(MyLogEvents.GetItemNotFound, "Get({Id}) NOT FOUND", id);
        return NotFound();
    }

    return ItemToDTO(todoItem);
}
````

- in the above, `MyLogEvents.TestItem` is the event Id. 
  - each log event can specify an id, which associates a set of events. i.e. dsplaying a list of items might all use the id of 1001. 
- In terms of log levels,
  - In Production, logging at trace, debug or information levels produces a high-volume of detailed log messages. 
    - to control costs and not exceed data storage limits this isn't ideal 
    - logging at Warning to Critical levels should provide few log messages
  - In development:
    - set to warning
    - add trace, debug or info when troubleshooting.
    - to limit output, set trace, debug or info only for the categories under investigation. 

- each log API uses a message template, which can contain placeholders for which arguments are pvodied. i.e. above the LogWarning, ` "Get({Id}) NOT FOUND"`
- the order of parameters, not the placeholder names determines which params are used in values in log messages. i.e. `_logger.LogInformation("Parameters: {Pears}, {Bananas}, {Apples}", apples, pears, bananas);` would actually log in this order: "Paramters: apples, bears, bananas"

- Log Exceptions
  - logger methods have overloads that take an exception parameter:

````c#
catch (Exception ex)
{
    _logger.LogWarning(MyLogEvents.GetItemNotFound, ex, "TestExp({Id})", id);
    return NotFound();
}
````

- Log scopes
  - a scope can group a set or logical operations, which can be used to attach the same data to each log thats created as part of a set, i.e. every log can include that trasnaction ID:

````c#
[HttpGet("{id}")]
public async Task<ActionResult<TodoItemDTO>> GetTodoItem(long id)
{
    TodoItem todoItem;
    var transactionId = Guid.NewGuid().ToString();
    using (_logger.BeginScope(new List<KeyValuePair<string, object>>
        {
            new KeyValuePair<string, object>("TransactionId", transactionId),
        }))
    {
        _logger.LogInformation(MyLogEvents.GetItem, "Getting item {Id}", id);

        todoItem = await _context.TodoItems.FindAsync(id);

        if (todoItem == null)
        {
            _logger.LogWarning(MyLogEvents.GetItemNotFound, 
                "Get({Id}) NOT FOUND", id);
            return NotFound();
        }
    }

    return ItemToDTO(todoItem);
}
````

#### Health checks

- healthchecks are offered by dotnet and exposed as HTTP endpoints as a way of reporting the health of app infrastructure
- healthprobes often used by container orchestrators and load balancers to check the apps status (maybe restarting a container or routing traffic away from the failing instance)
- health checks typically used with an external monitoring service (before using, decide on which monitoring system to use)

Basic Health Probe:
- for many apps this is sufficient - reports the apps availability to process requests 
- basic config calls health check middleware to respond to a url with a health response
  - by default, no specific health checks are registered to test any particular dependency or subsystem
  - the app is considered healthy if it can respond at the health endpoint URL.
- register `AddHealthChecks` in Program.cs then create a health check endpoint by calling `MapHealthChecks`

````c#
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapHealthChecks("/healthz"); // creates an endpoint at '/healthz'

app.Run();
````

Docker HEALTHCHECK:
- docker offers a built-in HEALTHCHECK directive that can be used to check the status of an app that uses the basic health check configuration: 
  - `HEALTHCHECK CMD curl --fail http://localhost:5000/healthz || exit`

Create Health Checks: (non-default/basic)
- implemented with the IHealthCheck interface.
- CheckHealthAsync returns a HealthCheckResult that includes the health as Healthy, Degraded, or Unhealthy
- the health checks logic is placed in the CheckHealthAsync method

````c#
public class SampleHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        var isHealthy = true;

        // ...

        if (isHealthy)
        {
            return Task.FromResult(
                HealthCheckResult.Healthy("A healthy result."));
        }

        return Task.FromResult(
            new HealthCheckResult(
                context.Registration.FailureStatus, "An unhealthy result."));
    }
}
````

Register Health check services:
- call AddCheck in Program.cs 

````c#
builder.Services.AddHealthChecks()
    .AddCheck<SampleHealthCheck>("Sample"); // implements the above custom one

builder.Services.AddHealthChecks()
    .AddCheck("Sample", () => HealthCheckResult.Healthy("A healthy result.")); // in-line custom health check
````

Require Host:
- requireHost specifies one or more permitted hosts for the health check endpoint. If a collection isn't supplied, any host is accepted
- `app.mapHealthChecks("/healthz").RequireHost("www.contoso.com:5001");`


Health check options:
- provides an oppourtunity to customise health check behaviour:
  - filter health checks
    - by default the health check middleware runs all registered health checks
    - to run a subset, provide a function that returns a boolean to the Predicate option 
  - customise http status code
    - use `ResultStatusCodes` to customise the mapping of health status to HTTP status codes. 
  - suppress cache headers
    - not cached by default
    - why would you touch this, we dont want it cached
  - customise output

````c#
app.MapHealthChecks("/healthz", new HealthCheckOptions
{
    Predicate = healthCheck => healthCheck.Tags.Contains("sample") // filter health checks, only those tagged with sample run
});

// customise http status code:
app.MapHealthChecks("/healthz", new HealthCheckOptions
{ // note these are the default used by the middleware
    ResultStatusCodes =
    {
        [HealthStatus.Healthy] = StatusCodes.Status200OK,
        [HealthStatus.Degraded] = StatusCodes.Status200OK,
        [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
    }
});
````

Database Probe
- a health check can specify a db query to run as a boolean test to indicate if the database is responding normally.
- there is a nuget package to use to execute against a SQL db that does a SELECT 1 query (not supported by microsoft)
- there is also a way to do this by using entity framework core 

````c#
// SQL query one
var conStr = builder.Configuration.GetConnectionString("DefaultConnection");
if (string.IsNullOrEmpty(conStr))
{
    throw new InvalidOperationException(
                       "Could not find a connection string named 'DefaultConnection'.");
}
builder.Services.AddHealthChecks()
    .AddSqlServer(conStr);

// entity framework core one
builder.Services.AddDbContext<SampleDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddHealthChecks()
    .AddDbContextCheck<SampleDbContext>();
````

Readiness and Liveness probes:
- in some scenarios, a pair of health checks is used
  - readiness indicates if the app is running normally but not ready to receive requests
  - liveness indicates an app has crashed and must be restarted 
- i.e. an app must download a large config file before its ready to process requests 
- how to do it here: https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-8.0#separate-readiness-and-liveness-probes

Kubernetes Example:
- useful in an env such as kubernetes

````yaml
spec:
  template:
  spec:
    liveness:
    path: /ping
    port: 80
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 1
    readinessProbe:
      # an http probe
      httpGet:
        path: /healthz
        port: 80
      # length of time to wait for a pod to initialize
      # after pod startup, before applying health checking
      initialDelaySeconds: 30
      timeoutSeconds: 1
    ports:
      - containerPort: 80
````

---

### HttpContext
- HttpContext encapsulates all info about an individual HTTP request and response.
- A HttpContext instance is initialized when a HTTP request is received. The HttpContext instance is accessible by middleware and app frameworks such as api controllers, razor pages, signalR etc.

- HttpRequest 
  - HttpContext.Request provides acces - HttpRequest has information about the incoming HTTP request and is initiated when a request is received by the server.
  - middleware can change request values in the middleware pipeline.
  - commonly used are things like `HttpRequest.Method` (e.g. GET), `HttpRequest.Headers`, `HttpRequest.Query` (values passed as query string), `HttpRequest.Body` (a stream for reading the request body)
- HttpRequest.Headers provides access to the request headers sent with the request. 
  - You can either provide the header name to the indexer on the header collection, i.e. `request.Headers["x-custom-header"]` or like `request.Headers.UserAgent` for commonly used ones. 
- Read request body
  - the request body is data associated with the request, such as the content of a JSON payload.
  - read using `context.Request.Body.CopyToAsync(writeStream)`

````c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapPost("/uploadstream", async (IConfiguration config, HttpContext context) =>
{
    var filePath = Path.Combine(config["StoredFilesPath"], Path.GetRandomFileName());

    await using var writeStream = File.Create(filePath);
    await context.Request.Body.CopyToAsync(writeStream);
});

app.Run();
````

- HttpResponse
  - accessed by `HttpContext.Response`
  - used to set information on the HTTP response sent back to the client
  - commonly used: `HttpResponse.StatusCode`, `HttpResponse.ContentType`, `HttpResponse.Headers`, `HttpResponse.Body`.
- HttpResponse.Headers provides access to the response headers sent with the HTTP response
  - set custom headers: `response.Headers["x-custom-header"] = "Custom value";`
  - set common headers: `response.Headers.CacheControl = "no-cache"`
- Write response Body
  - the response body is data associated with the response, such as a JSON payload


Request Aborted
- a `HttpContext.RequestAborted` cancellation token can be used to notify that the HTTP request has been aborted by the client or server. 
  - i.e. aborting a db query or http request to get data to return in the response 
  - the requestAborted cancellation token doesn't need to be used for request body read operations because reads always throw immediately when the request is aborted. 
    - also usually unnecessary when writing response bodies ebcause the writes immediately no-op when the request is aborted. 

````c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var httpClient = new HttpClient();
app.MapPost("/books/{bookId}", async (int bookId, HttpContext context) =>
{
    var stream = await httpClient.GetStreamAsync(
        $"http://contoso/books/{bookId}.json", context.RequestAborted);

    // Proxy the response as JSON
    return Results.Stream(stream, "application/json");
});

app.Run();
````

Abort()
- `httpContext.Abort()` can be used to abort a HTTP request from the server. 
  - this immediately triggers the `HttpContext.RequestAborted` cancellation token and sends a notification to the client that the server has aborted the request. 

````c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Use(async (context, next) =>
{
    if (RequestAppearsMalicious(context.Request))
    {
        // Malicious requests don't even deserve an error response (e.g. 400).
        context.Abort();
        return;
    }

    await next.Invoke();
});

app.Run();
````

HttpContext isn't thread safe
- i.e. when using async methods, ensure that HttpContext isn't accessed after the method has yielded control (i.e. after an 'await')

---

### Routing 
- A Route is a URL pattern mapped to a handler. The handler is typically a controller or a middleware. Dotnet routing gives you control over the URLs used by your app. 
  - depenses incoming requests to the app's executable endpoints 
  - endpoints are the app's units of executable request-handling code 
- Routing uses a pair of middleware, registered by `UseRouting` and `UseEndpoints`
  - `UseRouting` adds route matching to the middleware pipeline: it looks at the set of endpoints defined in the app and selects the best match based on the request
  - `UseEndpoints` adds endpoint execution to the middleware pipeline. It runs the delegate associated with the selected endpoint.
  - apps typically don't need to call these, `WebApplicationBuilder` adds them - however we can change the order they're run by calling the methods expilcitly. 
- Endpoints
  - Endpoints that can be matched and executed by the app are configured in `UseEndpoints`. 
    - we typically use `MapControllers` or `MapHealthchecks` (note: MapHealthChecks is similar to USeHealthChecks, generally Map is preferred)

````c#
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers();
});
````

- the order matters. see below, `UseAuthentication` and `UseAuthorisation` add the authentication and authorization middleware. These are placed between `UseRouting` and `UseEndpoints` so that they can:
  - see which endpoint was selected by `UseRouting`
  - Apply an auth policy before `UseEndpoints` dispatches to the endpoint

- URL Matching
  - the process by which routing matches an incoming request to an endpoint
  - based on the data in the URL path and headers
  - can be extended to consider any data in the request 

- Route Constraints: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-8.0#route-constraints
  - ensures endpoints are only matched when they match. i.e. instead of `"users/{id}"` you can have `"users/{id:int}"` - it only matches when its an int.
  - more complex ones available, i.e. `{filename:length(8,16)}` means string must be between 8 and 16 characters long 
  - regular expressions can be used too... this is potentially expensive and time consuming.

````c#
[Route("users/{id:int:min(1)}")]
public User GetUserById(int id) { }
````

  - can also make custom route constraints

- Optional route parameter order
  - optional params must come after all required route params and literals

````c#
[Route("api/[controller]")]
public class MyController : ControllerBase
{
    // GET /api/my/red/2/joe
    // GET /api/my/red/2
    // GET /api/my
    [HttpGet("{color}/{id:int?}/{name?}")]
    public IActionResult GetByIdAndOptionalName(string color, int id = 1, string? name = null)
    {
        return Ok($"{color} {id} {name ?? ""}");
    }
}
````

- Host matching in routes with RequireHost
  - applies a constraint to the route which requires the specific host, i.e: wwww.domain.com or *.domain.com would match subdomain.domain.com or wwww.subdomain.domain.com
  - for ports something like *.domain.com:5000
  - multiple params can be used

````c#
[Host("contoso.com", "adventure-works.com")]
public class HostsController : Controller
{
    public IActionResult Index() =>
        View();

    [Host("example.com")]
    public IActionResult Example() =>
        View();
}
````

---

### Handle Errors

- Problem details
  - problem details are commonly used to report errors for HTTP APIs
  - implements the `IProblemDetailsService` which supports creating problem details
  - the `AddProblemDetails()` extension method on services registers the default `IProblemDetailsService` implementation

````c#
builder.Services.AddProblemDetails();

var app = builder.Build();        

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler();
    app.UseHsts();
}
````

- Customise Problem Details
  - creation of problemDetails can be customised using one of the following options:
    1. ProblemDetailsOptions.CustomizeProblemDetails
    2. using a custom IProblemDetailsWriter
    3. call the IProblemDetailsService in a middleware
  - CustomizeProblemDetails operation:
    1. the customisations here are applied to all auto-generated problem details:

````c#
builder.Services.AddProblemDetails(options =>
  options.CustomizeProblemDetails = ctx =>
          ctx.ProblemDetails.Extensions.Add("nodeId", Environment.MachineName));
````

  - with the above, a 400 bad request will produce this problem details reponse body:

````json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "Bad Request",
  "status": 400,
  "nodeId": "my-machine-name"
}
````

  2. Custom IProblemDetailsWriter:
    - created for more advanced customisations 
    - When using a custom IProblemDetailsWriter, the custom IProblemDetailsWriter must be registered before calling 

````c#
public class SampleProblemDetailsWriter : IProblemDetailsWriter
{
    // Indicates that only responses with StatusCode == 400
    // are handled by this writer. All others are
    // handled by different registered writers if available.
    public bool CanWrite(ProblemDetailsContext context)
        => context.HttpContext.Response.StatusCode == 400;

    public ValueTask WriteAsync(ProblemDetailsContext context)
    {
        // Additional customizations.

        // Write to the response.
        var response = context.HttpContext.Response;
        return new ValueTask(response.WriteAsJsonAsync(context.ProblemDetails));
    }
}

// in startup:
builder.Services.AddTransient<IProblemDetailsWriter, SampleProblemDetailsWriter>();
// Middleware to handle writing problem details to the response.
app.Use(async (context, next) =>
{
    await next(context);
    var mathErrorFeature = context.Features.Get<MathErrorFeature>();
    if (mathErrorFeature is not null)
    {
        if (context.RequestServices.GetService<IProblemDetailsWriter>() is
            { } problemDetailsService)
        {

            if (problemDetailsService.CanWrite(new ProblemDetailsContext() { HttpContext = context }))
            {
                (string Detail, string Type) details = mathErrorFeature.MathError switch
                {
                    MathErrorType.DivisionByZeroError => ("Divison by zero is not defined.",
                        "https://en.wikipedia.org/wiki/Division_by_zero"),
                    _ => ("Negative or complex numbers are not valid input.",
                        "https://en.wikipedia.org/wiki/Square_root")
                };

                await problemDetailsService.WriteAsync(new ProblemDetailsContext
                {
                    HttpContext = context,
                    ProblemDetails =
                    {
                        Title = "Bad Input",
                        Detail = details.Detail,
                        Type = details.Type
                    }
                });
            }
        }
    }
});

// add controllers etc now.

````

  3. problem details from middleware
    - An alternative approach to using ProblemDetailsOptions with CustomizeProblemDetails is to set the ProblemDetails in middleware. A problem details response can be written by calling IProblemDetailsService.WriteAsync


- Produce a problem details payload for exceptions, info here: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling?view=aspnetcore-8.0#produce-a-problemdetails-payload-for-exceptions


- an alternative to produce problem details is to use a nuget package  


---

### Make HTTP Requests

- An implementation of `IHttpClientFactory` is available for creating `HttpClient` instances. The factory is superior to just using HttpClient because it:
  - Provides a central location for naming and configuring logical `HttpClient` instances. I.e., register and configure a github client for accessing Github.
  - Supports registration and chaining of multiple delegating handlers to buuld an outgoing request middleware pipeline. This pattern is similar to dotnet's inbound middleware pipeline. It provides a mechanism to manage cross-cutting concerns for HTTP requests including caching, error handling, serialization, and logging. 
  - Integrates with Polly - for transient fault handling. (circuit breaker)
  - Manages the pooling and lifetime of underlying `HttpClientHandler` instances to avoid common DNS problems that occur when managing `HttpClient` lifetimes manually. 
  - Adds a configurable logging experience via ILogger for all requests sent through clients created by the factory. 

- Basic Usage
  - register IHttpClientFactory by calling AddHttpClient in Program.cs: `builder.Services.AddHttpClient();`
  - an example of creating an instance:

````c#
public class BasicModel : PageModel
{
    private readonly IHttpClientFactory _httpClientFactory;

    public BasicModel(IHttpClientFactory httpClientFactory) =>
        _httpClientFactory = httpClientFactory;

    public IEnumerable<GitHubBranch>? GitHubBranches { get; set; }

    public async Task OnGet()
    {
        var httpRequestMessage = new HttpRequestMessage(
            HttpMethod.Get,
            "https://api.github.com/repos/dotnet/AspNetCore.Docs/branches")
        {
            Headers =
            {
                { HeaderNames.Accept, "application/vnd.github.v3+json" },
                { HeaderNames.UserAgent, "HttpRequestsSample" }
            }
        };

        var httpClient = _httpClientFactory.CreateClient();
        var httpResponseMessage = await httpClient.SendAsync(httpRequestMessage);

        if (httpResponseMessage.IsSuccessStatusCode)
        {
            using var contentStream =
                await httpResponseMessage.Content.ReadAsStreamAsync();
            
            GitHubBranches = await JsonSerializer.DeserializeAsync
                <IEnumerable<GitHubBranch>>(contentStream);
        }
    }
}
````

- named clients
  - a good choice when the app requires many distinct uses of `HttpClient` / many `HttpClient`s have different configuration

````c#
// an example of a named client:
builder.Services.AddHttpClient("GitHub", httpClient =>
{
    httpClient.BaseAddress = new Uri("https://api.github.com/");

    // using Microsoft.Net.Http.Headers;
    // The GitHub API requires two headers.
    httpClient.DefaultRequestHeaders.Add(
        HeaderNames.Accept, "application/vnd.github.v3+json");
    httpClient.DefaultRequestHeaders.Add(
        HeaderNames.UserAgent, "HttpRequestsSample");
});
````

- CreateClient
  - each time a createClient is called:
    - a new instance of HttpClient is created
    - the configuration action is called
  - to create a named client, pass its name into `CreateClient`: 

````c#
public class NamedClientModel : PageModel
{
    private readonly IHttpClientFactory _httpClientFactory;

    public NamedClientModel(IHttpClientFactory httpClientFactory) =>
        _httpClientFactory = httpClientFactory;

    public IEnumerable<GitHubBranch>? GitHubBranches { get; set; }

    public async Task OnGet()
    {
        var httpClient = _httpClientFactory.CreateClient("GitHub"); // 'Github' is the named clients name - same as above
        var httpResponseMessage = await httpClient.GetAsync(
            "repos/dotnet/AspNetCore.Docs/branches"); // note that no base address is passed, it uses the one in the config for the named client.

        if (httpResponseMessage.IsSuccessStatusCode)
        {
            using var contentStream =
                await httpResponseMessage.Content.ReadAsStreamAsync();
            
            GitHubBranches = await JsonSerializer.DeserializeAsync
                <IEnumerable<GitHubBranch>>(contentStream);
        }
    }
}
````

- Typed Clients (preferred over named clients)
  - same functionality as named clients but don't need to use strings as keys
  - provides intellisense when consuming clients 
  - provides a single location to configure and interact with a particular httpClient 
  - work with DI and injected where required.

````c#
public class GitHubService
{
    private readonly HttpClient _httpClient;

    public GitHubService(HttpClient httpClient)
    {
        _httpClient = httpClient;

        _httpClient.BaseAddress = new Uri("https://api.github.com/");

        // using Microsoft.Net.Http.Headers;
        // The GitHub API requires two headers.
        _httpClient.DefaultRequestHeaders.Add(
            HeaderNames.Accept, "application/vnd.github.v3+json");
        _httpClient.DefaultRequestHeaders.Add(
            HeaderNames.UserAgent, "HttpRequestsSample");
    }

    public async Task<IEnumerable<GitHubBranch>?> GetAspNetCoreDocsBranchesAsync() =>
        await _httpClient.GetFromJsonAsync<IEnumerable<GitHubBranch>>(
            "repos/dotnet/AspNetCore.Docs/branches");
}

// in startup.cs:
builder.Services.AddHttpClient<GitHubService>();

// typed client can then be consumed directly: 
public class TypedClientModel : PageModel
{
    private readonly GitHubService _gitHubService;

    public TypedClientModel(GitHubService gitHubService) =>
        _gitHubService = gitHubService;

    public IEnumerable<GitHubBranch>? GitHubBranches { get; set; }

    public async Task OnGet()
    {
        try
        {
            GitHubBranches = await _gitHubService.GetAspNetCoreDocsBranchesAsync();
        }
        catch (HttpRequestException)
        {
            // ...
        }
    }
}
````


- Make POST, PUT and DELETE requests
  - previous examples just showed get, but they are all supported
  - to do a POST you can do something like this:  `using var httpResponseMessage = await _httpClient.PostAsync("/api/TodoItems", todoItemJson);`


- use Polly-based handlers: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-8.0#use-polly-based-handlers 
  - IHttpClientFactory integrates with the nuget package Polly, which is a resilience and transient fault-handling library for dotnet.
    - allows retry, circuit breaker, timeout etc..

````c#
builder.Services.AddHttpClient("PollyMultiple")
    .AddTransientHttpErrorPolicy(policyBuilder =>
        policyBuilder.RetryAsync(3))
    .AddTransientHttpErrorPolicy(policyBuilder =>
        policyBuilder.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
````

---

## APIs

- Dotnet supports two approaches to creating APIs: a controller-based approach and minimal APIs
- Controllers derive from `ControllerBase`
  - controllers are classes that can take dependencies via construction injection or property injection and follow OOP patterns.
- Minimal APIs define endpoints with logical handlers in lambdas or methods
  - Minial apis hides the host class by default and focuses on config and extensibility via extension methods that take functions as lambda expressions

- Example of controller method:
````c#
// program.cs
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        builder.Services.AddControllers();
        var app = builder.Build();

        app.UseHttpsRedirection();

        app.MapControllers();

        app.Run();
    }
}

// WeatherForecastController.cs
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase
{
    private static readonly string[] Summaries = new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };

    private readonly ILogger<WeatherForecastController> _logger;

    public WeatherForecastController(ILogger<WeatherForecastController> logger)
    {
        _logger = logger;
    }

    [HttpGet(Name = "GetWeatherForecast")]
    public IEnumerable<WeatherForecast> Get()
    {
        return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            TemperatureC = Random.Shared.Next(-20, 55),
            Summary = Summaries[Random.Shared.Next(Summaries.Length)]
        })
        .ToArray();
    }
}
````

- the following code is the same but via minimal APIs:
````c#
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        var app = builder.Build();

        app.UseHttpsRedirection();

        var summaries = new[]
        {
            "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
        };

        app.MapGet("/weatherforecast", (HttpContext httpContext) =>
        {
            var forecast = Enumerable.Range(1, 5).Select(index =>
                new WeatherForecast
                {
                    Date = DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
                    TemperatureC = Random.Shared.Next(-20, 55),
                    Summary = summaries[Random.Shared.Next(summaries.Length)]
                })
                .ToArray();
            return forecast;
        });

        app.Run();
    }
}
````

- minimal APIs have the same capabilities as controller-based APIs bar a few
- in projects i've worked with we use controller-based apis for the main endpoints, minimal APIs are used for things like healthchecks and ping endpoints.

### Controller-based APIs
- should derivce from ControllerBase rather than Controller (controller adds supports for views, so its not for web API requests)
- ControllerBase class provides many properties and methods useful for handling HTTP requests, i.e. CreatedAtAction returns a 201 status code 
  - list of all methods is [here](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.controllerbase?view=aspnetcore-8.0)
````c#
[HttpPost]
[ProducesResponseType(StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public ActionResult<Pet> Create(Pet pet)
{
    pet.Id = _petsInMemoryStore.Any() ? 
             _petsInMemoryStore.Max(p => p.Id) + 1 : 1;
    _petsInMemoryStore.Add(pet);

    return CreatedAtAction(nameof(GetById), new { id = pet.Id }, pet);
}
````
- the ` Microsoft.AspNetCore.Mvc` namespace provides attributes that can be used to configure the behavior of web API controllers and action methods
- the below coase uses attributes to specify the supported HTTP action verb and any known HTTP status codes that could be returned:
````c#
[HttpPost] // http action verb
[ProducesResponseType(StatusCodes.Status201Created)] // status code
[ProducesResponseType(StatusCodes.Status400BadRequest)] // status code
public ActionResult<Pet> Create(Pet pet)
{
    pet.Id = _petsInMemoryStore.Any() ? 
             _petsInMemoryStore.Max(p => p.Id) + 1 : 1;
    _petsInMemoryStore.Add(pet);

    return CreatedAtAction(nameof(GetById), new { id = pet.Id }, pet);
}
````
- list of all methods is [here](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.controllerbase?view=aspnetcore-8.0)


- the `[ApiController]` attribute can be put on the controller class (see above the weatherforecastcontroller) to enable the following opinionated, api-specific behaviours:
  - Attribute routing requirement (i.e. you need `[Route("[controller]")]` under it)
  - Automatic HTTP 400 responses - model validation errors automatically trigger a http 400 response, no need to return a `BadRequest()`.
  - Binding source interface - defines the location at which an action parameters value is found (i.e. `[FromBody], [FromForm], [FromHeader], [FromQuery]`... etc)
  - problem details for error status codes - transforms an error result (a statys 400 or higher) to a result with `ProblemDetails`.
    - consider `if (pet == nul) return NotFound();` - this would return a problemDetails body like so:
````json
{
  type: "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  title: "Not Found",
  status: 404,
  traceId: "0HLHLV31KRN83:00000001"
}
````

### Minimal APIs
- A simplified approach for building fast HTTP APIs 
````c#
var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

app.MapGet("/users/{userId}/books/{bookId}", 
    (int userId, int bookId) => $"The user id is {userId} and book id is {bookId}");

app.Run();
````
- quick reference [here](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis?view=aspnetcore-8.0)

### OpenAPI
- the openAPI spec is a programming language-agnostic standard for documenting HTTP APIs
- 3 key aspects to integrating it into an app:
  - generating info about the endpoints in the app
  - gathering the info into a format that matches the openApi schema
  - exposing the generated openAPI schema via a visual UI or serialised file 
- more info [here](https://spec.openapis.org/oas/latest.html)

- easily done through SwashBuckle (Swagger) or NSwag 
  - Swagger was donated to openAPI, both names are used interchangably but Swagger is the tooling that uses openAPI specification

---

## Servers
- dotnet core apps use a HTTP server implementation to listen for HTTP request. The server surfaces requests to the app as a set of request features composed into a HttpContext 
- Kestrel is a cross-platform web server, it runs on windows / mac / linux. 
- IIS and HTTP.sys are for windows only. 

- Kestrel server is the default cross-platform HTTP server implementation, providing the best performance.
  - by itself it can be used as an edge server processing requests directly from a network, including the internet
  - with a reverse proxy server such as IIS, Nginx or Apache it can receives HTTP requests from the internet and forward them to Kestrel (internet => https => reverse proxy server => HTTP => kestrel => httpContext => app code)
    - using nginx with kestrel [here](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-8.0)

- Server startup
  - server is launched when the IDE starts the app
    - for visual studio / rider, launch profiles can be used to start the app and server 
  - the console can also be used to start the app, i.e. `dotnet run` which uses launchSettings.json. if launch profiles are present in launchSettings.json, use the `dotnet run --launch-profile {profileName}` option


### Kestrel 
- Configured by default in dot net 
- cross platform, high performance, wide protocol support 

````c#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
````

---

## Security and Identity 
- dot net allows devs to configure and manage security to build robust and secure apps 
- authentication: user provides credentials which are compared to those stored in a db. if matched, authenticated 
- authorisation: if authenticated, users can perform actions they are authorised for (what a user is allowed to do)

- common vulnerabilities in software
  - Cross-site scripting (XSS) attacks 
  - SQL injection attacks 
  - Cross-site Request Forgery (XSRF / CSRF) attacks 
  - Open redirect attacks 

### Authentication 
- the process of determining a users identity 
- in dotnet authentication is handled by the authentication service, `IAuthenticationService`, which is used by authentication middleware. 
  - the autheitcation services uses registered authentication handlers to complete authentication-related actions. Examples include:
    - authenticating a user 
    - responding when an unauthenticated user tries to access a restricted resource 
  - the registered authentication handlers and their configuration options are called "schemes".
  - authentication scheemes are specified by registering authentication services in Program.cs:
    - by calling a scheme-specific extension method after a call to `AddAuthentication`, such as `AddJwtBearer` or `AddCookie`
````c#
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme,
        options => builder.Configuration.Bind("JwtSettings", options))
    .AddCookie(CookieAuthenticationDefaults.AuthenticationScheme,
        options => builder.Configuration.Bind("CookieSettings", options));
````
  - the `AddAuthentication` param `JwtBearerDefaults.AuthenticationScheme` is the name of the scheme to use by default when a specific scheme isn't requested. 


- In some cases, the call to `AddAuthentication` is automatically made by other extension methods. I.e. when using Identity, AddAuthentication is called internally. 
- the authentication middleware is added in Program.cs by calling `UseAuthentication`. This registers the middleware that uses the previously registered authentication scehemes. Call `useAuthentication` before any middleware that depends on users being authenticated. 

Authentication Concepts 
- Authentication is responsible for providing the `ClaimsPrincipal` for authorisation to make permission decisions against 
- There are different scheme approaches to select which authentication handler is responsible for generating the correct set of claims 
  - authentication sceheme
  - directly set HttpContext.User
- if only one scheme is registered, it because the default. if multiple schemes are registered, you must specify one in the authorize attribute or an error is thrown 

authentication Scheme 
- this scheme selects which authentication handler is responsible for generating the correct set of claims
- the scheme name corresponds to an authentication handler 

authentication handler 
- this is a type that implements the behavior of a scheme 
- derived from `IAuthenticationHandler` 
- has the primary responsibility to authenticate users 
- the auth handler constructs the AuthenticationTicket objects representing the users identity (if successful, returns 'no result' if unsuccessful)

authenticate 
- an authentication scheme's authenticate action is responsible for constructing the users identity based on request context. 
- returns an authenticateResult indicating whether auth was successful, and if so, the users identity in an authentication ticket. i.e.:
  - a cookie auth scheme constructing the users identity from cookies 
  - a JWT bearer scheme deserializing and validating a JWT bearer token t oconstruct the users identity 

Challenge 
- an authentication challenge is invoked by Authorisation when an unauthenticated user requests and endpoint that requires authentication
- challenge examples include:
  - a cookie auth scheme redirecting the user to a login page 
  - a JWT bearer scheme returning a 401 with a `www-authenticate: bearer` header 

Forbid 
- an auth scheme's forbid action is called by authorization when an authenticated user attempts to access a resource they're not allowed to. i.e.:
  - a cookie auth scheme redirecting the user to a page indicating access is forbidden
  - a JWT bearer scheme returning a 403 result 
  - a custom auth scheme redirecting to a page where the user can request access 

#### Identity 
https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-8.0&tabs=visual-studio

- .net identity is an API that supports UI login functionality - managing users, passwords, profile data, roles, claims, tokens etc
- typically configured using an SQL server DB to store user names, passwords and profile data 
- .net identity adds user UI login functionality. to secure web APIs you need to use something else 

### Authorisation 
- Authorization refers to the process that determines what a user is able to do. E.g. an admin user is allowed to create a document library, add documents, edit documents and delete them. an non-admin user is only allowed to read them. 
- authorization is independent from authentication, however authorization requires an authentication mechanism. 
- Authorisation Types:
  - .net authorization provides a simple declarative role and a rich policy-based model
  - expressed in requirements, and handlers evaluate a users claims against requirements 
- Authorization is controlled with the `[Authorize]` attribute and its various parameters (i.e. on a controller).
````c#
[Authorize] // protect the entire controller
public class AccountController : Controller
{
    public ActionResult Login()
    {
    }

    //[Authorize] //alternatively you could just put it on some endpoints
    public ActionResult Logout()
    {
    }
}

// authorise based on roles: https://learn.microsoft.com/en-us/aspnet/core/security/authorization/roles?view=aspnetcore-8.0
[Authorize(Roles = "Administrator")]
public class AdministrationController : Controller
{
    public IActionResult Index() =>
        Content("Administrator");
}

// authorise based on claims or policies: 
// claims: https://learn.microsoft.com/en-us/aspnet/core/security/authorization/claims?view=aspnetcore-8.0
// policies: https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-8.0
[Authorize(Policy = "EmployeeOnly")]
public IActionResult VacationBalance()
{
    return View();
}



````
- custom authorizors can also be used, see [here](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/iard?view=aspnetcore-8.0)
- authorise based on roles:

### Secrets Management
- How to manage sensitive data for a .net core app on a development machine
- never store passwords or sensitive data in source code, access them for prod through somewhere like AWS param store or secrets manager.
- consider an app where a default db connection string is included in the appsettings.json file with the key DefaultConnection.
  - this string is for the localDB, which doesn't require a password. During deployment, this key value can be overridden with an environment variables value 

### Performance 

#### Caching 
- In-memory caching
  - uses server memory to store cached data
  - suitable for a single server or multiple servers using session affinity (also known as sticky sessions, i.e. the request from a client are always routed to the same server for processing)
- Distributed Cache
  - used when the app is hosted in a cloud or server farm
  - cache is shared across the servers that process requests 
  - a client can submit a request thats handled by any server in the group if cached data for the client is available
  - works with SQL server, redis and NCache
````c#
public class SomeService(IDistributedCache cache)
{
    public async Task<SomeInformation> GetSomeInformationAsync
        (string name, int id, CancellationToken token = default)
    {
        var key = $"someinfo:{name}:{id}"; // Unique key for this combination.
        var bytes = await cache.GetAsync(key, token); // Try to get from cache.
        SomeInformation info;
        if (bytes is null)
        {
            // Cache miss; get the data from the real source.
            info = await SomeExpensiveOperationAsync(name, id, token);

            // Serialize and cache it.
            bytes = SomeSerializer.Serialize(info);
            await cache.SetAsync(key, bytes, token);
        }
        else
        {
            // Cache hit; deserialize it.
            info = SomeSerializer.Deserialize<SomeInformation>(bytes);
        }
        return info;
    }

    // This is the work we're trying to cache.
    private async Task<SomeInformation> SomeExpensiveOperationAsync(string name, int id,
        CancellationToken token = default)
    { /* ... */ }
}
````
- HybridCache
  - recommended over the above two because of its simplicity of its api
  - an abstract class with a default implementation that handles most aspects of saving to cache and retrieving from cache
  - bridges some gaps in distributed cache and in-memory cache
  - it has stampede protection (a frequently used cache entry is revoked and too many requests try to repopulated the same cache entry at the same time. it uses concurrent operations, ensuring that all requests for a given response wait for the first request to populated the cache)
````c#
public class SomeService(HybridCache cache)
{
    public async Task<SomeInformation> GetSomeInformationAsync
        (string name, int id, CancellationToken token = default)
    {
        return await cache.GetOrCreateAsync(
            $"someinfo:{name}:{id}", // Unique key for this entry.
            async cancel => await SomeExpensiveOperationAsync(name, id, cancel),
            token: token
        );
    }
}
````


#### Rate Limiting Middleware 
https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-8.0

- 

