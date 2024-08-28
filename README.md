# Dotnet Deep Dive 

Contents:
- [Fundamentals](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/?view=aspnetcore-8.0)
  - [App start up](#app-startup)
  - [Dependency Injection (Services)](#dependency-injection-services)
  - [Middleware](#middleware)
    - [Custom middleware example](#example-of-writing-a-custom-middleware)
  - [Host](#host)
  - [Servers](#servers)
  - [Configuration](#configuration)
  - [Environments](#environments)
  - [Logging and monitoring](#logging-and-monitoring)
      - [Health checks](#health-checks)
  - [HttpContext](#httpcontext)
  - Routing
  - Handle errors
  - Make HTTP requests
  - Static Files 
- [APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/apis?view=aspnetcore-8.0) 
  - Controller-based APIs
  - Minimal APIs
- [Best practices](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices?view=aspnetcore-8.0)
- [Servers](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/?view=aspnetcore-8.0&tabs=windows)
  - Kestrel 
  - IIS
- [Security and Identity](https://learn.microsoft.com/en-us/aspnet/core/security/?view=aspnetcore-8.0)
  - Authentication
  - Authorization
  - Data protection
  - Secrets Management
  - Enforce HTTPS
  - Host docker with HTTPs
  - Docker Compose with HTTPS
  - GDPR support 
  - Prevent XSRF / CSRF attacks 
  - Prevent open redirect attacks
  - Prevent XSS attacks 
  - Enable CORS requests
  - Share cookies among apps 
  - SameSite cookies
  - IP safelist 
  - OWASP 
- [Performance](https://learn.microsoft.com/en-us/aspnet/core/performance/overview?view=aspnetcore-8.0)
  - Caching 
  - Rate limiting middleware
  - Timeouts middleware
  - Memory and GC
  - Object reuse with ObjectPool
  - Response compression
  - Diagnostic tools
  - Load and stress testing 
  - Event counters 

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


### Servers
  - dotnet core apps use a HTTP server implementation to listen for HTTP request. The server surfaces requests to the app as a set of request features composed into a HttpContext 
  - Kestrel is a cross-platform web server, it runs on windows / mac / linux. 
  - IIS and HTTP.sys are for windows only. 

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


---

- Routing 
  - A Route is a URL pattern mapped to a handler. The handler is typically a controller or a middleware. Dotnet routing gives you control over the URLs used by your app. 

- Make HTTP Requests
  - An implementation of `IHttpClientFactory` is available for creating `HttpClient` instances. The factory:
    - Provides a central location for naming and configuring logical `HttpClient` instances. I.e., register and configure a github client for accessing Github.
    - Supports registration and chaining of multiple delegating handlers to buuld an outgoing request middleware pipeline. This pattern is similar to dotnet's inbound middleware pipeline. It provides a mechanism to manage cross-cutting concerns for HTTP requests including caching, error handling, serialization, and logging. 
    - Integrates with Polly - for transient fault handling. (circuit breaker)
    - Manages the pooling and lifetime of underlying `HttpClientHandler` instances to avoid common DNS problems that occur when managing `HttpClient` lifetimes manually. 
    - Adds a configurable logging experience via ILogger for all requests sent through clients created by the factory. 


  /// note: up to: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/use-http-context?view=aspnetcore-8.0