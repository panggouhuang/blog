# Execution context

## HttpContextAccessor
We start by exploring the execution context of an ASP.NET Core web service. If it’s your first time implementing a web service with ASP.NET Core and using the built‑in `HttpContextAccessor` service, you might be curious about how it works. This is because `HttpContextAccessor` is registered as a singleton service, but `HttpContext` is created per request. So how can we use a singleton service to access per‑request data?

Let’s take a look at the code of `HttpContextAccessor`:

```csharp
// Licensed to the .NET Foundation under one or more agreements.
// The .NET Foundation licenses this file to you under the MIT license.

using System.Diagnostics;

namespace Microsoft.AspNetCore.Http;

/// <summary>
/// Provides an implementation of <see cref="IHttpContextAccessor" /> based on the current execution context.
/// </summary>
[DebuggerDisplay("HttpContext = {HttpContext}")]
public class HttpContextAccessor : IHttpContextAccessor
{
    private static readonly AsyncLocal<HttpContextHolder> _httpContextCurrent = new AsyncLocal<HttpContextHolder>();

    /// <inheritdoc/>
    public HttpContext? HttpContext
    {
        get => _httpContextCurrent.Value?.Context;
        set
        {
            // Clear the current HttpContext trapped in the AsyncLocal, as it’s done.
            _httpContextCurrent.Value?.Context = null;

            if (value != null)
            {
                // Use an object indirection to hold the HttpContext in the AsyncLocal,
                // so it can be cleared in all ExecutionContexts when it’s cleared.
                _httpContextCurrent.Value = new HttpContextHolder { Context = value };
            }
        }
    }

    private sealed class HttpContextHolder
    {
        public HttpContext? Context;
    }
}
```

If we ignore `HttpContextHolder` (which is essentially a wrapper for `HttpContext`), we see that `HttpContextAccessor` uses a static `AsyncLocal<HttpContextHolder>` to store the current `HttpContext`. It may seem odd to use a static variable to store a per‑request value.

Before talking about `AsyncLocal`, let’s first understand **async**.

## Asynchronous programming
In my opinion, asynchronous programming is event‑based concurrency. [Concurrency is when two or more tasks can start, run, and complete in overlapping time periods, whereas parallelism is when tasks literally run at the same time.](https://stackoverflow.com/questions/1050222/what-is-the-difference-between-concurrency-and-parallelism) Event‑based means that instead of blocking to wait for an operation to complete, we treat the operation as an event and register a callback to be executed when it finishes.

### Event loop
The approach is straightforward: register a callback for each event, and when the event occurs, invoke its callback. Your thread essentially waits for events and then executes their callbacks. Within a callback, besides handling the event, you can trigger another event:

```javascript
while (true) {
    const event = getNextEvent();
    event.callback();
}
```

Assuming `getNextEvent()` and `event` are already implemented, the most interesting challenge is capturing context in the callback. Normally, a callback isn’t just a static function—it also needs access to the environment in which it was created. Consider this C# example:

```csharp
class Program
{
    public Task ProcessAsyncWithContinue(int a)
    {
        return DoSomethingAsync(a).ContinueWith(t =>
        {
            Console.WriteLine($"Result: {t.Result}");
            Console.WriteLine(a);
        });
    }

    public async Task<int> DoSomethingAsync(int a)
    {
        await Task.Delay(500);
        return 100;
    }
}
```

The continuation after `DoSomethingAsync` consists of those two `Console.WriteLine` calls, but when it executes, it also needs the value of `a`, which is captured by the lambda’s closure. In C#, lambdas capture their surrounding environment, allowing you to access those variables later. Here, `a` is captured and remains available when the continuation runs. The `async`/`await` keywords are syntactic sugar for this continuation mechanism, making asynchronous code more readable. The following code is functionally equivalent but clearer when chaining continuations:

```csharp
public async Task ProcessAsync(int a)
{
    var result = await DoSomethingAsync(a);
    Console.WriteLine($"Result: {result}");
    Console.WriteLine(a);
}
```

## AsyncLocal
Before explaining how AsyncLocal works, let’s discuss why it’s useful. As mentioned, lambdas capture their creation environment. If you need something (like HttpContext) accessible in a continuation, you must ensure the variable is in scope in the caller. Without HttpContextAccessor, you would have to pass HttpContext as a parameter to every method that needs it, so it could be captured by the lambda. This would clutter every method signature with an extra HttpContext parameter.

This is where AsyncLocal comes in. An ambient context allows you to provide contextual information without explicitly passing it through method parameters. It lets you access certain data (such as HttpContext) from anywhere in the execution flow without having to pass it around.

In a synchronous world, you could store it in a thread‑local variable, since operations execute one by one. If the previous execution hasn’t finished, the next one can’t start, so you can safely set and clean up.

However, in an asynchronous world, execution can be paused and resumed later. We still need to access the same data in the continuation as we had in the creation context, and we need to support multiple executions running concurrently.

The core implementation of AsyncLocal—which underlies the .NET ExecutionContext—is quite simple. We just need to mimic what lambdas do with their closures: capture the context in the creation environment and then make it available in the continuation. Let’s assume we're implementing this for Node.js to assume there is only one thread.
```typescript
class ExecutionContext {
    public static current: ExecutionContext = new ExecutionContext();

    public value: number;
    public static Capture(): ExecutionContext {
        return new ExecutionContext(ExecutionContext.current);
    }

    public constructor(context?: ExecutionContext)
    {
       this.value = 0;
       if(context) {
         this.value = context.value;
       }
    }

    public static Run(context: ExecutionContext, cb: () => void): void {
        var previousContext = ExecutionContext.Capture();
        if(previousContext == context) {
            cb();
            return;
        }
        ExecutionContext.current = context;
        cb();
        ExecutionContext.current = previousContext;
    }
}


async function timeout() {
    await new Promise(resolve => {
        setTimeout(resolve, Math.random() * 100);
    })
}

async function runWithContext(value: number) {
    var ec = ExecutionContext.Capture();
    ec.value = value;
    await timeout();
    ExecutionContext.Run(ec, () => {
        console.log(ExecutionContext.current.value);
    });
}
(async () => {
   await Promise.all([runWithContext(1), runWithContext(2), runWithContext(3)]);
})();
```
