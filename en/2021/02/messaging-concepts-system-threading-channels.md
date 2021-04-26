---
title: 'Messaging: Point-to-point with System.Threading.Channels'
draft: true
description: 'In this post, we will showcase an abstraction of point-to-point messaging channels using the System.Threading.Channels library.'
tags: ['distributedsystems','architecture','messaging','dotnet']
cover_image:
# date: 2021-04-28
# canonical_url: 'https://mateuscviegas.net/2021/02/messaging-concepts-system-threading-channels'
---

## *tl/dr*

This post is the second one of a series about basic concepts within distributed architectures.

1) [Messaging: An introduction](https://mateuscviegas.net/2021/02/messaging-an-introduction)
2) [Messaging: Point-to-point with System.Threading.Channels](https://mateuscviegas.net/2021/02/messaging-concepts-system-threading-channels)
3) Messaging: Idempotence and Delivery Types
4) Messaging: Consistency and Resiliency Patterns

The goal of this series is to provide new venturers on distributed systems world with basic concepts of messaging, the tradeoffs of this kind of resource and how to provide solutions for some of these tradeoffs in order to achieve the most important benefit of messaging: loose coupling and high scalability within a distributed architecture.

In this post, we will showcase more in depth the concept of **point-to-point channels** using the System.Threading.Channels library, as well as introduce a very common caveat within messaging architectures. If you are not familiar with the mentioned library yet, I have [a post explaining the purpose and APIs in this library more in depth](https://mateuscviegas.net/2020/06/intro-system-threading-channels/).

> Disclaimer: even-though these concepts are language-agnostic, the examples over this post will be showcased with C# and .NET 5.

## Deffering requests, but why?

The use case we are going to showcase here it is one pretty common within the messaging world: an web API receives an incoming request that needs to be processed in background in an asynchronous manner, in order to provide API clients a fast response time. In this scenario, these sort of web APIs would:

* Receive a request;
* Forward this request as a message to a message channel;
* Return a success status code indicating that the message was received and will be processed later: 202 (accepted).

Finally, this message would then be transported over this channel to be processed by a *consumer*, a background process/application, which reads this message from the channel and processes it. Since there's no execution upon the incoming request besides forwarding this as a message, the web API will not depend on any factor that might increase its response latency rather than the message publishing, thus shortening this latency. Imagine a scenario where this latency might be increased during a request execution due to one of the following factors:

* Request payload size;
* Geographical distance from the infrastructure;
* Access to IO-bounded resources subject to delayed responses, such as overloaded databases and another APIs in external networks;
* Request execution happens in a long-running process, like a batch process, for instance.

If we need to decouple our API response times from the above situations, than messaging and a fire-and-forget asynchronous execution come both in hand, since, as we've seen, there will be only one dependency that will affect the response latency: the message publishing.

## Point-to-point channels

As seen on the introductory post, these type of channels imply that a given message published on a given channel will be received by one and only given consumer. We can emulate point-to-point channels in two manners: in-process, with a asynchronous handling delegated to consumers on a dettached CPU thread, and out-of-process, with the async handling delegated via an external infrastructure to consumers connected to it.

One limitation of using point-to-point channels approach is if a response which depends on the request execution output is required to to be returned to the client. Since the execution is deferred as asynchronous, this kind of responses cannot be given by the application. We are limited to the information available at the time of request.

### In-process

For this category, each language will have its own options. For C#, [Hangfire queue-based processing](https://www.hangfire.io/features.html) is an example of in-process point-to-point channels, as it uses threads with different execution context to dettach a producer from a consumer. As we'll see on the example later on, the [System.Threading.Channels library]() can also be used in a similar way.
### Out-of-process

Examples for out-of-process point-to-point channels, i.e. with an extra layer of infrastructure, are [Azure Service Bus Queues](https://docs.microsoft.com/en-gb/azure/service-bus-messaging/service-bus-queues-topics-subscriptions#queues) or a combination of [RabbitMQ Direct Exchange + Queue](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html). There's a lot of tutorial over these two on the internet, if you might want to look out.

For out-of-process messaging we introduce a **single point of failure**: the messaging infrastructure. A single point of failure, commonly named SPOF, it is a component within a system that, if fails, will stop the entire system of working.

One more bottleneck might be debugging: how do we track errors/tracings accross a transaction distributed over one or more several components within a system?

> It only makes sense to decouple a request execution from its caller via message, when there's an *actual need* justifying the extra complexity, such the ones mentioned on the previous section. Do not fall into trap of overengineering applications to messaging architectures without having to: you will find yourself spending more time to trace errors, adding extra costs for a high-available messaging infrastructure that will have to be maintained either by a specialized team or a cloud provider, adding operational complexity by having to deploy and monitor multiple applications instead of one and needing to increase the expertise of the development team in order to deal with all of these issues.

## Practical Example

Consider a simple web API written in C# and .NET. This API will have one single endpoint to receive a request and will forward this request as a message through a channel, acting as a *producer*. The channel will deliver the message to a connected *consumer* that, finally, will handle this message on the background as a [BackgroundService](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-5.0&tabs=visual-studio).

### Message Helpers

Consider that in this example we have a `MessageHeaders` class that contains the description of basic *headers* that will be sent with our messages:
* `MessageId`: with an unique identifier of the sent message, applications are not only able to identify a message, but also to use this id to guarantee they process each message only once, allowing *idempotence* to be introduced. Idempotence is a math concept that means that an operation which was executed multiple times has no other result than the one achieved from the first execution.
* `CorrelationId`: that will send actually an identifier of the request where the message was originated. These kind identifiers ids particularly useful in a scenario we briefly described within the trade-offs section earlier on: in distributed applications, it is hard to debug the end-to-end operation and identify where things might go wrong. By sharing a correlation id on messages, we can rely to logging and tracing mechanisms to match multiple messages that belong to a single operation. This approach is pretty common when we want to apply a technique called *distributed tracing*, that basically allows us to monitor a distributed operation that is executed across several components of a distributed system. But this is a subject for another more advanced series.

```csharp
 public class MessageHeaders
{
        public const string MessageId = "X-MESSAGE-ID";
        public const string CorrelationId = "X-CORRELATION-ID";
}
```

We can also create a generic class called `Message<T>`, that will contain both message headers and body:

```csharp
public class Message<T>
{
    public Dictionary<string, string> Headers { get; set; } = new Dictionary<string, string>
    {
        { MessageHeaders.CorrelationId, Activity.Current.Id },
        { MessageHeaders.MessageId, Guid.NewGuid().ToString() }
    };
    public T Body { get; set; }
}
```

On the above code we make use of the [System.Diagnostics.Activity](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.activity?view=net-5.0) class to retrieve metadata from the current operation beeing executed. Here we set our message the correlation id as the `Activity.Current.Id` so that we have the capability of tracking the operation where the message originated. Also, we create a new unique `Guid` to be used as the message id.

### Producer

The message producer will be responsible for sending the request over a Channel. Here we are using the `System.Threading.Channels.Channel` described in the beginning of this arcticle, but for our example you can also understand the Channel simply as a messaging channel abstraction. Our producer has basically a `Send` method that dispatches this messages through an unbounded channel and logs the timestamp of this operation.

```csharp
public class Producer
{
    private readonly ILogger<Producer> _logger;
    private readonly Channel<object> _channel;

    public Producer(
        ILogger<Producer> logger,
        Channel<object> channel)
    {
        _logger = logger;
        _channel = channel;
    }

    public void Send<T>(Message<T> message)
    {
        if (_channel.Writer.TryWrite(message))
        {
            _logger.LogDebug($"Message {message.Headers[MessageHeaders.orrelationId]} sent at {DateTime.Now:f}");
        }
        else
        {
            throw new Exception("Error while sending the message");
        }
    }
}
```


### Consumer

The consumer, by its turn, is connected to this unbounded channel and it is a BackgroundService that remains active while this channel is not completed.

> All of these concepts about unbounded channels and channel completion are on my [post about System.Threading.Channels](https://mateuscviegas.net/2020/06/intro-system-threading-channels/)).

```csharp
public class Consumer : BackgroundService
{
    private readonly ILogger<Consumer> _logger;
    private readonly Channel<object> _channel;

    public Consumer(ILogger<Consumer> logger, Channel<object> channel)
    {
        _logger = logger;
        _channel = channel;
    }

    protected override async Task ExecuteAsync(CancellationToken toppingToken)
    {
        _logger.LogInformation("Consumer started");
        
        while (await _channel.Reader.WaitToReadAsync(stoppingToken))
        {
            if (_channel.Reader.TryRead(out object message))
            {
                if (message is Message<string> typedMessage)
                {
                    await Task.Delay(500);

                    _logger.LogDebug($"Message {typedMessage.HeadersMessageHeaders.CorrelationId]} received at DateTime.Now:f}");
                }
            }
        }
        _logger.LogInformation("Consumer stopped");
    }
}
```


### Program.cs

Finally, putting it all together we will write our API endpoint in a Program.cs file using [top-level statements](https://docs.microsoft.com/en-us/dotnet/csharp/tutorials/exploration/top-level-statements). This endpoint will receive an web API request, and send a message through the `Producer` returning a success status code right afterwards. Here, we will rely on dependency injection and register the Channel, Producer and Consumer as dependencies that will be handled by the DI container and injected via constructors wherever we might need them.

```csharp
CreateHostBuilder(args).Build().Run();

static IHostBuilder CreateHostBuilder(string[] args) =>
    Host
    .CreateDefaultBuilder(args)
    .ConfigureWebHostDefaults(webBuilder => webBuilder
        .ConfigureServices((_, services) => SetupProducerAndConsumer(services))
        .Configure((_, app) =>
        {
            app.UseRouting();
            app.UseEndpoints(route => MapDefaultRoute(app, route));
        }));

static void MapDefaultRoute(IApplicationBuilder app, IEndpointRouteBuilder route)
{
    route.MapGet("/", async context =>
    {
        var producer = app.ApplicationServices.GetRequiredService<Producer>();
        producer.Send(new Message<string>() { Body = "OK" });
        await context.Response.WriteAsync("Hello world");
    });
}

static void SetupProducerAndConsumer(IServiceCollection services)
{
    services.AddSingleton(_ => Channel.CreateUnbounded<object>());
    services.AddSingleton<Producer>();
}
```

### Running the app and observing 

Before running the app, one last change needs to be made: the `appsettings.Development.json` must have the `Logging:LogLevel:Default` entry set with `Debug`. After that, we can run the application and call the endpoint written in the `Program.cs` file and see a similar output on the console:

```
dbug: MessagingChannels.Example.Producer[0]
      Message 00-e2bd4c5d962d1541bcf8208e468a48b6-0241e149ea6ef447-00 sent at 05:15:35.373

dbug: MessagingChannels.Example.Consumer[0]
      Message 00-e2bd4c5d962d1541bcf8208e468a48b6-0241e149ea6ef447-00 received at 05:15:35.878
```

This simple example is a very simple demonstration of a message transported over a channel to a consumer. The channel in this example, acts as a point-to-point channel we described in the [introductory post of this series]().

Here you could check that the `CorrelationId` message header we mentioned earlier is also displayed in the logs, allowing us to correlate the two operations: the web API request and the background service execution. The timestamp present on the outputs also made clear that, even though our consumption is delayed, the web API was able to respond quickly to the incoming request. Here we introduced a fake delay, but imagine that this delay happens due to one of the reasons previously described. Finally, we achieved the main advantage of fire-and-forget async web APIs: fast response times due to deferred and decoupled request execution.

## Wrap-up

This post was the second one in our series about basic messaging concepts. Here we showcased the decoupling of requests using a fire-and-forget approach by delegating their execution with a message transported through a channel to a consumer, using the in-mem abstractions from the System.Threading.Channels library.

To know how to work with messaging is important, but even more important is to know why and when, since it introduces more points of failure and more complexity for development, maintenance and monitoring. It also limits the kind of responses that a web API can return, since the request execution is deferred for a decoupled consumer, that might be another application.

Stay tuned, on the next post of this series we will talk a little bit more about a common and tricky scenario from point-to-point channels and its relation to three different message delivery types.

You can find the entire source code of this example in this [GitHub repository](https://github.com/mviegas/messaging-channels-example).
