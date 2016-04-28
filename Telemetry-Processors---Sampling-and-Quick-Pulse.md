[Telemetry Processors](https://azure.microsoft.com/en-us/documentation/articles/app-insights-api-filtering-sampling/#filtering-itelemetryprocessor) are enabled in full framework (currently not supported in .NET Core), when Application Insights (>= [1.0.0-RC1-Update4](https://github.com/Microsoft/ApplicationInsights-aspnetcore/releases/tag/v1.0.0-rc1-update4)) is installed to monitor your live ASP.NET 5 web app.

Telemetry processors, are configured as a part of telemetry configuration. From (>= [1.0.0-RC1-Update4](https://github.com/Microsoft/ApplicationInsights-aspnetcore/releases/tag/v1.0.0-rc1-update4)), ```TelemetryConfiguration.Active``` is enabled as the default telemetry configuration to configure modules and telemetry initializers. Once telemetry configuration instance is available, telemetry processors can be made available to the configuration. Following describes the process of accessing telemetry configuration, using custom telemetry processors, enabling sampling and activating quick pulse.

> Note: It is best advised to use telemetry processors in the method ```ConfigureServices``` of application ```startup.cs```.

## Accessing Telemetry Configuration

From (>= 1.0.0-RC1-Update4), telemetry configuration can be accessed either using ```Active```

``` c#
var telemetryConfiguration = TelemetryConfiguration.Active
``` 

or by getting it from services 

``` c#
var telemetryConfiguration = services.GetRequiredService<TelemetryConfiguration>();
```

## Using Custom Telemetry Processor

Custom telemetry processors can be created to enable the process of filtering the telemetry, as described in [Create Custom Telemetry Processor](https://azure.microsoft.com/en-us/documentation/articles/app-insights-api-filtering-sampling/#filtering-itelemetryprocessor). Custom telemetry processors can be made available to configuration through ```TelemetryProcessorChainBuilder``` as described below:

``` c#
var builder = telemetryConfiguration.TelemetryProcessorChainBuilder;
builder.Use((next) => new CustomTelemetryProcessor(next));

// If you have more processors:
builder.Use((next) => new AnotherCustomTelemetryProcessor(next));

builder.Build();
```

## Sampling

[Sampling](https://azure.microsoft.com/en-us/documentation/articles/app-insights-sampling) is achieved through telemetry processors. Two basic types of sampling are available:

* [Fixed rate sampling](https://azure.microsoft.com/en-us/documentation/articles/app-insights-sampling/#fixed-rate-sampling-for-aspnet-web-sites)
* [Adaptive sampling](https://azure.microsoft.com/en-us/documentation/articles/app-insights-sampling/#adaptive-sampling-at-your-web-server)

Adaptive sampling is by default enabled for the applications. The default sampling feature can be disabled when we add Application Insights service in the method ```ConfigureServices```, using ```ApplicationInsightsServiceOptions```:

``` c#
var aiOptions = new Microsoft.ApplicationInsights.AspNet.Extensions.ApplicationInsightsServiceOptions();
aiOptions.EnableAdaptiveSampling = false;

services.AddApplicationInsightsTelemetry(Configuration, aiOptions);
```

Sampling can be manually enabled using extension methods of ```TelemetryProcessorChainBuilder``` as described below:

``` c#
var builder = telemetryConfiguration.TelemetryProcessorChainBuilder;
// Using adaptive sampling
builder.UseAdaptiveSampling();
 
// Using fixed rate sampling   
double fixedSamplingPercentage = 50;
builder.UseSampling(fixedSamplingPercentage);

builder.Build();
```

## Quick Pulse

[Quick pulse](https://azure.microsoft.com/en-us/documentation/articles/app-insights-sampling) captures the live metrics and provides the current working scenario of the application. Quick pulse is not enabled by default in AspNet 5 applications. In order to enable the feature, ```QuickPulseTelemetryProcessor```, as well as ```QuickPulseTelemetryModule``` should be registered with the telemetry configuration. 

It is best advised to have ```QuickPulseTelemetryProcessor``` as the first telemetry processor, which can ensure that all the telemetry (before filtered/sampled) is gone through live metrics. So, the default adaptive sampling has to be disabled first, register the quick pulse module and processor and then add adaptive sampling telemetry processor back to the configuration, as described below:

The function call 

``` c#
services.AddApplicationInsightsTelemetry(Configuration);
``` 

in ```ConfigureServices``` of application ```startup.cs``` should be replaced by

``` c#
// Disable the default adaptive sampling feature
var aiOptions = new Microsoft.ApplicationInsights.AspNet.Extensions.ApplicationInsightsServiceOptions();
aiOptions.EnableAdaptiveSampling = false;
services.AddApplicationInsightsTelemetry(Configuration, aiOptions);

// Initialize QuickPulseTelemetryModule
var module = new QuickPulseTelemetryModule();
module.Initialize(TelemetryConfiguration.Active);

// Use and Register QuickPulseTelemetryProcessor
QuickPulseTelemetryProcessor processor = null; 
var builder = TelemetryConfiguration.Active.TelemetryProcessorChainBuilder;
builder.Use((next) => {
    processor = new QuickPulseTelemetryProcessor(next);
    module.RegisterTelemetryProcessor(processor);
    return processor;
} );

// Re-enable Adaptive sampling
builder.UseAdaptiveSampling();

// Build the processors
builder.Build();
```

As a result, live metrics will be collected as soon as the application is run.