# Getting Started Part 1 - An Introduction to View-Model Initialization, Properties, and Commands
## Introduction
In this article, you see how to create a simple cross-platform app using Codon. You look at creating a .NET Standard library to contain your apps UI and business logic. You explore how to create view-models by deriving from a `ViewModelBase` class. You then touch on using one of Codon's core services: the dialog service, to display messages to the user.

The code presented herein is located in Sample001 in the [Samples repository](https://github.com/CodonFramework/Samples)

When creating a cross-platform app the question of where to place business logic soon arises. In the past, the answer was PCLs, shared projects, or file linking. None of which offered a friction free development experience. Fortunately, there's now a fourth option: .NET Standard library projects. .NET Standard allows you to use most of the APIs in .NET and according to Microsoft, .NET Standard is set to replace PCLs. If you're interested in .NET Standard, you can read more about it over [here](https://blogs.msdn.microsoft.com/dotnet/2016/09/26/introducing-net-standard/).

You do **not**, however, have to understand .NET Standard to use Codon.

## Setting Up Your Solution
To create a .NET Standard class library, select 'Add new project' in Visual Studio 2017 and choose .NET Standard from the list of project types.

Right click on the project node and select Manage NuGet Packages. Enter Codon in the Search box of the Browse tab. You'll see a bunch of Codon packages listed. Select "Codon", which is a .NET Standard library that contains the core features of Codon.

With the NuGet reference to Codon, you can begin creating view-models for your app.

## Exploring the Sample Code
View-models in a Codon based app usually subclass one of the view-model base classes. In the sample `Page1ViewModel` subclasses `ViewModelBase`, located in the `Codon.UIModel` namespace.

### Creating Binding Friendly Properties
`Page1ViewModel` contains a property named `Foo`, which demonstrates how to create a property that will automatically raise `PropertyChanging` and `PropertyChanged` when its value is set. See Listing 1.

The get and set accessors are implemented using C# 7's expression syntax.

UI elements have thread affinity with the UI thread. All changes must be performed on the UI thread. That's where Codon's `ViewModelBase` comes in handy, because the base class's `Set` method takes care of ensuring that the property change events are always raised on the UI thread.

**Listing 1.** Using the Set method.

``` csharp
string foo;

public string Foo
{
	get => foo;
	set => Set(ref foo, value);
}
```

If you need to check whether the value was actually set, the `Set` method returns an `AssignmentResult` enumeration value, which can be one of the following:

* AlreadyAssigned: Indicates that the field was already set to the specified value.
* Cancelled: Indicates that a subscriber to the class PropertyChanging event, decided to cancel the update.
* OwnerDisposed: Indicates that view-model was disposed.
* Success: Indicates that the field was set to the specified value.

### Using Dependency Injection to Initialize a View-Model
Codon's default IoC container supports dependency injection, so that if we request an instance of the `Page1ViewModel` it is automatically populated with the dialog service, navigation service, and settings service. See Listing 2. We explore the services later in the article.

I've used C# 7's new relaxed throw statements, combined with the null-coalescing operator to ensure that none of the arguments is null.

**Listing 2.** Page1ViewModel constructor.

```cs
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

In XAML based apps, `ICommands` are a commonly used way to connect UI elements, such as buttons, to your apps business logic. In other words, `ICommands` encapsulate what needs to be done when, for example, a button is tapped. In Codon, the principle `ICommand` implementation is the `ActionCommand` class. An `ActionCommand` is initialized with an `Action`, which is invoked when the `ActionCommand` is executed.

`Page1ViewModel` contains a number of commands. Let's take a look at the `ShowDialogCommand`, which is declared as shown:
```cs
ActionCommand showDialogCommand;
public ICommand ShowDialogCommand => showDialogCommand
	?? (showDialogCommand = new ActionCommand(ShowDialog));
```


I've used a lambda expression for the property, which is a more concise way to express simple properties. The command is lazily instantiated, meaning that when the property is first retrieved, the `ActionCommand` is created.

**NOTE:** Of course, the `ActionCommand` class also allow you to provide a `Func` to determine if the `ICommand` is executable. 

As an aside, there are other variations of the `ActionCommand` in Codon, such as the `UICommand`, which has various properties, such as `Text` and `Visible` properties; and the `UICompositeCommand` that allows you to bind multiple commands to a single UI element. The Extras package also includes support for asynchronous commands, but advanced commanding is outside the scope of this article.

The `ShowDialog` method is invoked when the `ICommand.Execute` method is called.

`ShowDialog` uses the `IDialogService.ShowMessageAsync` method to display a message box to the user, as shown:
```cs
void ShowDialog(object arg)
{
	dialogService.ShowMessageAsync(
		"This is a sample message.", "Message from Sample 1");
}
```

There is an `IDialogService` implementation for each platform. The dialog service can be used to not only display messages or warnings, but also to show toasts and ask the user text response questions. We'll explore that in a later article.

Before we proceed to examining the settings and navigation services, let's explore how the view-model is bound to a view.

In the sample, we have four projects. The first, a .NET Standard library, we have already looked at. The other three include a Xamarin Android app project, a WPF app project, and a UWP app project.

All three follow the same pattern, they each contain pages (or in the case of the Android project, activities) that are coupled with the view-models in the .NET Standard project.

Let's look at the UWP project first.

The *Sample1.Uwp* project references the *Codon.Uwp* NuGet package. The *Codon.Uwp* package complements the Codon package with, for example, a UWP platform specific implementation of the `IDialogService`.

Each page in the project retrieves its respective view-model from the IoC container using the `Dependency` class. See Listing 3.

In the UWP app we also declare the view-model as a property of the view. This allows the use of the `x:Bind` markup extension, as we see in a moment.

**Pro Tip**
You may be wondering how Codon knows about the `IDialogService`, `ISettingsService`, or the `INavigationService` implementations. Codon's IoC container, `FrameworkContainer`, supports declarative type associations. So, no bootstrapper is required. If you take a look at any of the services, you'll see they are decorated with a `DefaultTypeNameAttribute` and/or a `DefaultTypeAttrubute`, which tells the container where to find an implementation. In addition, you can use the built-in `System.ComponentModel.DefaultValue` attribute to specify a default implementation, without referencing the Codon core .NET Standard library.

Of course, you can override the default type association by simply registering a type association using the `Dependency.Register` method. 

**Listing 3.** Sample1.Uwp Page1 class excerpt.
```cs
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

You may assume that we have to something radically dissimilar with Xamarin Android, since there's no built-in binding infrastructure. Well, lo and behold, Codon has support for layout file binding, as demonstrated by this excerpt from the *Sample1.Android* project's *Page1.axml* layout file:

```xml
<Button
        l:Binding="Target=Click, Path=ShowDialogCommand"
        android:id="@+id/Page1_Button_ShowDialog"
        android:text="Show Dialog" />
```

**NOTE:** Layout file binding requires that the Android project has a reference to the *Codon.Extras* package.

We explore how to configure data-binding in Android later.

## Conclusion


In this article, you saw how to create a simple cross-platform app using Codon. You looked at creating a .NET Standard library to contain your apps UI and business logic. You explored how to create view-models by deriving from a `ViewModelBase` class. You then touched on using one of Codon's core services: the dialog service, to display messages to the user.
In the [next part]( 002_Getting_Started_2.html), we return to .NET Standard project to further explore the view-model logic and we look at using Codon's settings service. 

