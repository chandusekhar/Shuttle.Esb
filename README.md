# Shuttle.Esb

A highly flexible and free .NET open-source enterprise service bus.

# Documentation 

There is [extensive documentation](http://shuttle.github.io/shuttle-esb/) on our site and you can make use of the [samples](https://github.com/Shuttle/Shuttle.Esb.Samples) to get you going.

# Overview

Start a new **Console Application** project and select a Shuttle.Esb queue implementation from the [supported queues]({{ site.baseurl }}/packages/#queues):

<div class="nuget-badge">
	<p>
		<code>Install-Package Shuttle.Esb.Msmq</code>
	</p>
</div>

Now we'll need select one of the [supported containers](http://shuttle.github.io/shuttle-core/overview-container/#Supported):

<div class="nuget-badge">
	<p>
		<code>Install-Package Shuttle.Core.Autofac</code>
	</p>
</div>

We'll also need to host our endpoint using the [service host](http://shuttle.github.io/shuttle-core/overview-service-host/):

<div class="nuget-badge">
	<p>
		<code>Install-Package Shuttle.Core.ServiceHost</code>
	</p>
</div>

Next we'll implement our endpoint in order to start listening on our queue:

``` c#
internal class Program
{
	private static void Main()
	{
		ServiceHost.Run<Host>();
	}
}

public class Host : IServiceHost
{
	private IServiceBus _bus;

	public void Start()
	{
		var containerBuilder = new ContainerBuilder();
		var registry = new AutofacComponentRegistry(containerBuilder);

		ServiceBus.Register(registry);

		var resolver = new AutofacComponentResolver(containerBuilder.Build());

		_bus = ServiceBus.Create(resolver).Start();
	}

	public void Stop()
	{
		_bus.Dispose();
	}
}
```

A bit of configuration is going to be needed to help things along:

``` xml
<configuration>
	<configSections>
		<section name="serviceBus" type="Shuttle.Esb.ServiceBusSection, Shuttle.Esb"/>
	</configSections>

	<serviceBus>
		<inbox 
			workQueueUri="msmq://./shuttle-server-work" 
			deferredQueueUri="msmq://./shuttle-server-deferred" 
			errorQueueUri="msmq://./shuttle-error" />
	</serviceBus>
</configuration>
```

### Send a command message for processing

``` c#
using (var bus = ServiceBus.Create(resolver).Start())
{
	bus.Send(new RegisterMemberCommand
	{
		UserName = "Mr Resistor",
		EMailAddress = "ohm@resistor.domain"
	});
}
```

### Publish an event message when something interesting happens

``` c#
using (var bus = ServiceBus.Create(resolver).Start())
{
	bus.Publish(new MemberRegisteredEvent
	{
		UserName = "Mr Resistor"
	});
}
```

### Subscribe to those interesting events

``` c#
resolver.Resolve<ISubscriptionManager>().Subscribe<MemberRegisteredEvent>();
```

### Handle any messages

``` c#
public class RegisterMemberHandler : IMessageHandler<RegisterMemberCommand>
{
	public void ProcessMessage(IHandlerContext<RegisterMemberCommand> context)
	{
		Console.WriteLine();
		Console.WriteLine("[MEMBER REGISTERED] : user name = '{0}'", context.Message.UserName);
		Console.WriteLine();

		context.Publish(new MemberRegisteredEvent
		{
			UserName = context.Message.UserName
		});
	}
}
```

``` c#
public class MemberRegisteredHandler : IMessageHandler<MemberRegisteredEvent>
{
	public void ProcessMessage(IHandlerContext<MemberRegisteredEvent> context)
	{
		Console.WriteLine();
		Console.WriteLine("[EVENT RECEIVED] : user name = '{0}'", context.Message.UserName);
		Console.WriteLine();
	}
}
```
