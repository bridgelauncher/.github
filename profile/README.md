# Bridge Launcher

An Android launcher that is a middleman between your HTML, JS and CSS and the Android system.  
Requires Android 8 (API level 24).

## BRIDGE IS NOT A REGULAR LAUNCHER

Do not expect Bridge to contain a regular homescreen with pages you can drag apps and widgets onto, or a highly customizable app drawer. If you're looking for a regular launcher, check out this [Launcher Comparison Table](https://bit.ly/TheLaunchersTable2) by Grabster.

Bridge aims to enable you to create your own launcher using web technologies that are more accessible than regular Android development. You can of course also use it to load a project made by someone else.

## System structure

The core of Bridge is the [Bridge Launcher](https://github.com/bridgelauncher/launcher) Android app. This app replaces your homescreen with a WebView housing a HTML file of your choosing.
Additionaly, the app exposes endpoints allowing you to load a list of apps installed on the device and their icons (for example via `fetch()`).

Javascript code running inside the WebView is given access to a global variable called `Bridge`, through which it can interact with Android system features and the launcher app. Examples include obtaining the height of the status bar, launching apps, expanding the notification shade and plenty more.

Additionaly, the launcher app notifies Javascript of events (such as apps being installed/uninstalled and permission state changes) by calling the global variable `onBridgeEvent` (if it is set to a function).

To allow you to have a WIP project without losing the ability to launch apps, Bridge provides a simple, searchable app drawer that you can also open from the JS API.

### Features

- Load any HTML, JS and CSS that can run in the native Android WebView (usually chromium-based) as your Android homescreen,
- Show the system wallpaper behind the WebView (`<html>` and `<body>` must be transparent).  
  Works with live wallpapers (yes, KLWP too!),
- Notable features of the Javascript API:
    - Launch apps, request uninstallation, open app info in system settings,
    - Interact with the system wallpaper: set pages, set scroll position, send taps to live wallpapers,
    - Set status & navigation bar appearance (hide, light foreground, dark foreground),
    - Change the system night mode (requires granting `android.permission.WRITE_SECURE_SETTINGS` via `adb`),
    - Lock (turn off) the screen (requires granting system admin access to Bridge Launcher and allowing projects to lock the screen in settings),

- Development features:
    - Export apps installed on the device and their icons to use during development,
    - [TypeScript types](https://github.com/bridgelauncher/api) for the JS API ([also available via `npm`](https://www.npmjs.com/package/@bridgelauncher/api)),
    - [BridgeMock](https://github.com/bridgelauncher/api-mock) for mocking the Bridge API for development purposes ([also available via `npm`](https://www.npmjs.com/package/@bridgelauncher/api-mock)).

### Limitations

- DIY. The only built-in component is a simple app drawer.
- Icon pack support is not implemented yet,
- No widget support,
- Unknown battery life impact,
- Update scope & frequency will be heavily limited by my free time.

## Loading a project

1. [Download and install the launcher app](https://github.com/bridgelauncher/launcher/releases),
2. Open settings via the prompt in the middle of the screen or from a menu that appears after tapping the Bridge button (on the bottom right),
3. Tap "Change" in the "Active project" card,
4. Grant necessary storage permissions,
5. Navigate to the folder containing the `index.html` file you wish to load,
6. Tap "Use this folder",
7. Return to the homescreen. The project should load.

## Creating a project

### Choosing a framework

I strongly recommend using a front-end framework or at the very least using Typescript over vanilla Javascript. Keep in mind that the framework you choose must be capable of running only in the browser, as Bridge does not provide a way to run server-side code. 

I can personally recommend [Vue](https://vuejs.org/), which is what I built the [example project](https://github.com/bridgelauncher/api-tester) with.


### Installing types

TypeScript types for the JS API are available [on GitHub](https://github.com/bridgelauncher/api) and [via `npm`](https://www.npmjs.com/package/@bridgelauncher/api).

#### npm
```
npm i @bridgelauncher/api
```
#### direct
Place the [Bridge.d.ts](https://github.com/bridgelauncher/api/blob/master/Bridge.d.ts) file in the root directory of your Typescript code.

### Calling the API

The Bridge API is accessible through the global variable `Bridge`:

```ts
Bridge.showToast('Hello, world!');
```

Currently, documentation is only available via JSDoc comments on the types. After you install types, editors like [Visual Studio Code](https://code.visualstudio.com/) will show you available functions with their descriptions after you enter `Bridge.` and press `Ctrl + Space`.

You can open the [Bridge.d.ts](https://github.com/bridgelauncher/api/blob/master/Bridge.d.ts) file on GitHub or by going to definition on any Bridge type to see available API methods and events.

### Loading apps and icons

Bridge serves the current list of apps as JSON from an endpoint accessible via `Bridge.getAppsURL()`. Example using `fetch()`:
```ts
fetch(Bridge.getAppsEndpoint())
    .then(resp => resp.json() as BridgeGetAppsResponse)
    .then(resp => {
        // do something with the list of apps
        resp.apps 
    })
```

App icons are served similarly, but the endpoint expects a package name of the app you want to get the icon of as a query parameter. This enpoint is accessible via `Bridge.getDefaultAppIconURL([package name])`. You can pass the URL as the `src` parameter for an `<img>` tag, which saves you having to process the returned image:

```ts
const img = document.createElement('img')
// let the img handle the loading
img.src = Bridge.getDefaultAppIconUrl('com.tored.bridgelauncher')
document.body.appendChild(img);
```

### Listening for events

If the global variable `onBridgeEvent` is set to a function, Bridge will call that function whenever an event occurs. The first argument passed will be name of the event, and the second will be an object if the event has arguments, or `undefined` if the event does not have arguments.

The [Bridge.d.ts](https://github.com/bridgelauncher/api/blob/master/Bridge.d.ts) file contains a `BridgeEventMap` interface, which is a map of event names to their argument types.

#### Single listener

The simplest way to handle events is to do everything directly inside the listener function:

```ts
window.onBridgeEvent = (name, args) => {
    // could use a switch() statement instead
    if (name === 'appRemoved') {        // autocomplete will help with event names
        console.log(args.packageName);  // args will be strongly typed
    } else if (...) {
        ...
    }
}
```

#### Mutliple listeners

You can create your own system to dispatch the events received via `onBridgeEvent` to multiple listeners. Here is a simple example:

```ts
// set that will hold all registered event listeners
const listeners = new Set<BridgeEventListener>();

// upon receiving an event, forward it to all listeners
window.onBridgeEvent = (...event: BridgeEventListenerArgs) => {
    listeners.forEach(l => l(...event));
}

// adding a listener later in the code
listeners.add((name, args) => {
    if (name === 'appRemoved') {        // autocomplete will help with event names
        console.log(args.packageName)   // args will be strongly typed
    }
})
```

Please note the arguments being specified as `(...event: BridgeEventListenerArgs)` and later used as `...event` - this is to maintain a connection between the event name and its args.

```ts
window.onBridgeEvent = (name, args) => {
    // typescript won't maintain a connection between the event name and the type of args,
    // which will result in an error on this line
    listeners.forEach(l => l(name, args)); 
}
```



### Mocking the Bridge API

Testing every change to your project in the launcher would get incredibly tedious very quickly. Mocking the Bridge API allows you to test the project in whatever development environment you are using. 

To mock the Bridge API, assign a JS object implementing the `JSToAndroidAPI` interface to the `window.Bridge` global variable.

#### Default mock

A simple mock is available [via `npm`](https://www.npmjs.com/package/@bridgelauncher/api-mock):

```
npm i @bridgelauncher/api-mock
```

The source code is available in a [GitHub repo](https://github.com/bridgelauncher/api-mock).

Usage:

```ts
import { BridgeMock } from '@bridgelauncher/api-mock';

// only mock when not injected by the launcher
// make sure this runs before any code that uses the API!
if (!window.Bridge) window.Bridge = new BridgeMock({
    // check autocomplete for available configuration options
    logWallpaperEvents: false,
    ...
});

// access mock-specific properties from, for example, a dev panel
if (window.Bridge instanceof BridgeMock) 
    window.Bridge.config.logWallpaperEvents = true;
```

#### App list & icons

The "Development" section of the Bridge Launcher settings screen includes an option to export the list of apps and their icons to a folder on your device. The folder will contain a `.json` file containing the same JSON you'd obtain from `fetch(Bridge.getAppsUrl())` and a subfolder with app icon PNG files. You can then transfer this folder to your development environment and serve it, so it can be obtained via `fetch()`.

For example, if your front-end framework provides a directory for public files to be served as-is, you can put the folder there. In Vue, this folder is called `public`. You can then configure the API mock to return URLs pointing to the served files. Here's how to do it with the default mock:

```ts
if (!window.Bridge) window.Bridge = new BridgeMock({
    appsUrl: '/mock/apps.json',
    makeGetDefaultIconUrl: (packageName: string) => `/mock/icons/default/${packageName}.png`,
    ...
});
```

Please note that this assumes you did not hard-code the URLs and are using API methods like `Bridge.getAppsUrl()` to obtain them when needed.


#### Customizing the mock

- Change the configuration passed to the `BridgeMock` class,
- Create your own class that extends `BridgeMock` and override public or protected methods,
- Get the [source code](https://github.com/bridgelauncher/api-mock) and modify it to your needs.



## About

Designed & written by Tored.  
Bridge Launcher is my attempt at making launcher development approachable by reducing dealing with Android to using a simple API. 

- [Discord server](https://discord.gg/Tv23aZrVb8) - I blogpost as I go
- [theothertored@gmail.com](mailto:theothertored@gmail.com) - Contact email
