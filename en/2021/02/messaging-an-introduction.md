---
title: 'Messaging: an Introduction'
published: true
description: 'Messaging is one of the pillars for distributed architectures to be sustainable. Here we will showcase basic theoretical concepts such as Messages and their types, as well as Transport Channels and their types.'
tags: ['distributedsystems','architecture','messaging','dotnet']
cover_image:
date: 2021-02-21
canonical_url: 'https://mateuscviegas.net/2021/02/messaging-an-introduction'
---

# *tl/dr*

This post is the first one of a series about basic concepts within distributed architectures. 

1) Messaging: An introduction
2) Messaging: Practical basic concepts with System.Threading.Channels
3) Messaging: Interaction Styles
4) Messaging: Consistency and Resiliency Patterns

The goal of this series is to provide new venturers on distributed systems world with basic concepts of messaging, the tradeoffs of this kind of resource and how to provide solutions for some of these tradeoffs in order to achieve the most important benefit of messaging: loose coupling and high scalability within a distributed architecture.

Messaging is one of the pillars for this kind of architecture to be sustainable. Here we will showcase basic theoretical concepts such as Messages and their types, as well as Transport Channels and their types. Also, we will give a high level overview about when you should or shouldn't consider messaging.

# Loosely coupling, but why?

First things first: we can define messaging as a way of communication between two or more resources. In this manner, these two resources communicate by sending and receiving *messages* through a *messaging infrastructure* instead of doing direct and synchronous requests to each other.

This is achieved through the exchange of *messages* through a *channel*: a sender (application/component) writes messages to a channel inside this messaging infrastructure, and a receiver reads messages from it. As a result, when introduced, messaging removes a direct coupling between two components within a system that, as a result, turns out to be called "loosely coupled".

As pretty as it might seems, one question remains though: why would we want to make one or more components or applications within a system loosely coupled? There are several reasonable reasons here, but first one let's one major type of coupling that motivates the use of messaging.

## Temporal coupling

A system usually has some IO-bounded dependencies, like databases, external APIs, email sender services, blob storages, or even another systems, between many more. Each time a request cross an IO boundary and has to access either disk or network, we are dependent on the response of the request in order the calling thread continue its execution. 

Even with modern concepts such as Task-based programming in C#, the called "asynchronous" code is, de facto, async only from a thread perspective. This means, talking here in a really rough manner, that applications like an web API, will still have the incoming requests waiting for each one of these async blocks of code to complete in order respond the client. This asynchronicity here relies only on the not locking CPU resources while IO operations are in place, since there's a limited number of threads to execute CPU-bounded operations on the thread pool. Imagine the following situation:
* There's an incoming request;
* This request does a query to the database and return a response with the result;
    * Since database queries are IO operations, the calling thread might be blocked while awaits for the database to execute the query. If the database is, by any means, overlodaded, it might take a while. In this case, imagine that there are multiple requests coming in. Since the number of threads is limited, there might come a time when there's no more free threads to handle incoming requests: starvation happens.
    * Now, imagine that these incoming requests access the database via an `async` method that executes the query. When that happens, a state machine is created with the request context and the control of this state machine is passed to a new IO-bounded thread, so the CPU-bounded calling thread gets free to be used by any other incoming request. This IO-bounded thread executes its operation on the database and, since it has this state machine, it might have a *continuation* through a *synchronization context* indicating what to do next, a.k.a., how to provide a response to the API request with the query result. In this case, we give some buffer for our application performace, allowing our thread pool to relies only on CPU-bounded work.

That whole explanation was to clarify that this execution is, from the **temporal** perspective, still synchronous. The overall response time still depends on the database execution. Or, perhaps, on another external and unstable web API. And that's why this type of coupling is called *temporal coupling* and might be an issue if you need an application to be responsive.

Messaging allows us to remove temporal coupling by deferring requests to be executed either in background or by third-party applications. In this case, a system component sends a message containing some sort of data through a channel and another system component, which is connected to this channel, receives this message and execute. Therefore, the component that sends the message is temporal coupled only to the message dispatching and not to an entire ecosystem of dependencies that there might exist.

# Message Types

Messages are chunks of data structured with both a *body* and also some *headers*. And even though the body contains the central information in a message, the headers might have some really useful metadata such as an *unique message identifier*, one or more *correlation identifiers* that could be used to correlate operations on different systems, etc. But what's important to understand is: messages are literally DTOs (data transfer objects). These DTOs are then sent serialized and sent via a channel. Either in JSON, binary, XML or any other format.

It is common to think about messages as their meanings. It allows them to be categorised into types. Some of the most popular message types are *commands* and *events*. 

## Commands

Commands can be understood as requests specifications. A descriptive method name with zero or more parameters. Imagine, for instance, a `CreateUserCommand`. This command might have some parameters such as `Email` and `Password`. We could describe this message as the following class:

```csharp
public class CreateUserCommand
{
   public string Email { get; set; }
   public string Password { get; set; }
}
```

When this class is sent over a channel, the recipient system would then read this specification and execute the operation. The meaning of a command is basically *"do something with these attributes"*, in this case, this message is telling to the recipient: *"create a user with this email and this password"*.

## Events

On the other hand, we have *events*, that indicates and describe something that happened within the system. So for a `CreateUserCommand`, we could have a `UserCreatedEvent` describing a side-effect on our system, result from a previous operation. In this particular case, the event would be pretty similar to the command.

```csharp
public class UserCreatedEvent
{
   public string Email { get; set; }
}
```

You could read this as "an user was created with this email". This information could then be broadcasted via messaging, so that any other interested part of the system could receive this information and do something about it, like sending an welcome email, for instance.

# Channels

The [Enterprise Integration Patterns book](https://www.amazon.com/o/asin/0321200683/ref=nosim/enterpriseint-20) has the following description about channels: *"when an application has information to communicate, it doesn't just fling the information into the messaging system, it adds the information to a particular Message Channel. An application receiving information doesn't just pick it up at random from the messaging system; it retrieves the information from a particular Message Channel."*

![](https://www.enterpriseintegrationpatterns.com/img/MessageChannelSolution.gif)

There are mainly two types of message channels:

* Point-to-point: this type of channel delivers the message to exactly one, and only one, consumer that is listening to the channel. If there are multiple consumers, it means that they will concur for the incoming messages and each message will only be read by one consumer. To have multiple and competing consumers in a point-to-point channel could be useful to increase the throughput of the channel, if we have a lot of messages being sent to it. It is basically a way of scaling the message comsuption. Commands are often sent through this type of channel.

* Publish-subscribe: this type of channel might have multiple consumers that receive, each one, a copy of the message. Events are often sent through this type of channel. This type of channel is particularly good for some scenarios. With *pub/sub* we can easily move information around many interested parties with no temporal coupling between them. One system X can know what happens to other system Y by receiving messages sent by Y to a message channel that X, by its turn, is listening to. That way, we can easily move information around systems with no tight coupling between them, making the integration process smoother. 

# First trade-offs

Even though messaging is often referred as a way of integrating and remove coupling between distributed systems, we can use the same concept to extend functionality inside one single system. By using the concept of commands and events within an application, we can easily expand its features without changing existing ones. I talk a little bit more about it in an old post about the correlation between ["the open-closed principle and event-driven architectures"](https://dev.to/mviegas/a-brief-brainstorm-about-the-open-closed-principle-and-event-driven-architectures-323h).

However, just as anything regarding to software engineering and architecture, messaging has its trade-offs. By delegating the message handling to an external consumer via a message channel we introduce a **single point of failure**: the messaging infrastructure. A single point of failure, commonly named SPOF, it is a component within a system that, if fails, will stop the entire system of working. 

Therefore, since all the communication between the systems rely on a central piece of communication, the messaging infrastructure, it becomes a requirement that this piece has a very high level of availability. 

There still many other trade-offs of applying messaging and distributing your system, such as maintanence, monitoring and development complexity introduced by many side-effects caused by the decoupling, and we will talk more in detail about them on the next posts of this series. 

# Wrap-up

Messages are a nice concept and are very fundamental when we talk about distributed systems and architectures. The concept itself is an abstraction, which implementations are diverse, as well as the possible infrastructure resources to handle it. 

In this introductory post, we showcased how we exchange temporal for loose coupling with messaging, and why it my good. We introduced basic message types, channel types and also discussed briefly about the consequences of introducing messaging to a system.

On the next post will showcase an abstraction of the concepts showcased in this introduction: removing temporal coupling in a web API sendi messages via channels created with C# and the System.Threading.Channels library.