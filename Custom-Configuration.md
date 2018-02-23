Set your Instrumentation Key
============================
There are two ways to set instrumentation key - using configuration or via code

Configuration-based approach
----------------------------
If you are using json configuration provider - add the following into config.json
```
"ApplicationInsights": {
    "InstrumentationKey": "11111111-2222-3333-4444-555555555555"
}
```
If you are using environment variables:
```
SET ApplicationInsights:InstrumentationKey=11111111-2222-3333-4444-555555555555
```
***note:*** *Environment variable that is set by azure website (APPINSIGHTS_INSTRUMENTATIONKEY) is not supported.*

Or any other configuration provider format you defined in your application (configuration setting name is ApplicationInsights:InstrumentationKey) before the call to add insights telemetry in Startup.cs:
```
services.AddApplicationInsightsTelemetry(Configuration);
```

Code-based approach
-------------------
Assuming you need to change instrumentation key after Startup.cs. You need to get access to ```IServiceProvider``` to get ```TelemetryClient```. This telemetry client will be used by ApplicationInsights middleware so changing it's instrumentation key will change:
```
this.Resolver.GetService<TelemetryClient>().Context.InstrumentationKey = "11111111-2222-3333-4444-555555555555";
```

Alternately, modify the key in the TelemetryConfiguration.Active instance directly.
```
TelemetryConfiguration.Active.InstrumentationKey = "ikeyhere";
```


Track custom trace/event/metric
===============================
Please refer to [Application Insights custom metrics API reference](http://azure.microsoft.com/en-us/documentation/articles/app-insights-custom-events-metrics-api/) for description of custom data reporting in Application Insights.

Get TelemetryClient using dependency injection: 
``` c#
public class HomeController : Controller
{
    private TelemetryClient telemetry;

    public HomeController(TelemetryClient telemetry)
    {
        this.telemetry = telemetry;
    }
```

Track event:
```
    public IActionResult Index()
    {
        this.telemetry.TrackEvent("HomePageRequested");
        return View();
    }

```

Add application level context properties
========================================
You may want to set additional properties on all telemetry data your application reports. Since it's a global setting you need to do it in "Configure" method of your Startup class:
```
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerfactory)
{
    app.ApplicationServices.GetService<TelemetryClient>().Context.Properties["Environment"] = env.EnvironmentName;
```


Add additional telemetry item properties
========================================
Add properties for request data item
------------------------------------
If you want to add additional properties for request telemetry you need to get an instance of ```RequestTelemetry``` from ```HttpContext.Features```:
``` c#
public class HomeController : Controller
{
    private RequestTelemetry requestTelemetry;

    public HomeController()
    {
        this.requestTelemetry = HttpContext.Features.Get<RequestTelemetry>();
    }
```
and then add required properties:
```
public IActionResult Index()
{
    this.requestTelemetry.Context.Properties["PageImportance"] = "High";
```
***Note:*** *You should not expect that ```RequestTelemetry``` will already be populated with data. Telemetry initializers will be called and will populate RequestTelemetry only when any telemetry item was tracked (send to the portal).*


Add request-level properties for all telemetry items
---------------------------------------------------- 
In order to set request level context properties for all telemetry items you need to configure your telemetry initializer.

First, define new class with IHttpContextAccessor dependency. This class will get property from the current http context and set corresponding property on telemetry data item:
``` c#
public class PropertyTelemetryInitializer : ITelemetryInitializer
{
    IHttpContextAccessor httpContextAccessor;

    public PropertyTelemetryInitializer(IHttpContextAccessor httpContextAccessor)
    {
        this.httpContextAccessor = httpContextAccessor;
    }

    public void Initialize(ITelemetry telemetry)
    {
        telemetry.Context.Properties["tenantName"] = httpContextAccessor.Value.Items["tenantName"].ToString();
    }
}
```
Secondly, register your telemetry initializer into global collection:
```
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerfactory)
{
    var initializer = new PropertyTelemetryInitializer(app.ApplicationServices.GetService<IHttpContextAccessor>());
    var configuration= app.ApplicationServices.GetService<TelemetryConfiguration>();
    configuration.TelemetryInitializers.Add(initializer);
``` 

Configure Telemetry initializers
=====================================
## Adding new TelemetryInitializer.
Create new Initializer.

Register it into DI Containers before AddApplicationInsightsTelemetry() is called in the ConfigureServices of your Startup class.
For eg:
```
services.AddSingleton<ITelemetryInitializer>(new MyCustomTelemetryInitializer());
```
This will ensure the telemetry initializer will be part of the TelemetryConfiguration object.

Alternately, one can obtain the TelemetryConfiguration object (either from DI or by using TelemetryConfiguration.Active), and add new initializers.
```
TelemetryConfiguration.Active.TelemetryInitializers.Add(new MyCustomTelemetryInitializer())
```
or 
```
var configuration= app.ApplicationServices.GetService<TelemetryConfiguration>();
configuration.TelemetryInitializers.Add(new MyCustomTelemetryInitializer())
```

Similar approach can be used to remove all TelemetryInitializers. (or remove a specific one)
``` c#
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerfactory)
{
    var configuration= app.ApplicationServices.GetService<TelemetryConfiguration>();
    configuration.TelemetryInitializers.Clear();
```

Redirect traffic to the different endpoint
==========================================

Configuration-based approach
----------------------------
If you are using json configuration provider - add the following into config.json
```
"ApplicationInsights": {
    "InstrumentationKey": "ikey",
    "TelemetryChannel": {
      "EndpointAddress": "http://localhost:8888/v2/track"
    }
  }
```
Then use UseApplicationInsights() method to add application insights.

If using AddApplicationInsightsTelemetry(), construct Configuration first and pass it to AddApplicationInsightsTelemetry in ConfigureServices method of Startup class as shown in below example.
```C#
var configBuilder = new ConfigurationBuilder()
               .SetBasePath(this.HostingEnvironment.ContentRootPath)
               .AddJsonFile("appsettings.json", true)               
               .AddEnvironmentVariables();            
            services.AddApplicationInsightsTelemetry(configBuilder.Build());
```