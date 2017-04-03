# Getting Started Part 2 - Storing Data with the Settings Service
## Introduction
In the [previous article](../001_GettingStarted) we familiarised ourselves with view-model initialization and working with properties and commands. In this article, you see how to save and retrieve persistent data using the settings service. 

The code presented herein is located in Sample001 in the [Samples repository](https://github.com/CodonFramework/Samples)

Just about every app needs to store and retrieve settings. Codon provides an abstracted API for storing and retrieving settings via its `ISettingsService` implementations.  You may wonder why it might be a good idea to abstract a set of APIs that is seemingly present across all supported platforms. There are a couple of reasons. While there exists an Isolated Storage Settings API in .NET Standard, the API doesn't behave as you might expect across all platforms. In particular, in the WPF implementation `IsolatedStorageSettings.ApplicationSettings` is null. Codon works around this issue with a custom implementation supporting WPF \*. 

Another reason for abstracting the settings API is that it affords you the opportunity to swap out the underlying storage provider with, for example, an SQLite implementation. I did this for one of my apps when users started reporting settings being reset and I realised a more robust storage provider was required.

**\*** See the `IsolatedStorageSettingsWpf` class in the Codon.Platform.Shared project if you're interested in the custom WPF `IsolatedStorageSettings` implementation.

**NOTE:** The settings service is able to serialize and store any object or primitive. Binary serialization is used for complex types.

## Demonstrating the Settings Service

The `Page1ViewModel` class's `UpdateCreationCount` method is called whenever the view-model is instantiated. See Listing 1. The method retrieves the setting with the key *Page1CreationCount* along with a default value. If no such setting exists, then the default value is returned.

The setting is then incremented via `ISettingsService.SetSetting`. The view-model's `CreationCount` property is set to the setting value.

**Listing 1.** Page1ViewModel.UpdateCreationCount Method

```cs
void UpdateCreationCount()
{
	const string key = "Page1CreationCount";
	/* Use the settings service to retrieve, increment, 
	 * and store a counter. 
	 * The settings service is cross-platform. */
	int count = settingsService.GetSetting(key, 0);
	settingsService.SetSetting(key, ++count);

	CreationCount = count;
}
```

We bind the CreationCount property to a text field in UWP project, like so:

```xml
<TextBlock Text="{x:Bind ViewModel.CreationCount, Mode=OneWay}" />
```

In the WPF project, it is bound in a similar manner (without using x:Bind):

```xml
<TextBlock Text="{Binding CreationCount}" />
```

While in the Android project, we do this:

```xml
<TextView
        l:Binding="Target=Text, Path=CreationCount"
        android:id="@+id/Page1_TextView_ShownCount" ... />
```

**NOTE:** Codon's Android data-binding infrastructure requires that views with data-bindings have an `android:id` attribute.

## Conclusion

In this article, you saw how to save and retrieve persistent data using the settings service. In the [next part](../003_GettingStarted), we look at passing messages between app components. 



