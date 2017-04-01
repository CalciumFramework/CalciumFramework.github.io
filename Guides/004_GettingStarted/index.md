# Getting Started Part 4 - Navigating Between Pages Using the Navigation Service
In the [previous article](003_Getting_Started_3.html) you looked at enabling communication between components in your app using the `Messenger` class. In this article, you look at enabling page navigation using the routing service in conjunction with the navigation service.

The code presented herein is located in Sample001 in the [Samples repository](https://github.com/CodonFramework/Samples)

The page navigation APIs differ significantly across platforms. Some platforms, like the UWP, allow you to navigate to a page according to its `Type`. In WPF, you navigate to a Page instance. While others use a completely different scheme, such as Android and its use of `Intents`. To ease these differences, Codon employs a routing system, that allows you to register a string URL with an associated `Action`. When the navigation service receives a request to navigate to a particular URL, the action is invoked. 

**NOTE:** The URL can, in fact, be any string, and serves merely as a key to look up the associated Action. 

Codon *doesn't* require bootstrapping. Default IoC container registrations are performed automatically. However, if you intend to use Codon's navigation service for anything besides back navigation, you need to configure its routes. 

In the sample, you can find a class named Bootstrapper in each platform project. Its sole purpose is to configure the routing service. 

Each of the `Bootstrapper` classes contain a `Run` method, which is called when the application launches. The `Bootstrapper` in the UWP sample project, retrieves the `IRoutingService` from the IoC container and registers the path "/Page2" (contained within the Routes class) with an Action to call the `Bootstrapper` class's `Navigate` method. See Listing 1.

**Listing 1.** UWP Sample Bootstrapper class.

```cs
public void Run()
{
	/* This method should be called when your app starts. 
		* The IoC container knows about several default types, 
		* that's why no type registrations are necessary. */
	var routingService = Dependency.Resolve<IRoutingService>();

	/* When the navigation service receives a request for "/Page2", 
		* it uses the routing service to look up the path 
		* and calls Navigate<Page2>(). */
	routingService.RegisterPath(Routes.Page2, Navigate<Page2>);
}
```

The generic `Navigate` method calls the non-generic navigate method, which then retrieves the `Frame` object from the `Window`, and calls its `Navigate` method with the page `Type` as a parameter. See Listing 2.

**NOTE:** You don't have set up the `Bootstrapper` like this. It's merely a guide. Remember, the routing service accepts an `Action`, so the flexibility is there to do whatever you like.

**Listing 2.** UWP Bootstrapper Navigate methods
```cs
void Navigate<TPage>()
{
	Navigate(typeof(TPage));
}

void Navigate(Type pageType)
{
	/* Navigation in UWP is achieved using the root Frame. */
	var frame = (Frame)Window.Current.Content;
	frame.Navigate(pageType);
}
```

The sample `Bootstrapper` for WPF look the same as the UWP `Bootstrapper`, apart from the `Navigate` method. See Listing 3.

The Codon `INavigationService` implementation for WPF contains a convenient helper method, which locates the built-in `System.Navigation.NavigationService` from current `Frame` or `Window`. Since we know the `Bootstrapper` is running on WPF we can safely case the `INavigationService` to its concrete implementation `NavigationService`. 

The WPF `Bootstrapper` uses the `Dependency` class to build-up the new page, which is then passed to the built-in `NavigationService` object.

**Listing 3.** WPF Sample Bootstrapper Navigate method

```cs
void Navigate(Type pageType)
{
	var navigationService = (NavigationService)Dependency.Resolve<INavigationService>();
	var page = Dependency.ResolveWithType(pageType);

	/* The platform specific implementation of INavigationService 
		* has a Navigate(object) method that we can use navigate. 
		* It automatically resolves the root Frame or NavigationPage. */
	navigationService.Navigate(page);
}
```

The `Bootstrapper` class in the Sample Android app, works much the same as the other platforms. The main difference, however, is that it makes use of Android Intents. The `Bootstrapper` class's `Run` method retrieves the `IRoutingService` instance and registers a *Page 2* route with an associated call to the `LaunchActivity` method, as shown:

```cs
routingService.RegisterPath(Routes.Page2, () => LaunchActivity<Page2Activity>(1));
```

Navigating in Xamarin Android requires an `Activity` or `Context` object. Because there isn't a global handle to the current `Activity`, Codon requires that when an `Activity` becomes active, that it is registered with the IoC container. 

The non-generic `LaunchActivity` method of the Android Sample `Bootstrapper` retrieves the current `Activity` from the IoC container. See Listing 4. It then creates an `Intent` for the new activity. The new activity is started using the `Intent` object and a request code. More on request codes in a later article.

**Listing 4.** Android Sample Bootstrapper LaunchActivity methods
```cs
void LaunchActivity<TActivity>(int requestCode)
{
	LaunchActivity(typeof(TActivity), requestCode);
}

void LaunchActivity(Type activityType, int requestCode)
{
	var activity = Dependency.Resolve<Activity>();
			
	/* Launch an intent for the activity. */
	Intent intent = new Intent(activity, activityType);
	activity.StartActivityForResult(intent, requestCode);
}
```

Returning to the `Page1ViewModel`, you see that the `ActionCommand` named `NavigateToPage2Command` calls the view-models `NavigateToPage2` method, as shown:
```cs
ActionCommand navigateToPage2Command;

public ICommand NavigateToPage2Command => navigateToPage2Command
	?? (navigateToPage2Command = new ActionCommand(NavigateToPage2));
```

The `NavigateToPage2` method uses the navigation service to navigate to URL defined in Routes.Page2 ("/Page2"). See Listing 5. The navigation service then asks the routing service if there is a route registered with the path "/Page2", and as there is, invokes the `Action` associated with the URL. 

**Listing 5.** Page1ViewModel NavigateToPage2 method

```cs
void NavigateToPage2(object arg)
{
	navigationService.Navigate(Routes.Page2);
}
```

To recap, use the `IRoutingService` to associate URLs with `Actions` that perform the navigation. Use the `INavigationService` to navigate to the URLs.

There is a button on each of the platform Page1 views that is bound to the `NavigateToPage2Command`. When activating the button, the UWP/WPF/Android apps navigate to Page 2. On Page 2 there is a button that is bound to the `Page2ViewModel` class's `NavigateBackCommand`. When the command executes, the `INavigationService` is called upon to perform a back navigation, like so:

```cs
void NavigateBack(object arg)
{
	navigationService.GoBack();
}
```

Each platform specific implementation of `INavigationService` knows how to perform a back navigation for its particular platform.

In this article, you look at enabling page navigation using the routing service in conjunction with the navigation service.

