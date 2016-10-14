﻿Contour
========

![Contour status](https://ci.appveyor.com/api/projects/status/github/sdventures/contour?branch=master&svg=true)
[![NuGet Status](http://img.shields.io/nuget/v/Contour.svg?style=flat)](https://www.nuget.org/packages/Contour/)
[![Nuget](https://img.shields.io/nuget/dt/Contour.svg)](https://nuget.org/packages/Contour)

## About

Contour is the message bus implementation to provide communication .NET services via different transport protocols.
There is support AMQP/RabbitMQ transport now.

## Configuring via xml-file

### Connection

Configuration can be set in .config file, declaring the ‘endpoints’ section inside the ‘serviceBus’ group:

```xml
<sectionGroup name="serviceBus">
    <section name="endpoints" type="Contour.Configurator.EndpointsSection, Contour" />
</sectionGroup>
```

### Endpoints declaration

Service bus works with the endpoints at which messages are sent. For example, if application has two endpoints, then for service bus it is interaction between different participants. And vice versa. If two different applications use the same endpoints, then it is one interaction participant for service bus (when default routing is used)

Endpoints configuration is defined in ‘serviceBus’ section:
```xml
<serviceBus>
    <endpoints>
        <endpoint name="point1" connectionString="amqp://localhost:5672/ ">
            <!-- ... -->
        </endpoint>
        <endpoint name="point2" connectionString="amqp://localhost:5672/ ">
            <!-- ... -->
        </endpoint>
    </endpoints>
</serviceBus>
```

It is possible to define the configuration for multiple endpoints. Each endpoint has mandatory parameters, unique name and connection string to the broker.

Additionally, we can pass component name of [IBusLifecycleHandler](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/IBusLifecycleHandler.cs) type in the attribute ‘lifecycleHandler’, which will be invoked when the state of service bus client is changed. For example, when it starts or stops.

```xml
<serviceBus>
    <endpoints>
        <endpoint name="point1" connectionString="amqp://localhost:5672/" lifecycleHandler="SpecialLifecycleHandler">
            <!-- ... -->
        </endpoint>
    </endpoints>
</serviceBus>
```
It may be helpful, if your component needs to know when service bus was started or stopped.

### Declaration of global message validators

Each incoming message passes through global validators, so they can be optionally defined in ‘validators’ section. Each class of validator must implement [IMessageValidator](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/Validation/IMessageValidator.cs) interface (for more convenience you can inherit from class [FluentPayloadValidatorOf<T>](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/Validation/Fluent/FluentPayloadValidatorOf.cs), which has already implemented this interface).

Also you can set a value of ‘group’ parameter to ‘true’. This would mean that you don’t need to enumerate all validators in component, and just add the name of a validators group, which will include all classes, implementing interface [IMessageValidator](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/Validation/IMessageValidator.cs).

```xml
<endpoint name="point1" connectionString="amqp://localhost:5672/ ">
    <validators>
        <add name="NameValidator" group="true"/>
    </validators>
    <!-- ... -->
</endpoint>
```

### Caching

To control the caching of Request/Reply queries the tag ‘caching’ is used.

You can enable caching by setting attribute to ‘enabled’. At the requested side response is cached by the key, which is generated on the basis of the request body. Caching time is defined by the responder at the ‘Reply’.

‘Publisher/Subscriber’ requests are not cached.

```xml
<endpoint name="point1" connectionString="amqp://localhost:5672/">
    <caching enabled="true"/>
    <!-- ... -->
</endpoint>
```

### Configuring ParallelismLevel

Endpoint provides ability to setup the necessary number of incoming messages handles, each will be run in its own thread. Messages will be distributed between them by the Round-Robin principle.
```xml
<endpoints>
    <endpoint name="Emitter" connectionString="amqp://localhost:5672/" parallelismLevel="4">
    </endpoint>
</endpoints>
```
By default, one handler for incoming messages is used.

### Configuring QoS

Endpoint also provides ability to setup a certain number of incoming messages by one access to the broker.
```xml
<endpoints>
    <endpoint name="Consumer" connectionString="amqp://localhost:5672/">
        <qos prefetchCount="8" />
        <incoming />
    </endpoint>
</endpoints>
```
By default, uses the value of 50 messages, which will be read by one access.

### Sender’s dynamic routing

On the sender’s side you can turn on dynamic routing. It allows you to send messages with label, which wasn’t configured during the endpoint creation.

Dynamic routing can be useful in cases, when you don’t have enough information about all labels at the moment of the endpoint creation.
```xml
<endpoints>
    <endpoint name="point1" connectionString="amqp://localhost:5672/">
        <dynamic outgoing="true" />
    </endpoint>
</endpoints>
```

### Declaration of outgoing messages

All outgoing messages declare in the ‘outgoing’ collection. Each message label declares with an individual tag ‘route’. ‘Key’ and ‘label’ attributes are mandatory.

Key is a label alias and allows you not to mention concrete label of the message. For referring the label from the application, you must specify the alias adding a colon (:) as a prefix.

Additionally, you can set the parameters (as attributes of the ‘route’ element):

 - __persist__ – broker saves the message for reliable delivery (_false_ by default);
 - __confirm__ – broker must confirm message receipt for the processing (message will be stored if persist="true", _false_ by default);
 - __ttl__ – message lifetime (unlimited, if the value is not specified);
 - __timeout__ – response time for requests (default value – 30 seconds).

To support requests, you must declare a subscription point for response messages waiting. At the moment you can declare only default subscription point.
```xml
<outgoing>
    <route key="message" label="message.label" persist="true" ttl="00:01:00" />
    <route key="request" label="request.label" confirm="true" timeout="00:00:50">
        <callbackEndpoint default="true" />
    </route>
</outgoing>
```

### Declaration of incoming messages

All incoming messages are declared in the ‘incoming’ collection. Each message label declares with an individual tag ‘on. ‘Key’ and ‘label’ attributes are mandatory.

Additionally, you can set the parameters (as attributes of the element ‘on’):
 - __requiresAccept__ – event processing must be explicitly confirmed in the handler. Otherwise message will be returned to the queue (false by default);
 - __react__ – message handler name (for example, the name of the handler is registered in the IoC-container);
 - __type__ – CLR-type of the event body. Can be represented by fully qualified name. In this case, default mechanism of type searching is used. Or you can use a short name type. In this case, it will search for all loaded assemblies in the application domain.  If type isb’t mentioned will be using ExpandoObject by default. If type is not specified, ExpandoObject will be used by default.
 - __validate__ – message validator name (class that implements [IMessageValidator](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/Validation/IMessageValidator.cs)), which must operate only within a declared subscription;
 - __lifestyle__ – allows you to specify a handler lifestyle.
Lifestyle possible values:
 - __Normal__ – handler is requested, when you create a bus client (default value);
 - __Lazy__ – handler is requested once, when you get the first message;
 - __Delegated__ – handler is requested every time you get the message, allowing you to manage handler lifetime more flexible.
```xml
<incoming>
    <on key="message" label="message.label" react="MessageHandler" />
    <on key="request" label="request.label" react="RequestHandler" type="RequestPayload" validate="InputValidator" requiresAccept="true" />
    <on key="lazy" label="lazy.label" react="LazyHandler" lifestyle="Lazy" />
</incoming>
```

### Applying configuration

To use configurator, you need to create an instance of the [AppConfigConfigurator](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/Configurator/AppConfigConfigurator.cs) class.

Class constructor can take an object, implementing [IDependencyResolver](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/Configurator/IDependencyResolver.cs) interface, which is uses for getting bus client dependencies. It can be used for searching message handlers or specifying life cycle handler. If concrete client configuration is not required to specify external dependencies, this parameter can be omitted. For simplifying, instead of implementing a particular class, you can use the delegate of [DependencyResolverFunc](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/Configurator/LambdaDependencyResolver.cs) type.

For example, a typical creation of object [AppConfigConfigurator](https://github.com/SDVentures/Contour/blob/master/Sources/Contour/Configurator/AppConfigConfigurator.cs) using Ninject for obtaining handlers looks as follows: 
```csharp
var configurator = new AppConfigConfigurator((name, type) => kernel.Get(type, name));
```

To apply configuration, method Configure is used, which must be invoked during the bus instance creation.
```csharp
var bus = new BusFactory().Create(cfg =>
{
    configurator.Configure("point1", cfg);
});
```

The advantage of this method, is that you can combine configuration via code and configuration file.

If endpoints names are unknown (or should not be known) at compile time, they can be accessed through the _Endpoints_ property. Example of creating and configuring all bus clients:

```csharp
var busFactory = new BusFactory();
var busInstances = configurator
                    .Endpoints
                    .Select(e => busFactory.Create(cfg => configurator.Configure(e, cfg)))
                    .ToList();
```

## Configuring via C# code

Endpoint with all its parameters (except lifestyle) can be configured via code as well.

To use dynamic routing, it is necessary to set message label to the value _MessageLabel.Any_.

```csharp
IBus bus1 = new BusFactory().Create(
    cfg =>
        {
            cfg.UseRabbitMq();
            cfg.SetEndpoint("point1");
            cfg.SetConnectionString("amqp://localhost:5672/ ");
            cfg.HandleLifecycleWith(new SpecialLifecycleHandler());
            cfg.RegisterValidator(new NameValidator());
            cfg.EnableCaching();
            cfg.UseParallelismLevel(4);
            cfg.SetDefaultQoS(8);
            cfg.Route("out.message.label")
                .Persistently()
                .WithTtl(new TimeSpan(0, 1, 0));
            cfg.Route("out.request.label")
                .WithRequestTimeout(new TimeSpan(0, 0, 50))
                .WithConfirmation();
            cfg.On("in.message.label")
                .ReactWith(MessageHandler());
            cfg.On<RequestPayload>("in.request.label")
                .ReactWith(RequestHandler())
                .WhenVerifiedBy(new InputValidator())
                .RequiresAccept();
            cfg.On<RequestPayload>("in.lazy.label")
                .ReactWith(LazyHandler());
        });

IBus bus2 = new BusFactory().Create(
    cfg =>
        {
            cfg.UseRabbitMq();
            cfg.SetEndpoint("point2");
            cfg.SetConnectionString("amqp://localhost:5672/");
        });

```

## Contour headers

Often interaction comprises of several components coming one by one in a chain, and a need to track message passing through this chain appears. For this reason, some incoming messages headers are copied in the outgoing messages.

Header field name | Description | Copying
----------------- | ----------- |--------
x-correlation-id | The correlation identifier is needed to combine a set of messages in one group. For example, it allows you to match reply message with the request | Yes
x-expires | Header, which contains the rules of data deterioration. For example: x-expires: at 2016-04-01T22:00:33Z or x-expires: in 100 | No
x-message-type | Message label with which it was sent. Using this header is not recommended | No
x-persist | Need this message be persistent (saved on disk) or not | No
x-reply-route | Reply message address to the request. For example, x-reply-route: direct:///amq.gen-n9DsUj1qm4vgCq0MHHPoBQ | No
x-timeout | Response timeout to the request | No
x-ttl | Message TTL | No
x-breadcrumbs | List of all endpoints, through which message has passed, separated by semicolon (;) | Yes (adding new value)
x-original-message-id | Message identifier, that started the messages exchange | Yes

## Channels and pipes in Contour

In Contour you can organize message processing as a set of sequential atomic message transformations based on a Pipes and Filters template.

For example, you need to forward messages, that has field ‘Tick’ and its value is odd. Other messages should be filtered. It can be done with the following set of filters:

```csharp
var configurator = new AppConfigConfigurator((name, type) => kernel.Get(type, name));
configuration.On("document.systemtick")
    .ReactWith(new PipelineConsumerOf<ExpandoObject>(
        new IMessageOperator[]
            {
                new JsonPathFilter("Tick"),
                new Filter(request => ((dynamic)request.Payload).Tick % 2 == 1),
                new StaticRouter("document.oddsystemtick")
            }));
```
Most of the filters (messages transformers) are universal and can be used for processing different types of messages.

### Contour filters

Contour comprises the following set of the universal message filters:
 - Acceptor,
 - ContentBasedRouter,
 - Filter,
 - JsonPathFilter,
 - RecipientList,
 - Reply,
 - Splitter,
 - StaticRouter,
 - Translator,
 - TransparentReply,
 - WireTap.
 
### Acceptor

Confirms the message processing and forwards it to the next filter.






## Build the project

 - clone the repository
 - run "build.cmd" to make sure all unit tests are still passing.
 - run "build.cmd RunAllTests" to make sure all integration and unit tests are still passing. In this case you have to configure access to the RabbitMQ broker.

## Library license

The library is available under MIT. For more information see the [License file][1] in the GitHub repository.

 [1]: https://github.com/SDVentures/Contour/blob/master/LICENSE.md
