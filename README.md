# React Native Plugin for CodePush

This plugin provides client-side integration for the [CodePush service](https://microsoft.github.io/code-push), allowing you to easily add a dynamic update experience to your React Native app(s).

*Note: This README covers the current state of the master branch, and therefore, may include slight API differences as compared to what is published to NPM. For the latest "stable" documentation, please refer to the [website](http://microsoft.github.io/code-push/docs/react-native.html).*

## How does it work?

A React Native app is composed of a JavaScript bundle file, which is generated by the [packager](https://github.com/facebook/react-native/tree/master/packager) and distributed as part of your app's binary (i.e. the `.ipa` or `.apk` file). Once the app is released, updating the JavaScript code (e.g. making bug fixes, adding new features) requires you to recompile and redistribute the entire binary, which of course, includes any review time associated with the store(s) you are publishing to.

The CodePush plugin helps get product improvements in front of your end-users instantly, by keeping the JavaScript bundle synchronized with updates that are released to the CodePush server. This way, your app gets the benefits of an offline mobile experience, as well as the "web-like" agility of side-loading updates as soon as they are available. It's a win-win!

*Note: Any product changes which touch native code (e.g. modifying your `AppDelegate.m` file, adding a new plugin) cannot be distributed via CodePush, and therefore, must be updated via the appropriate store(s).*

## Supported React Native platforms

- iOS
- Coming soon: Android (Try it out on [this branch](https://github.com/Microsoft/react-native-code-push/tree/first-check-in-for-android))

## Getting Started

Once you've followed the overall ["getting started"](http://microsoft.github.io/code-push/docs/getting-started.html) instructions for setting up your CodePush account, you can acquire the React Native CodePush plugin by running the following command from within your app's root directory:

```
npm install --save react-native-code-push
```

## Plugin Installation 

Once you've acquired the CodePush plugin, you need to integrate it into the Xcode project of your React Native app. To do this, take the following steps:

1. Open your app's Xcode project
2. Find the `CodePush.xcodeproj` file within the `node_modules/react-native-code-push` directory, and drag it into the `Libraries` node in Xcode

    ![Add CodePush to project](https://cloud.githubusercontent.com/assets/516559/10322414/7688748e-6c32-11e5-83c1-00d3e6758df4.png)

3. Select the project node in Xcode and select the "Build Phases" tab of your project configuration.
4. Drag `libCodePush.a` from `Libraries/CodePush.xcodeproj/Products` into the "Link Binary With Libraries" secton of your project's "Build Phases" configuration.

    ![Link CodePush during build](https://cloud.githubusercontent.com/assets/516559/10322221/a75ea066-6c31-11e5-9d88-ff6f6a4d6968.png)

5. Under the "Build Settings" tab of your project configuration, find the "Header Search Paths" section and edit the value.
Add a new value, `$(SRCROOT)/../node_modules/react-native-code-push` and select "recursive" in the dropdown.

    ![Add CodePush library reference](https://cloud.githubusercontent.com/assets/516559/10322038/b8157962-6c30-11e5-9264-494d65fd2626.png)

## Plugin Configuration

Once your Xcode project has been setup to build/link the CodePush plugin, you need to configure your app to consult CodePush for the location of your JS bundle, since it is responsible for synchronizing it with updates that are released to the CodePush server. To do this, perform the following steps:

1. Open up the `AppDelegate.m` file, and add an import statement for the CodePush headers:

    ```
    #import "CodePush.h"
    ```

2. Find the following line of code, which loads your JS Bundle from the app binary for production releases:

    ```
    jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
    ```
    
3. Replace it with this line:

    ```
    jsCodeLocation = [CodePush bundleURL];
    ```

This change configures your app to always load the most recent version of your app's JS bundle. On the first launch, this will correspond to the file that was compiled with the app. However, after an update has been pushed via CodePush, this will return the location of the most recently installed update.

*NOTE: The `bundleURL` method assumes your app's JS bundle is named `main.jsbundle`. If you have configured your app to use a different file name, simply call the `bundleURLForResource:` method (which assumes you're using the `.jsbundle` extension) or `bundleURLForResource:withExtension:` method instead, in order to overwrite that default behavior*

To let the CodePush runtime know which deployment it should query for updates against, perform the following steps:

1. Open your app's `Info.plist` and add a new `CodePushDeploymentKey` entry, whose value is the key of the deployment you want to configure this app against (e.g. the Staging deployment for FooBar app). You can retreive this value by running `code-push deployment ls <appName>` in the CodePush CLI, and copying the value of the `Deployment Key` column which corresponds to the deployment you want to use. 
2. In your app's `Info.plist` make sure your `CFBundleShortVersionString` value is a valid [semver](http://semver.org/) version (e.g. 1.0.0 not 1.0)

## Plugin consumption

With the CodePush plugin downloaded and linked, and your app asking CodePush where to get the right JS bundle from, the only thing left is to add the necessary code to your app to control the following policies:

1. When (and how often) to check for an update? (e.g. app start, in response to clicking a button in a settings page, periodically at some fixed interval)
2. When an update is available, how to present it to the end-user?

The simplest way to do this is to perform the following in your app's root component:

1. Import the JavaScript module for CodePush:

    ```
    var CodePush = require("react-native-code-push")
    ```

2. Call the `sync` method from within the `componentDidMount` lifecycle event, to initiate a background update on each app start:

    ```
    CodePush.sync();
    ```

If an update is available, it will be silently downloaded, and installed the next time the app is restarted (either explicitly by the end-user or by the OS), which ensures the least invasive experience for your end-users. If you would like to display a confirmation dialog (an "active install"), or customize the update experience in any way, refer to the `sync` method's [API reference](#codepushsync) for information on how to tweak this default behavior.

## Releasing code updates

Once your app has been configured and distributed to your users, and you've made some JS changes, it's time to release it to them instantly! To do this, run the following steps:

1. Execute `react-native bundle` in order to generate the updated JS bundle for your app.
2. Execute `code-push release <appName> ./ios/main.jsbundle <appVersion> --deploymentName <deploymentName>` in order to publish the generated JS bundle to the server (assuming your CWD is the root directory of your React Native app). 

And that's it! For more information regarding the CodePush API, including the various options you can pass to the `sync` method, refer to the reference section below. Additionally, for more information regarding the CLI and how the release (or promote) commands work, refer to it's [documentation](http://microsoft.github.io/code-push/docs/cli.html).

---

## API Reference

When you require `react-native-code-push`, the module object provides the following top-level methods:

* [checkForUpdate](#codepushcheckforupdate): Asks the CodePush service whether the configured app deployment has an update available. 

* [getCurrentPackage](#codepushgetcurrentpackage): Gets information about the currently installed "package" (e.g. description, installation time, size).

* [notifyApplicationReady](#codepushnotifyapplicationready): Notifies the CodePush runtime that an installed update is considered successful. This is an optional API, but is useful when you want to expicitly enable "rollback protection" in the event that an exception occurs in code that you've deployed to production.

* [restartPendingUpdate](#codepushrestartPendingUpdate): Conditionally restarts the app if a previously installed update is currently pending (e.g. it was installed using the `ON_NEXT_RESTART` or `ON_NEXT_RESUME` modes, and the app hasn't been restarted or resumed yet).

* [sync](#codepushsync): Allows checking for an update, downloading it and installing it, all with a single call. Unless you need custom UI and/or behavior, we recommend most developers to use this method when integrating CodePush into their apps

### codePush.checkForUpdate

```javascript
codePush.checkForUpdate(deploymentKey: String = null): Promise<RemotePackage>;
```

Queries the CodePush service to see whether the configured app deployment has an update available. By default, it will use the deployment key that is configured in your `Info.plist` file, but you can override that by specifying a value in the optional `deploymentKey` parameter.

This method returns a `Promise` which resolves to one of two values:

* `null` if there is no update available.
* A `RemotePackage` instance which represents an available update that can be inspected and/or subsequently downloaded.

Example Usage: 

```javascript
codePush.checkForUpdate()
.then((update) => {
    if (!update) {
        console.log("The app is up to date!"); 
    } else {
        console.log("An update is available! Should we download it?");
    }
});
```

### codePush.getCurrentPackage

```javascript
codePush.getCurrentPackage(): Promise<LocalPackage>;
```

Retreives the metadata about the currently installed "package" (e.g. description, installation time). This can be useful for scenarios such as displaying a "what's new?" dialog after an update has been applied.

This method returns a `Promise` which resolves to the `LocalPackage` instance that represents the currently running update. 

Example Usage: 

```javascript
codePush.getCurrentPackage()
.then((update) => {
    // If the current app "session" represents the first time
    // this update has run, and it had a description provided
    // with it upon release, let's show it to the end-user
    if (update.isFirstRun && update.description) {
        // Display a "what's new?" modal
    }
});
```

### codePush.notifyApplicationReady

```javascript
codePush.notifyApplicationReady(): Promise<void>;
```

Notifies the CodePush runtime that an update should be considered successful, and therefore, an automatic rollback isn't necessary. Calling this function is required whenever the `rollbackTimeout` parameter is specified when calling either ```LocalPackage.install``` or `sync`. If you specify a `rollbackTimeout`, and don't call `notifyApplicationReady`, the CodePush runtime will assume that the installed update has failed and roll back to the previous version.

If the `rollbackTimeout` parameter was not specified, the CodePush runtime will not enforce any automatic rollback behavior, and therefore, calling this function is not required and will result in a no-op.

### codePush.restartPendingUpdate		
		
```javascript		
codePush.restartPendingUpdate(): void;		
```		
		
Applies the pending update (if applicable) by immediately restarting the app, and optionally starting the rollback timer. This method is for advanced scenarios, and is only useful when the following conditions are true:		
		
1. Your app is specifying an install mode value of `ON_NEXT_RESTART` or `ON_NEXT_RESUME` when calling the `sync` or `LocalPackage.install` methods. This has the effect of not applying your update until the app has been restarted (by either the end-user or OS)	or resumed, and therefore, the update won't be immediately displayed to the end-user 	.
2. You have an app-specific user event (e.g. the end-user navigated back to the app's home route) that allows you to apply the update in an unobtrusive way, and potentially gets the update in front of the end-user sooner then waiting until the next restart or resume.		
		
If you call this method, and there isn't a pending update, it will result in a no-op. Otherwise, the app will be restarted in order to display the update to the end-user.

### codePush.sync

```javascript
codePush.sync(options: Object, syncStatusChangeCallback: function(syncStatus: Number), downloadProgressCallback: function(progress: DownloadProgress)): Promise<Number>;
```

Synchronizes your app's JavaScript bundle with the latest release to the configured deployment. Unlike the `checkForUpdate` method, which simplies checks for the presence of an update, and let's you control what to do next, `sync` handles the update check, download and installation experience for you.

This method provides support for two different (but customizable) "modes" to easily enables apps with different requirements:

1. **Silent mode** *(the default behavior)*, which automatically downloads available updates, and applies them the next time the app restarts. This way, the entire update experience is "silent" to the end-user, since they don't see any update prompt and/or "synthetic" app restarts.

2. **Active mode**, which when an update is available, prompts the end-user for permission before downloading it, and then immediately applies the update. If an update was released using the `mandatory` flag, the end-user would still be notified about the update, but they wouldn't have the choice to ignore it.


Example Usage: 

```javascript
// Fully silent update which keeps the app in
// sync with the server, without ever 
// interrupting the end-user
codePush.sync();

// Active update, which lets the end-user know
// about each update, and displays it to them
// immediately after downloading it
codePush.sync({ updateDialog: true, installMode: codePush.InstallMode.IMMEDIATE });
```

*Note: If you want to decide whether you check and/or download an available update based on the end-user's device battery level, network conditions, etc. then simply wrap the call to `sync` in a condition that ensures you only call it when desired.*

While the `sync` method tries to make it easy to perform silent and active updates with little configuration, it accepts an "options" object that allows you to customize numerous aspects of the default behavior mentioned above:

* __deploymentKey__ *(String)* - Specifies the deployment key you want to query for an update against. By default, this value is derived from the `Info.plist` file, but this option allows you to override it from the script-side if you need to dynamically use a different deployment for a specific call to `sync`.

* __installMode__ *(CodePush.InstallMode)* - Indicates when you would like to "install" the update after downloading it, which includes reloading the JS bundle in order for any changes to take affect. Defaults to `CodePush.InstallMode.ON_NEXT_RESTART`. Refer to the [`InstallMode`](#installmode) enum reference for a description of the available options and what they do.

* __rollbackTimeout__ *(Number)* - The number of milliseconds that you want the runtime to wait after an update has been installed before considering it failed and rolling it back. Defaults to `0`, which disables rollback protection. Note that if you set this parameter to a non-zero number, then your app needs to call `notifyApplicationReady` in order to prevent the automatic rollback from happening.

* __updateDialog__ *(UpdateDialogOptions)* - An "options" object used to determine whether a confirmation dialog should be displayed to the end-user when an update is available, and if so, what strings to use. Defaults to `null`, which has the effect of disabling the dialog completely. Setting this to any truthy value will enable the dialog with the default strings, and passing an object to this parameter allows enabling the dialog as well as overriding one or more of the default strings. The following list represents the available options and their defaults:
    * __appendReleaseDescription__ *(Boolean)* - Indicates whether you would like to append the description of an available release to the notification message which is displayed to the end-user. Defaults to `false`.
    
    * __descriptionPrefix__ *(String)* - Indicates the string you would like to prefix the release description with, if any, when displaying the update notification to the end-user. Defaults to `" Description: "`
    
    * __mandatoryContinueButtonLabel__ *(String)* - The text to use for the button the end-user must press in order to install a mandatory update. Defaults to `"Continue"`.
    
    * __mandatoryUpdateMessage__ *(String)* - The text used as the body of an update notification, when the update is specified as mandatory. Defaults to `"An update is available that must be installed."`.
    
    * __optionalIgnoreButtonLabel__ *(String)* - The text to use for the button the end-user can press in order to ignore an optional update that is available. Defaults to `"Ignore"`.
    
    * __optionalInstallButtonLabel__ *(String)* - The text to use for the button the end-user can press in order to install an optional update. Defaults to `"Install"`.
    
    * __optionalUpdateMessage__ *(String)* - The text used as the body of an update notification, when the update is optional. Defaults to `"An update is available. Would you like to install it?"`.
    
    * __title__ *(String)* - The text used as the header of an update notification that is displayed to the end-user. Defaults to `"Update available"`.

Example Usage:

```javascript
// Use a different deployment key for this
// specific call, instead of the one configured
// in the Info.plist file
codePush.sync({ deploymentKey: "KEY" });

// Download the update silently
// but install is on the next resume
// instead of waiting until the app is restarted
codePush.sync({ installMode: codePush.InstallMode.ON_NEXT_RESUME });

// Changing the title displayed in the
// confirmation dialog of an "active" update
codePush.sync({ updateDialog: { title: "An update is available!" } });
```

In addition to the options, the `sync` method also accepts two optional function parameters which allow you to subscribe to the lifecycle of the `sync` "pipeline" in order to display additional UI as needed (e.g. a "checking for update modal or a download progress modal):

* __syncStatusChangedCallback__ *((syncStatus: Number) => void)* - Called when the sync process moves from one stage to another in the overall update process. The method is called with a status code which represents the current state, and can be any of the [`SyncStatus`](#syncstatus) values.

* __downloadProgressCallback__ *((progress: DownloadProgress) => void)* - Called periodically when an available update is being downloaded from the CodePush server. The method is called with a `DownloadProgress` object, which contains the following two properties:
    * __totalBytes__ *(Number)* - The total number of bytes expected to be received for this update package
    * __receivedBytes__ *(Number)* - The number of bytes downloaded thus far.

Example Usage:

```javascript
// Prompt the user when an update is available
// and then display a "downloading" modal 
codePush.sync({ updateDialog: true }, (status) => {
    switch (status) {
        case CodePush.SyncStatus.DOWNLOADING_PACKAGE:
            // Show "downloading" modal
            break;
        case CodePush.SyncStatus.INSTALLING_UPDATE:
            // Hide "downloading" modal
            break;
    }
});
```

This method returns a `Promise` which is resolved to a `SyncStatus` code that indicates why the `sync` call succeeded. This code can be one of the following `SyncStatus` values:

* __CodePush.SyncStatus.UP_TO_DATE__ *(4)* - The app is up-to-date with the CodePush server.
* __CodePush.SyncStatus.UPDATE_IGNORED__ *(5)* - The app had an optional update which the end-user chose to ignore. (This is only applicable when the `updateDialog` is used)
* __CodePush.SyncStatus.UPDATE_INSTALLED__ *(6)* - The update has been installed and will be run either immediately after the `syncStatusChangedCallback` function returns or the next time the app resumes/restarts, depending on the `InstallMode` specified in `SyncOptions`.

If the update check and/or the subsequent download fails for any reason, the `Promise` object returned by `sync` will be rejected with the reason.

The `sync` method can be called anywhere you'd like to check for an update. That could be in the `componentWillMount` lifecycle event of your root component, the onPress handler of a `<TouchableHighlight>` component, in the callback of a periodic timer, or whatever else makes sense for your needs. Just like the `checkForUpdate` method, it will perform the network request to check for an update in the background, so it won't impact your UI thread and/or JavaScript thread's responsiveness.

### Package objects

The `checkForUpdate` and `getCurrentPackage` methods return promises, that when resolved, provide acces to "package" objects. The package represents your code update as well as any extra metadata (e.g. description, mandatory?). The CodePush API has the distinction between the following types of packages:

* [LocalPackage](#localpackage): Represents a downloaded update package that is either already running, or has been installed and is pending an app restart.
* [RemotePackage](#remotepackage): Represents an available update on the CodePush server that hasn't been downloaded yet.

#### LocalPackage

Contains details about an update that has been downloaded locally or already installed. You can get a reference to an instance of this object either by calling the module-level `getCurrentPackage` method, or as the value of the promise returned by the `RemotePackage.download` method.

##### Properties
- __appVersion__: The app binary version that this update is dependent on. This is the value that was specified via the `appStoreVersion` parameter when calling the CLI's `release` command. *(String)*
- __deploymentKey__: The deployment key that was used to originally download this update. *(String)*
- __description__: The description of the update. This is the same value that you specified in the CLI when you released the update. *(String)*
- __failedInstall__: Indicates whether this update has been previously installed but was rolled back. The `sync` method will automatically ignore updates which have previously failed, so you only need to worry about this property if using `checkForUpdate`. *(Boolean)*
- __isFirstRun__: Indicates whether this is the first time the update has been run after being installed. This is useful for determining whether you would like to show a "What's New?" UI to the end-user after installing an update. *(Boolean)*
- __isMandatory__: Indicates whether the update is considered mandatory.  This is the value that was specified in the CLI when the update was released. *(Boolean)*
- __label__: The internal label automatically given to the update by the CodePush server. This value uniquely identifies the update within it's deployment. *(String)*
- __packageHash__: The SHA hash value of the update. *(String)*
- __packageSize__: The size of the code contained within the update, in bytes. *(Number)*

##### Methods
- __install(rollbackTimeout: Number = 0, installMode: CodePush.InstallMode = CodePush.InstallMode.ON_NEXT_RESTART): Promise&lt;void&gt;__: Installs the update by saving it to the location on disk where the runtime expects to find the latest version of the app. The `installMode` parameter controls when the changes are actually presented to the end-user. The default value is to wait until the next app restart to display the changes, but you can refer to the [`InstallMode`](#installmode) enum reference for a description of the available options and what they do.
<br /><br />
If a value greater than zero is provided to the `rollbackTimeout` parameter, the application will wait for the `notifyApplicationReady` method to be called for the given number of milliseconds before considering the update failed and automatically rolling back to the previous version.
<br /><br />
*Note: The "rollback timer" doesn't start until the update has actually become active. If the `installMode` is `IMMEDIATE`, then the rollback timer will also start immediately. However, if the `installMode` is `UPDATE_ON_RESTART` or `UPDATE_ON_RESUME`, then the rollback timer will start the next time the app starts or resumes, not at the point that you called `install`.*

#### RemotePackage

Contains details about an update that is available for download from the CodePush server. You get a reference to an instance of this object by calling the `checkForUpdate` method when an update is available. If you are using the `sync` API, you don't need to worry about the `RemotePackage`, since it will handle the download and installation process automatically for you.

##### Properties

The `RemotePackage` inherits all of the same properties as the `LocalPackage`, but includes one additional one:

- __downloadUrl__: The URL at which the package is available for download. This property is only needed for advanced usage, since the `download` method will automatically handle the acquisition of updates for you. *(String)*

##### Methods
- __download(downloadProgressCallback?: Function): Promise&lt;LocalPackage&gt;__: Downloads the available update from the CodePush service. If a `downloadProgressCallback` is specified, it will be called periodically with a `DownloadProgress` object (`{ totalBytes: Number, receivedBytes: Number }`) that reports the progress of the download until it completes. Returns a Promise that resolves with the `LocalPackage`.

### Enums

The CodePush API includes the following enums which can be used to customize the update experience:

#### InstallMode

This enum specified when you would like an installed update to actually be applied, and can be passed to either the `sync` or `LocalPackage.install` methods. It includes the following values:
* __CodePush.InstallMode.IMMEDIATE__ *(0)* - Indicates that you want to install the update and restart the app immediately. This value is appropriate for debugging scenarios as well as when displaying an update prompt to the user, since they would expect to see the changes immediately after accepting the installation.

* __CodePush.InstallMode.ON_NEXT_RESTART__ *(1)* - Indicates that you want to install the update, but not forcibly restart the app. When the app is "naturally" restarted (due the OS or end-user killing it), the update will be seamlessly picked up. This value is appropriate when performing silent updates, since it would likely be disruptive to the end-user if the app suddenly restarted out of nowhere, since they wouldn't have realized an update was even downloaded. This is the default mode used for both the `sync` and `LocalPackage.install` methods.
 
* __CodePush.InstallMode.ON_NEXT_RESUME__ *(2)* - Indicates that you want to install the update, but don't want to restart the app until the next time the end-user resumes it from the background. This way, you don't disrupt their current session, but you can get the update in front of them sooner then having to wait for the next natural restart. This value is appropriate for silent installs that can be applied on resume in a non-invasive way.

#### SyncStatus

This enum is provided to the `syncStatusChangedCallback` function that can be passed to the `sync` method, in order to hook into the overall update process. It includes the following values:

* __CodePush.SyncStatus.CHECKING_FOR_UPDATE__ *(0)* - The CodePush server is being queried for an update.
* __CodePush.SyncStatus.AWAITING_USER_ACTION__ *(1)* - An update is available, and a confirmation dialog was shown to the end-user. (This is only applicable when the `updateDialog` is used)
* __CodePush.SyncStatus.DOWNLOADING_PACKAGE__ *(2)* - An available update is being downloaded from the CodePush server.
* __CodePush.SyncStatus.INSTALLING_UPDATE__ *(3)* - An available update was downloaded and is about to be installed.
* __CodePush.SyncStatus.UP_TO_DATE__ *(4)* - The app is fully up-to-date with the configured deployment.
* __CodePush.SyncStatus.UPDATE_IGNORED__ *(5)* - The app has an optional update, which the end-user chose to ignore. (This is only applicable when the `updateDialog` is used)
* __CodePush.SyncStatus.UPDATE_INSTALLED__ *(6)* - An available update has been installed and will be run either immediately after the `syncStatusChangedCallback` function returns or the next time the app resumes/restarts, depending on the `InstallMode` specified in `SyncOptions`.
* __CodePush.SyncStatus.UNKNOWN_ERROR__ *(-1)* - The sync operation encountered an unknown error. 