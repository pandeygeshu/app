Application Insights SDK is integrated with Asp.Net Core's [logging infra](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging).
When running the application from Visual Studio IDE, all traces logged via ```ILogger``` interface is automatically captured. If an ikey is configured, then these traces are sent to Application Insights service. To limit the level of logging captured by Application Insights when ran from Visual Studio see this [issue](https://github.com/Microsoft/ApplicationInsights-aspnetcore/issues/603)

Following shows how to configure Application Insights to collect traces logged via ```ILogger``` interface.

Application Insights SDK for Asp.Net Core provides an extension method ```ILoggerFactory``` on ```ILoggerFactory``` to configure logging. Modify the ```Startup.cs``` class of your application as follows to enable Application Insights logging.
```
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
/*...existing code..*/
            loggerFactory.AddApplicationInsights(app.ApplicationServices, LogLevel.Warning);

}
```
This above snippet causes all messages (Warning or above) to be sent to Application Insights. 

As of today, this is the only way to enable application insights traces. We will be adding new extension methods on ```ILoggingBuilder``` (soon)[https://github.com/Microsoft/ApplicationInsights-aspnetcore/issues/536].