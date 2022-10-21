---
layout: post
title: AspNetCore Custom Auth
---

The purpose of authentication (authN) is to verify that someone or something is who or what they claim to be. The purpose of authorization (authZ) is to determine whether a user or service has been granted a certain level of access.

There are many ways to authenticate users, including out-of-the-box solutions in Azure and AspNetCore. This article describes how to implement custom authentication and authorization flows. There are many reasons you might want to do this, including:

- You want to validate authentication details using multiple sources (ex. you want to ensure info in the token matches info from a database).

- You want to get authorization details from another service or database.

- You want to assert claims from multiple sources (id_token, Microsoft Graph, databases, etc).

- You want different behaviors for different environments or configurations.

- You want to support multi-tenant authentication scenarios.

- You want to eliminate session state requirements.

- You want to authenticate a service offering comprised of multiple applications.

- You want to control how long the token for access is issued for.

- You want to implement single-page applications to authenticate and renew without implicit grants.

- You want to implement XSS and XSRF protection in a custom way.

It is not always required to implement both a custom authentication and authorization flow, sometimes one or the other is sufficient for your use-case.

## Custom AspNetCore Authentication

Implementating a custom authentication scheme is simple, you simply create a new class that inherits from AuthenticationHandler and write custom logic in the HandleAuthenticateAsync method...

```csharp
public class CustomAuthNHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // if authentication is not performed...
        if (DISABLE_AUTHENTICATION)
        {
            return AuthenticateResult.NoResult();
        }

        try
        {
            // TODO: write custom authentication code

            // if authentication succeeds, provide a ticket which includes a principal and identity with claims...
            var claims = new List<Claim>
            {
                new Claim("sub", "the-user-or-service-account"),
            };
            var identity = new ClaimsIdentity(claims, this.Scheme.Name, "sub", ClaimTypes.Role);
            var principal = new ClaimsPrincipal(identity);
            var ticket = new AuthenticationTicket(principal, scheme);
            return AuthenticateResult.Success(ticket);
        }
        catch (Exception ex)
        {
            // if authentication fails...
            return AuthenticateResult.Fail(ex);
        }
    }
}
```

Then add the implementation to the service collection...

```csharp
services
    .AddAuthentication("customAuthN") // default scheme
    .AddScheme<AuthenticationSchemeOptions, CustomAuthNHandler>("customAuthN", o => new AuthenticationSchemeOptions());
```

As you might guess from the above implementation, you can specify your own custom class for the authentication scheme options as well.

## AspNetCore Authorization Policies

It is not always necessary to implement custom authorization code, sometimes a custom policy will do. You can use policies to validate claims, roles, get external authorization service etc.

- Ref: <https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-6.0>

- Ref: <https://learn.microsoft.com/en-us/aspnet/core/security/authorization/iauthorizationpolicyprovider?view=aspnetcore-6.0>

An example of using a custom policy and handler to access an external authorization service can be found here: <https://github.com/msft-davidlee/ext-auth-service-iauthorizationpolicyprovider>.

## Custom AspNetCore Authorization

To implement a custom authorization scheme, generally you create a custom attribute for the authorization...

```csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
[ExcludeFromCodeCoverage]
public class CustomAuthZAttribute : Attribute
{
    public string Permission { get; set; }
}
```

Then you create some middleware to handle the processing...

```csharp
public class CustomAuthZMiddleware
{
    private readonly RequestDelegate next;

    public CustomAuthZMiddleware(RequestDelegate next)
    {
        this.next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        // check for CustomAuthZAttribute
        var endpoint = context.Features.Get<IEndpointFeature>()?.Endpoint;
        var attribute = endpoint?.Metadata.GetMetadata<CustomAuthZAttribute>();

        // check authentication if it was requested via presence of CustomAuthZAttribute
        var hasAttribute = attribute != null;
        if (hasAttribute && !DISABLE_AUTHENTICATION)
        {
            var isAuthenticated = context?.User?.Identity?.IsAuthenticated ?? false;
            if (!isAuthenticated)
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsync("Unauthorized");
                await context.Response.CompleteAsync();
                return;
            }
        }

        // implement any custom code for authorization
        try
        {
            // TODO: write authorization code
        }
        catch (Exception ex)
        {
            // TODO: log ex
            context.Response.StatusCode = 403;
            await context.Response.WriteAsync("Forbidden");
            await context.Response.CompleteAsync();
            return;
        }

        // next
        await this.next(context);
    }
}
```

Next, the middleware has to be registered when you configure the host (remember that middleware is processed in order)...

```csharp
app.UseMiddleware<CustomAuthZMiddleware>();
```

Finally, you may decorate a controller or operation with your custom header...

```csharp
[ApiController]
[Route("api")]
public class MyController : Controller
{
    [HttpGet("do-operation")]
    [CustomAuthZ(Permission = "my-custom-permission")]
    public ActionResult DoOperation()
    {
        // TODO: implement the operation
    }
}
```

## How to make this work with Azure Functions

To make this solution work with Azure Functions, you need to encapsulate your custom authentication and authorization code into a separate service that can be dependency injected. You might create a service that adheres to an interface like this...

```csharp
public interface ICustomAuthProvider
{
    Task<AuthenticateResult> AuthenticateAsync(HttpRequest req);

    Task VerifyPermission(string policy);
}
```

You might then use that provider in an Azure Function endpoint...

```csharp
public class MyHttpFunction
{
    private readonly ICustomAuthProvider authProvider;

    public MyHttpFunction(ICustomAuthProvider authProvider)
    {
        this.authProvider = authProvider;
    }

    [FunctionName("do-operation")]
    public async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)] HttpRequest req,
        ILogger logger)
    {
        // authenticate and authorize
        var authNResult = await this.authProvider.AuthenticateAsync(request);
        if (authNResult.Succeeded)
        {
            request.HttpContext.User = authNResult.Principal;
        }
        try
        {
            await this.authProvider.VerifyPermission("my-custom-permission");
        }
        catch (Exception ex)
        {
            return new ForbidResult();
        }
    }
}
```

Of course, that code could be extracted into an extension method for re-use. If you are using the Azure Function Isolated Process Model, you can even implement this via middleware (similar to the above design): <https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide#differences-with-net-class-library-functions>.

If you want this same solution to work in aspnetcore and Azure Functions, then you can simply have this custom service injected into your CustomAuthNHandler constructor and CustomAuthZMiddleware constructor.
