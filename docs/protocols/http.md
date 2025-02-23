---
id: http
title: HTTP
sidebar_position: 0
---

import ConsoleMetricsImage from './img/console_metrics.jpg';
import HTMLHistoryMetricsImage from './img/html_history_metrics.jpg';

To work with HTTP, NBomber provides [NBomber.Http](https://github.com/PragmaticFlow/NBomber.Http) plugin that includes:
- API to create, send request, receive response with tracking of data transfer and status codes.
- [HttpMetricsPlugin](#httpmetricsplugin) to get real-time metrics about the current Http connections.

:::info
To install [NBomber.Http](https://www.nuget.org/packages/nbomber.http) package you should execute the following *dotnet* command:

[![build](https://github.com/PragmaticFlow/NBomber.Http/actions/workflows/build.yml/badge.svg)](https://github.com/PragmaticFlow/NBomber.Http)
[![NuGet](https://img.shields.io/nuget/v/nbomber.http.svg)](https://www.nuget.org/packages/nbomber.http/)

```code
dotnet add package NBomber.Http
```
:::

## HTTP API

HTTP plugin provides helper methods that works with native [HttpClient](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclient?view=net-8.0). These methods help reduce the boilerplate needed to build `HttpRequestMessage`, send or receive JSON objects, calculate data transfer, etc. 

Basic Example:

```csharp
using var httpClient = new HttpClient();

var scenario = Scenario.Create("http_scenario", async context =>
{   
    var request =
        Http.CreateRequest("GET", "https://nbomber.com")
            .WithHeader("Content-Type", "application/json")
            .WithBody(new StringContent("{ some JSON }", Encoding.UTF8, "application/json"));

    var response = await Http.Send(httpClient, request);

    return response;
});

```

Advanced Example:

```csharp
using var httpClient = new HttpClient();

var scenario = Scenario.Create("http_scenario", async context =>
{
    var step1 = await Step.Run("step_1", context, async () =>
    {
        var request =
            Http.CreateRequest("GET", "https://nbomber.com")
                .WithHeader("Content-Type", "application/json")
                .WithBody(new StringContent("{ some JSON }", Encoding.UTF8, "application/json"));

        var response = await Http.Send(httpClient, request);

        return response;
    });

    // example of sending JSON as a body 
    // Http.WithJsonBody<T>() will automatically serialize object to JSON
    // the header "Content-Type": "application/json" will be added automatically

    var step2 = await Step.Run("step_2", context, async () =>
    {
        var user = new UserData { UserId = 1, Title = "test user" };

        var request = 
            Http.CreateRequest("POST", "https://nbomber.com")
                .WithJsonBody(user);
        
        var response = await Http.Send(httpClient, request);

        return response;
    });

    // example of using Http.Send<TResponse>
    // it sends HTTP request 
    // and deserialize JSON response to specific type

    var step3 = await Step.Run("step_3", context, async () =>
    {
        var request = 
            Http.CreateRequest("GET", "https://jsonplaceholder.typicode.com/todos/1")
                .WithHeader("Content-Type", "application/json");

        var response = await Http.Send<UserData>(httpClient, request);

        // user: UserData        
        var user = response.Payload.Value.Data;
        var userId = user.UserId;

        return response;
    });

    // example of using CancellationToken for timeout operation

    var step4 = await Step.Run("step_4", context, async () =>
    {
        using var timeout = new CancellationTokenSource();
        timeout.CancelAfter(50); // the operation will be canceled after 50 ms

        var clientArgs = HttpClientArgs.Create(timeout.Token);

        var request = 
            Http.CreateRequest("GET", "https://jsonplaceholder.typicode.com/todos/1")
                .WithHeader("Content-Type", "application/json");

        var response = await Http.Send(httpClient, clientArgs, request);

        return response;
    });

    return Response.Ok();
});
```

### CreateRequest

This method should be used to create HTTP request. 

```csharp
public static HttpRequestMessage CreateRequest(string method, string url)
```

Example:

```csharp
var scenario = Scenario.Create("http_scenario", async context =>
{
    var request = Http.CreateRequest("GET", "https://nbomber.com")
                      .WithHeader("Content-Type", "text/html");
                      // .WithHeader("Content-Type", "application/json")
                      // .WithBody(new StringContent("{ id: 1 }", Encoding.UTF8, "application/json");
                      // .WithBody(new ByteArrayContent(new [] {1,2,3}))
    ... 
});
```

### Send

This method should be used to send HTTP request.

```csharp
public static Task<Response<HttpResponseMesage>> Send(HttpClient client, HttpRequestMessage request);

public static Task<Response<HttpResponseMesage>> Send(HttpClient client, HttpClientArgs clientArgs, HttpRequestMessage request);
```

Example 1:

```csharp
using var httpClient = new HttpClient();

var scenario = Scenario.Create("http_scenario", async context =>
{
    var request = Http.CreateRequest("GET", "https://nbomber.com")
                      .WithHeader("Content-Type", "text/html");
                      // .WithHeader("Content-Type", "application/json")
                      // .WithBody(new StringContent("{ id: 1 }", Encoding.UTF8, "application/json");
                      // .WithBody(new ByteArrayContent(new [] {1,2,3}))
    
    var response = await Http.Send(httpClient, request);

    return response;
});
```

Example 2: in this example we use [HttpClientArgs](#httpclientargs).

```csharp
using var httpClient = new HttpClient();

var scenario = Scenario.Create("http_scenario", async context =>
{
    var request = Http.CreateRequest("GET", "https://nbomber.com")
                      .WithHeader("Content-Type", "application/json");
                      // .WithHeader("Content-Type", "application/json")
                      // .WithBody(new StringContent("{ id: 1 }", Encoding.UTF8, "application/json");
                      // .WithBody(new ByteArrayContent(new [] {1,2,3}))    
    
    var clientArgs = HttpClientArgs.Create(
        CancellationToken.None,
        httpCompletion: HttpCompletionOption.ResponseContentRead,
        jsonOptions: JsonSerializerOptions.Default
    );
            
    var response = await Http.Send(httpClient, clientArgs, request);

    return response;
});
```

*You can find the complete example by this [link](https://github.com/PragmaticFlow/NBomber/blob/dev/examples/Demo/HTTP/HttpClientArgsExample.cs).*

### JSON support

HTTP plugin provides helper methods that simplify working with JSON format.

- `Http.WithJsonBody<T>(data)` - Populates request body by serializing data record to JSON format. Also, it adds HTTP header: *"Content-Type": "application/json"*.

```csharp
using var httpClient = new HttpClient();

Http.GlobalJsonSerializerOptions = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase
};

var scenario = Scenario.Create("http_scenario", async context =>
{    
    var user = new UserData { UserId = 1, Title = "anton" };

    var request =
        Http.CreateRequest("GET", "https://nbomber.com")
            .WithJsonBody(user);          
            
    var response = await Http.Send(httpClient, request);

    return response;
});
```

*You can find the complete example by this [link](https://github.com/PragmaticFlow/NBomber/blob/dev/examples/Demo/HTTP/HttpSendJsonExample.cs).*

- `Http.Send<TResponse>` - Send request and deserialize HTTP response body to JSON format.

```csharp
using var httpClient = new HttpClient();

Http.GlobalJsonSerializerOptions = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase
};

var scenario = Scenario.Create("http_scenario", async context =>
{
    var request =
        Http.CreateRequest("GET", "https://jsonplaceholder.typicode.com/todos/1")
            .WithHeader("Content-Type", "application/json");

    var response = await Http.Send<UserData>(httpClient, request);

    var userData = response.Payload.Value.Data;
    var title = userData.Title;
    var userId = userData.UserId;

    return response;    
});
```

*You can find the complete example by this [link](https://github.com/PragmaticFlow/NBomber/blob/dev/examples/Demo/HTTP/HttpSendJsonExample.cs).*

:::info
For JSON serialization you can set global serializer options:

```csharp
Http.GlobalJsonSerializerOptions = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase
};
```

If the global serializer options doesn't fit for your use case and you need more granular control, you can pass different serializer options for each request via **HttpClientArgs**.

```csharp
var clientArgs = HttpClientArgs.Create(
    CancellationToken.None,
    httpCompletion: HttpCompletionOption.ResponseContentRead,
    jsonOptions: JsonSerializerOptions.Default // you set custom options
);
        
var response = await Http.Send(httpClient, clientArgs, request);
```
:::

### Timeout operation

```csharp
var scenario = Scenario.Create("http_scenario", async context =>
{
    using var timeout = new CancellationTokenSource();
    timeout.CancelAfter(50); // the operation will be canceled after 50 ms

    var clientArgs = HttpClientArgs.Create(timeout.Token);

    var request = 
        Http.CreateRequest("GET", "https://jsonplaceholder.typicode.com/todos/1")
            .WithHeader("Content-Type", "application/json");

    var response = await Http.Send(httpClient, clientArgs, request);

    return response;
});
```

*You can find the complete example by this [link](https://github.com/PragmaticFlow/NBomber/blob/dev/examples/Demo/HTTP/HttpWithTimeoutExample.cs).*

### HttpClientArgs

HttpClientArgs represents a structure that can configure HTTP clients per request. You can use it to set request timeout, or `JsonSerializerOptions`.

Example:

```csharp
var scenario = Scenario.Create("http_scenario", async context =>
{
    var request =
        Http.CreateRequest("GET", "https://nbomber.com")
            .WithHeader("Content-Type", "application/json");
            .WithBody(new StringContent("{ some JSON }", Encoding.UTF8, "application/json"));

    var clientArgs = HttpClientArgs.Create(
        CancellationToken.None,
        httpCompletion: HttpCompletionOption.ResponseHeadersRead, // or ResponseContentRead
        jsonOptions: new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        }
    );

    var response = await Http.Send(httpClient, clientArgs, request);

    return response;
});
```

*You can find the complete example by this [link](https://github.com/PragmaticFlow/NBomber/blob/dev/examples/Demo/HTTP/HttpClientArgsExample.cs).*

### Tracing HTTP requests

The HTTP plugin supports tracing requests and corresponding responses. To do this, you need to pass `Logger` into `HttpClientArgs`.

```csharp
using var httpClient = new HttpClient();

var scenario = Scenario.Create("http_scenario", async context =>
{
    var request =
        Http.CreateRequest("GET", "https://jsonplaceholder.typicode.com/todos/1")
            .WithHeader("Content-Type", "application/json");

    // highlight-start
    var clientArgs = HttpClientArgs.Create(logger: context.Logger);
    // highlight-end

    var response = await Http.Send(httpClient, clientArgs, request);

    return response;
})
```

After running this example, we will have a log file populated with HTTP tracing. Each trace message contains correspondent `TraceId`.

```csharp title="nbomber-log-20240406.txt"
2024-04-06 10:56:11.090 +03:00 [DBG] [ThreadId:7] HTTP Request:
// highlight-start
 TraceId: 30774c79337a4d3e9c6fa897bdb87ad1
// highlight-end
 Method: GET
 RequestUri: "https://jsonplaceholder.typicode.com/todos/1"
 HttpVersion: 1.1
 Headers: Content-Type: application/json
 Content: 

2024-04-06 10:56:11.994 +03:00 [DBG] [ThreadId:9] HTTP Response:
// highlight-start
 TraceId: 30774c79337a4d3e9c6fa897bdb87ad1
// highlight-end
 HttpVersion: 1.1
 StatusCode: "OK"
 ReasonPhrase: OK
 Headers: Date: Sat, 06 Apr 2024 07:56:10 GMT, Connection: keep-alive, Report-To: {"group":"heroku-nel","max_age":3600,"endpoints":[{"url":"https://nel.heroku.com/reports?ts=1712102103&sid=e11707d5-02a7-43ef-b45e-2cf4d2036f7d&s=NEDTWuReF%2Ftx6jFV3Ve%2FTGNlbenGGlra6iQD4Wu%2BL1k%3D"}]}, Reporting-Endpoints: heroku-nel=https://nel.heroku.com/reports?ts=1712102103&sid=e11707d5-02a7-43ef-b45e-2cf4d2036f7d&s=NEDTWuReF%2Ftx6jFV3Ve%2FTGNlbenGGlra6iQD4Wu%2BL1k%3D, Nel: {"report_to":"heroku-nel","max_age":3600,"success_fraction":0.005,"failure_fraction":0.05,"response_headers":["Via"]}, X-Powered-By: Express, X-Ratelimit-Limit: 1000, X-Ratelimit-Remaining: 999, X-Ratelimit-Reset: 1712102115, Vary: Origin, Accept-Encoding, Access-Control-Allow-Credentials: true, Cache-Control: max-age=43200, Pragma: no-cache, X-Content-Type-Options: nosniff, ETag: W/"53-hfEnumeNh6YirfjyjaujcOPPT+s", Via: 1.1 vegur, CF-Cache-Status: HIT, Age: 23727, Accept-Ranges: bytes, Server: cloudflare, CF-RAY: 8700384848945b45-VIE, Alt-Svc: h3=":443"
 Content: {
  "userId": 1,
  "id": 1,
  "title": "delectus aut autem",
  "completed": false
}
```

## HttpMetricsPlugin

HttpMetricsPlugin - provides a monitoring layer for HTTP connections.

Example:

```csharp
NBomberRunner
    .RegisterScenarios(scenario)
    .WithWorkerPlugins(new HttpMetricsPlugin(new [] { HttpVersion.Version1 }))
    // .WithWorkerPlugins(new HttpMetricsPlugin(new [] { HttpVersion.Version1, HttpVersion.Version2 }))
    .Run();
```

After running with HttpMetricsPlugin, you will get real-time HTTP connections metrics for the Console output:

<center><img src={ConsoleMetricsImage} width="50%" height="50%" /></center>

Also, after running with HttpMetricsPlugin, you will get HTTP connections history metrics for the HTML report:

<center><img src={HTMLHistoryMetricsImage} width="100%" height="100%" /></center>

## Connections limit

You may need to limit the sockets connections count.

```csharp
var socketsHandler = new SocketsHttpHandler
{
    MaxConnectionsPerServer = 3
};

using var httpClient = new HttpClient(socketsHandler);
```

## Best practices

### Blog posts
- [Load Testing HTTP API on C# with NBomber](../../blog/2023/08/16/load-testing-http-api)

### Load simulation
HTTP services should be considered as [Open system](../nbomber/load-simulation). Open systems - it's where you control the arrival rate of users. For Open systems NBomber provides the following load simulations: [Inject](../nbomber/load-simulation#inject), [RampingInject](../nbomber/load-simulation#ramping-inject) and [InjectRandom](../nbomber/load-simulation#inject-random).

### HttpClient
HttpClient should be used carefully since the wrong use of it can cause `socket exhaustion problems`. You can read more about this problem in this article: [You are using HttpClient wrong](https://www.aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/). The basic recommendations are:
- Use a singleton HttpClient (shared instance) per Scenario. 
- Do not create many HttpClient instances. Instead just reuse a single instance per Scenario.
- Disposing HttpClient is not a cheap operation. It can cause `socket exhaustion problems`.

```csharp
// this usage is WRONG
// since HttpClient will be created and disposed for each Scenario iteration

var scenario = Scenario.Create("my scenario", async context =>
{   
    using var httpClient = new HttpClient();
    
    var request = Http.CreateRequest("GET", "https://nbomber.com")
    var response = await Http.Send(httpClient, request);

    ...
});
```

```csharp
// this usage is OK
// since HttpClient will be created once and then reused for each Scenario iteration

using var httpClient = new HttpClient(); 

var scenario = Scenario.Create("my scenario", async context =>
{
    var request = Http.CreateRequest("GET", "https://nbomber.com")
    var response = await Http.Send(httpClient, request);
    
    ...
});
```

<!-- ## test lifecicle
## how to read reports -->
