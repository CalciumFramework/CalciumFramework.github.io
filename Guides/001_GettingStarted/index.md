# Getting Started Part 1 - An Introduction to View-Model Initialization, Properties, and Commands
## Introduction
In this article, you see how to create a simple cross-platform app using Calcium. You look at creating a .NET Standard library to contain your apps UI and business logic. You explore how to create view-models by deriving from a `ViewModelBase` class. You then touch on using one of Calcium's core services: the dialog service, to display messages to the user.

The code presented herein is located in Sample001 in the [Samples repository](https://github.com/CalciumFramework/Samples)

When creating a cross-platform app, the question of where to place business logic soon arises. In the past, the answer was PCLs, shared projects, or file linking. None of which offered a friction-free development experience. Fortunately, there's now a fourth option: .NET Standard library projects. .NET Standard allows you to use most of the APIs in .NET and, according to Microsoft, .NET Standard is set to replace PCLs. If you're interested in .NET Standard, you can read more about it over [here](https://blogs.msdn.microsoft.com/dotnet/2016/09/26/introducing-net-standard/).

By the way, you do **not** have to understand .NET Standard to use Calcium.

## Setting Up Your Solution
To create a .NET Standard class library, select 'Add new project' in Visual Studio 2017 and choose .NET Standard from the list of project types.

Right click on the project node and select Manage NuGet Packages. Enter Calcium in the Search box of the Browse tab. You'll see a bunch of Calcium packages listed. Select "Calcium", which is a .NET Standard library that contains the core features of Calcium.

With the NuGet reference to Calcium, you can begin creating view-models for your app.

## Exploring the Sample Code
View-models in a Calcium based app usually subclass one of the view-model base classes. In the sample, `Page1ViewModel` subclasses ` Calcium.UIModel.ViewModelBase`.

### Creating Binding Friendly Properties
`Page1ViewModel` contains a property named `Foo`, which demonstrates how to create a property that automatically raises `PropertyChanging` and `PropertyChanged` when its value is set. See Listing 1.

The get and set accessors are implemented using C# 7's expression syntax.

UI elements in UWP, WPF, and Android have thread affinity with the UI thread. That is, all changes affecting a UI elements state must be performed on the UI thread. That's where Calcium's `ViewModelBase` comes in handy, because the base class's `Set` method takes care of ensuring that the property change events are always raised on the UI thread.

**Listing 1.** Using the Set method.

```csharp
string foo;

public string Foo
{
	get => foo;
	set => Set(ref foo, value);
}
```

If you need to check whether the value was actually set, the `Set` method returns an `AssignmentResult` enumeration value, which can be one of the following:

* **AlreadyAssigned**: Indicates that the field was already set to the specified value.
* **Cancelled**: Indicates that a subscriber to the class PropertyChanging event, decided to cancel the update.
* **OwnerDisposed**: Indicates that view-model was disposed.
* **Success**: Indicates that the field was set to the specified value.

### Using Dependency Injection to Initialize a View-Model
Calcium's default IoC container supports dependency injection. If we request an instance of the `Page1ViewModel`, the resultant object is automatically populated with the dialog service, navigation service, and settings service. See Listing 2. We explore these services a bit later.

The `Page1ViewModel` uses C# 7's new relaxed throw statements, combining them with the null-coalescing operator, to ensure that none of the arguments is null.

**Listing 2.** Page1ViewModel constructor.

```csharp
public Page1ViewModel(
	IDialogService dialogService, 
	INavigationService navigationService,
	ISettingsService settingsService)
{
	this.dialogService = dialogService 
		?? throw new ArgumentNullException(nameof(dialogService));
	this.navigationService = navigationService 
		?? throw new ArgumentNullException(nameof(navigationService));
	this.settingsService = settingsService 
		?? throw new ArgumentNullException(nameof(settingsService));

	UpdateCreationCount();
}
```

In XAML based apps, `ICommands` are a commonly used way to connect UI elements, such as buttons, to your apps business logic. `ICommands` encapsulate what needs to be done when, for example, a button is tapped. In Calcium, the principle `ICommand` implementation is the `ActionCommand` class. An `ActionCommand` must be initialized with an `Action` that is invoked when the `ActionCommand` is executed.

`Page1ViewModel` contains a number of commands. Let's take a look at the `ShowDialogCommand`, which is declared as shown:
```csharp
ActionCommand showDialogCommand;
public ICommand ShowDialogCommand => showDialogCommand
	?? (showDialogCommand = new ActionCommand(ShowDialog));
```


The property uses lambda expressions for the getter and setter accessors. Its a concise way to express simple properties. The command is lazily instantiated, meaning that when the property is first retrieved, the `ActionCommand` is created.

**NOTE:** The `ActionCommand` class also allows you to provide a `Func` to determine if the `ICommand` is executable. 

As an aside, there are other variations of the `ActionCommand` in Calcium, including the `UICommand`, which has various properties, such as `Text` and `Visible` properties. In addition, the `UICompositeCommand` allows you to bind multiple commands to a single UI element. The Extras package also includes support for asynchronous commands if you need it. Advanced commanding is, however, outside the scope of this article.

The `ShowDialog` method is invoked when the `ICommand.Execute` method is called.

`ShowDialog` uses the `IDialogService.ShowMessageAsync` method to display a message box to the user, as shown:
```csharp
void ShowDialog(object arg)
{
	dialogService.ShowMessageAsync(
		"This is a sample message.", "Message from Sample 1");
}
```

An `IDialogService` implementation exists for each supported platform. The dialog service can be used to not only display messages or warnings, but also to show toasts and ask the user text response questions. We'll explore that in a later article.

Before we proceed to examining the settings and navigation services, let's take a look at how the view-model is bound to a view.

In the sample, we have four projects. The first, we have already looked at: a .NET Standard library. The other three include a Xamarin Android app project, a WPF app project, and a UWP app project.

All three follow the same pattern. They each contain pages (or in the case of the Android project, activities) that are coupled with the view-models in the .NET Standard project.

Let's look at the UWP project first.

The *Sample1.Uwp* project references the *Calcium.Uwp* NuGet package. The *Calcium.Uwp* package complements the Calcium package with a UWP platform specific implementation of the `IDialogService`.

Each page in the project retrieves its respective view-model from the IoC container using the `Dependency` class. See Listing 3.

In the UWP app we declare the view-model as a property of the view. This allows the use of the `x:Bind` markup extension, as we see in a moment.

**Pro Tip**
You may be wondering how Calcium knows about the `IDialogService`, `ISettingsService`, or the `INavigationService` implementations. Calcium's IoC container, `FrameworkContainer`, supports declarative type associations. So, no bootstrapper is required. If you take a look at any of the services, you'll see they are decorated with a `DefaultTypeNameAttribute` and/or a `DefaultTypeAttrubute`, which tells the container where to find an implementation. In addition, you can use the built-in `System.ComponentModel.DefaultValue` attribute to specify a default implementation, without referencing the Calcium core .NET Standard library.

You can override a default type association by registering a type association using the `Dependency.Register` method. 

**Listing 3.** Sample1.Uwp Page1 class excerpt.
```csharp
public sealed partial class Page1 : Page
{
    public Page1()
    {
	/* Using the Dependency class to resolve 
	 * the view-model automatically causes it to receive 
	 * the services it needs via dependency injection. */
	 DataContext = Dependency.Resolve<Page1ViewModel>();
	        
        InitializeComponent();
    }

	public Page1ViewModel ViewModel => (Page1ViewModel)DataContext;
}
```

*Page1.xaml* contains various buttons that are bound to commands in the view-model. The following excerpt shows how the view is bound to the view-model's `ShowDialogCommand`.

```xml
<Button Command="{x:Bind ViewModel.ShowDialogCommand}">Show Message 1</Button>
```

When the user activates the *Show Message 1* button, a dialog is displayed. See Figure 1.

![Dialog Box showing message](/Guides/001_GettingStarted/MessageShown.png)

**Figure 1.** Dialog box displays message to user.

The WPF version of the app looks much the same. It binds to the view-model's `ShowDialogCommand`, like so:

```xml
<Button Command="{Binding ShowDialogCommand}">Show Message</Button>
```

There's no built-in binding infrastructure for Xamarin Android. So, you'd be forgiven for thinking that we'd have to do something radically different to enable data-binding. Not so. Calcium has support for layout file binding, as demonstrated by this excerpt from the *Sample1.Android* project's *Page1.axml* layout file:

```xml
<Button
        l:Binding="Target=Click, Path=ShowDialogCommand"
        android:id="@+id/Page1_Button_ShowDialog"
        android:text="Show Dialog" />
```

**NOTE:** Layout file binding requires that the Android project has a reference to the *Calcium.Extras* package.

## Conclusion


In this article, you saw how to create a simple cross-platform app using Calcium. You looked at creating a .NET Standard library to contain your apps UI and business logic. You explored how to create view-models by deriving from a `ViewModelBase` class. You then touched on using one of Calcium's core services: the dialog service, to display messages to the user.
In the [next part](../002_GettingStarted), we return to .NET Standard project to further explore the view-model logic and we look at using Calcium's settings service. 

