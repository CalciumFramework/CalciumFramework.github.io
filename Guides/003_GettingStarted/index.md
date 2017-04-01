# Getting Started Part 3 - Passing Messages Between App Components

## Introduction
In the [previous article](002_Getting_Started) we looked at using Codon's settings service to store and retrieve settings. In this article, you look at enabling communication between components in your app.

The code presented herein is located in Sample001 in the [Samples repository](https://github.com/CodonFramework/Samples)

Communication between components is an essential aspect of any mildly complex app. By using a weak referencing pub/sub messaging system you can decouple components and reduce the likelihood of memory leaks caused by event subscriptions.
Codon's pub/sub messaging service is an implementation of the `IMessenger` interface. It allows components in your app to subscribe to message objects of a particular type, and for other components to dispatch those messages. It's used extensively inside Codon itself for such things as notifying subscribers that a setting has changed. 

## Understanding the Messenger
Codon's `IMessenger` implementation uses an interface based subscription model. You nominate what messages you'd like your class to receive by implementing the `IMessageSubscriber<T>` interface. The CLR `Type` of `T` determines the signature or identity of the message.  

**NOTE:** This declarative approach to message subscription increases transparency because message subscription is visible at the type level.

To receive messages as they are published, a subscriber needs to call the messenger's Subscribe method. 


**NOTE:** If you derive from`ViewModelBase` the messenger's Subscribe method is called for you in the base class constructor. 

The Sample's ` Page1ViewModel` class declares a subscription to the `ExampleMessage` message by implementing the `IMessageSubscriber<ExampleMessage>` interface. See Listing 1. Message objects can be POCOs. The `ExampleMessage` class does not derive from a Codon base class.

Behind the scenes, when the `ViewModelBase` class calls the messenger's `Subscribe` method, the messenger creates a weak reference to the view-model and calls the `ReceiveMessageAsync(ExampleMessage message)` method whenever an `ExampleMessage` object is published.

**TIP:** You can unsubscribe from the `IMessenger` by calling its `Unsubscribe` method.

**Listing 1.** Page1ViewModel Implements the IMessageSubscriber<ExampleMessage> interface.

```cs
public class Page1ViewModel : ViewModelBase, 
		IMessageSubscriber<ExampleMessage>
    {
...
	public Task ReceiveMessageAsync(ExampleMessage message)
	{
		dialogService.ShowMessageAsync(
			"Received the message.", "Message from App");

		return Task.FromResult<object>(null);
	}
...
}
```

If an object calls the messenger's `PublishAsync` method using an `ExampleMessage` object as the parameter, the page receives the message.

## Dispatch the Message from the View-Model

The `Page1ViewModel` class contains a `PublishMessageCommand`, which is bound to a button in the view.

```cs
ActionCommand publishMessageCommand;

public ICommand PublishMessageCommand => publishMessageCommand 
?? (publishMessageCommand = new ActionCommand(PublishMessage));
```

When the button is tap/clicked, the `PublishMessage` method is called. See Listing 2.

The `ViewModelBase` class contains a `Messenger`, which means we don't need to use the dependency injection to retrieve it. 

**NOTE:** `PublishAsync` is an *awaitable* method, which is useful, for example, if your message object contains a property allowing an action to be cancelled. Codon has some prebaked messages to use for this purpose. An example of which is the `CancellableMessageBase` class.

**NOTE:** It is not necessary to derive message classes from a base class. POCOs are fine.

**Listing 2:** Page1ViewModel PublishMessage method.
```cs
void PublishMessage(object arg)
{
	Messenger.PublishAsync(new ExampleMessage(), true);
}
```

Tapping the *Publish Message* button on *Page 1* dispatches the message, and it is received within the same view-model. See Figure 1. Although pointless, it demonstrates how to send and receive a message. You wouldn't ordinarily subscribe to a message that is only dispatched from the same class.

![Dialog confirming message received](/Guides/003_GettingStarted/MessageReceivedDialog.png)

**Figure 1.** Dialog box confirms message received.

**Pro Tip:** Some classes in Codon, such as the `Messenger` and `ActionCommand` classes implement a global exception handling system. By default, if an exception is raised by one of these classes, the default exception handler class, `LoggingExceptionHandler`, logs the exception and request that the exception be rethrown. This alleviates the need to wrap all code blocks in try/catch blocks. 

**NOTE:** While UWP and WPF have the means to catch unhandled exceptions, the same is not true for Android. If there's an unhandled exception in your Android app, the OS will terminate it. It is recommended that you implement your own IExceptionHandler and register it with the IoC container, like so:

```cs
Dependency.Register<IExceptionHandler>(() => new MyExceptionHandler(), true);
```

## Conclusion
In this article, you looked at enabling communication between components using the `Messenger` class. In the [next part](004_Getting_Started), you look at navigating between pages using the navigation service


