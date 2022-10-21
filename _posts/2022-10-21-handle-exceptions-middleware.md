---
layout: post
title: Handle Exceptions Middleware
---

There are many places in an aspnetcore solution that could throw an exception that might not be the appropriate exception to pass on to callers. In addition, there might be cases were we need to mutate the response that would have gone to users in middleware before those responses are sent. For example:

1. sending a specific status code to a caller when an exception is raised
2. ensure that exceptions are safe to pass to the caller (ie. don't contain sensitive information, don't have stack traces, etc.)
3. an expection was set at the beginning of middleware that was not fulfilled by the end
4. ensure all exceptions meet the RFC7807 problem details requirement
5. translate the output to a different language

Generally exceptions are thrown before content is written to the response stream, in which case, middleware can respond to those exceptions without any special plumbing. However, in cases where an exception might be thrown __after__ a response has been sent to the stream, that response cannot be changed (and may even have already been delivered to the client). This pattern will address this through the use of a memory buffer.

This pattern discusses a way to address those requirements by using a middleware component at the top of the middleware stack to buffer and control responses.

## Impact

Dispatching content to the client as it is available via the use of the response stream is desireable because it reduces latency and memory. This pattern negates those advantanges, so you generally only want to use it for small payloads. Registering the middleware with UseWhen will allow you to apply it when appropriate.

## HttpStatusException

There are many articles online debating the use of results objects versus exceptions to respond to clients with appropriate status. To be clear, where possible, you should generally use results objects (such as those implmementing IActionResult), however, that is not always possible or practical. In those cases, consider creating an exception that can carry a status code all the way back to the middleware.

```csharp
public class HttpStatusException : Exception
{
    public HttpStatusException(HttpStatusCode statusCode, string message, Exception innerException = null)
    : base(message, innerException)
    {
        this.StatusCode = (int)statusCode;
    }

    public HttpStatusException(int statusCode, string message, Exception innerException = null)
    : base(message, innerException)
    {
        this.StatusCode = statusCode;
    }

    public int StatusCode { get; }
}
```

## HandleExceptionsMiddleware

It is important that this be registered as soon as possible as it will only buffer and handle exceptions for middleware and controller operations encapsulated by it.

Notice that this code uses a MemoryStream to buffer the response instead of allowing that to be passed directly to the caller.

```csharp
public class HandleExceptionsMiddleware
{
    private readonly RequestDelegate next;
    private readonly ILogger<HandleExceptionsMiddleware> logger;
    private readonly IHandleExceptionsMiddlewareConfig config;

    public HandleExceptionsMiddleware(
        RequestDelegate next,
        ILogger<HandleExceptionsMiddleware> logger)
    {
        this.next = next;
        this.logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        var exist = context.Response.Body;
        using var buffer = new MemoryStream();
        context.Response.Body = buffer;
        try
        {
            await this.next(context);
            buffer.Seek(0, SeekOrigin.Begin);
            await buffer.CopyToAsync(exist);
            context.Response.Body = exist;
        }
        catch (HttpStatusException ex) // addresses scenario #1 above
        {
            context.Response.Body = exist;
            context.Response.Clear();
            this.logger.LogError(ex, "the following exception was raised by a controller...");
            context.Response.StatusCode = ex.StatusCode;
            context.Response.Headers["Content-Type"] = "text/plain";
            await context.Response.WriteAsync(ex.Message);
            await context.Response.CompleteAsync();
        }
        catch (Exception ex) // addresses scenario #2 above
        {
            context.Response.Body = exist;
            context.Response.Clear();
            this.logger.LogError(ex, "the following exception was raised by a controller...");
            context.Response.StatusCode = 500;
            context.Response.Headers["Content-Type"] = "text/plain";
            await context.Response.WriteAsync("Internal Server Error"); // a safe message to send to the caller
            await context.Response.CompleteAsync();
        }
        // TODO: this code could be modified to address other scenarios
    }
}
```
