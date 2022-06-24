---
layout: post
title: FromBody Formatters for AspNetCore
---

The [FromBody] property decorator for aspnetcore allows you to deserialize JSON bodies and other types of messages. However, this is extensible to support other types of bodies, including this example of a plain text deserializer...

```c#
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc.Formatters;

/// <summary>
/// This formatter allows [FromBody] to accept text/plain content.
/// </summary>
public class TextPlainInputFormatter : InputFormatter
{
    public TextPlainInputFormatter()
    {
        this.SupportedMediaTypes.Add("text/plain");
    }

    public override async Task<InputFormatterResult> ReadRequestBodyAsync(InputFormatterContext context)
    {
        var request = context.HttpContext.Request;
        using (var reader = new StreamReader(request.Body))
        {
            var content = await reader.ReadToEndAsync();
            return await InputFormatterResult.SuccessAsync(content);
        }
    }

    public override bool CanRead(InputFormatterContext context)
    {
        return context.HttpContext.Request.ContentType.StartsWith("text/plain");
    }
}
```

To register a formatter...

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers(x => x.InputFormatters.Add(new TextPlainInputFormatter()));
}
```
