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
    - Lock (turn off) the screen (requires enabling an accessibility service and allowing projects to lock the screen in settings),

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

### Changing system night mode

To change system night mode, Bridge must be granted either `android.permission.WRITE_SECURE_SETTINGS` or `android.permission.MODIFY_DAY_NIGHT_MODE`.
The former can be granted via adb, the latter can't be granted without serious effort at the time of writing this message, because (for unknown reasons) it is restricted to system apps only.

[How to install ADB on Windows, macOS, and Linux | XDA Developers](https://www.xda-developers.com/install-adb-windows-macos-linux/)

```
adb shell pm grant com.tored.bridgelauncher android.permission.WRITE_SECURE_SETTINGS
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


### Tips & tricks

#### Scrolling page by page instead of continously

I highly recommend familiarizing yourself with CSS scroll snapping. This will give your scrolling a feeling close to or identical with native scrolling, without Javascript (which can be laggy for scrolling, especially on mobile!).

- [Practical CSS Scroll Snapping | CSS Tricks](https://css-tricks.com/practical-css-scroll-snapping/)
- [CSS scroll snap - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_scroll_snap)

#### Converting a number of pages to wallpaper offset steps

To inform Android about the number of pages you want the wallpaper to have, horizontal and vertical "wallpaper offset steps" need to be provided. The steps are numbers between 0 and 1. To translate from pages to steps, think about the `step` like this:

- `0` means the user is on the 1st page
- `1` means the user is on the last page
- `0 + step` means the user is on the 2nd page
- `0 + step + step` means the user is on the 3rd page
- etc.

For example, if there are 2 pages, the 2nd page = the last page, so `0 + step = 1`, therefore `step = 1`.  
If there are 3 pages, the 3rd `0 + step` is the second page and `0 + step + step` is the last page.  
Simplyfing, `step * 2 = 1`, therefore `step = 0.5`.

The formula is `1 / (page count - 1)` when `page count > 1` and `0` when `page count = 1` (to avoid division by 0). 

Example page count to offset steps conversion:

```js
const xPages = 3; 
const yPages = 2;
const p2o = p => p > 1 ? 1 / (p - 1) : 0;
Bridge.setWallpaperOffsetSteps(p2o(xPages), p2o(yPages));
```

#### Converting a scroll position to wallpaper offsets

To inform Android about the current position the wallpaper should be scrolled to, "wallpaper offsets" need to be provided. The offsets are numbers between 0 and 1.

- `0` means the user is on the 1st page
- `1` means the user is on the last page
- `0.5` means the user is halfway between the 1st and last page. 
    - If you have 2 pages, this means the user is halfway between the 1st and 2nd page. 
    - If you have 3 pages, this means the user is on the 2nd page. 
    - etc.

Your scrolling will most likely be done using a HTML element (`el`) with `overflow: scroll` or `auto`.

In JS, `el.scrollWidth` and `el.scrollHeight` can be used to get the total height of the element's contents. These numbers include the element's own height, plus the height of any overflowing content. You can imagine the element as a small rectangle and its contents as a bigger rectangle that moves when the element is scrolled, but never leaves an empty spot between itself and any edge of the small rectangle.

`el.scrollLeft` and `el.scrollTop` can be used to get the current horizontal and vertical scroll offset of the element respectively. The maximum value for `el.scrollLeft` is `el.scrollWidth - el.clientWidth`. This is because scrolling an element to the end means the content's end lines up with the element's end and there's one "screen" worth of content still visible.

The X offset is `0` when `el.scrollLeft = 0` and `1` when `el.scrollLeft = el.scrollWidth - el.clientWidth`.
Therefore, X offset = `el.scrollLeft / (el.scrollWidth - el.clientWidth)`, or 0 if `el.scrollWidth - el.clientWidth = 0`.

Example window scroll position to wallpaper offsets conversion:

```js
window.addEventListener('scroll', ev => {
     requestAnimationFrame(() => {
         const xMaxScroll = window.scrollWidth - window.innerWidth;
         const yMaxScroll = window.scrollHeight - window.innerHeight;
         const xScroll = window.scrollLeft;
         const yScroll = window.scrollTop;
         Bridge.setWallpaperOffsets(
            xMaxScroll === 0 ? 0 : xScroll / xMaxScroll, 
            yMaxScroll === 0 ? 0 : yScroll / yMaxScroll
        );
     });
});
```

#### Disable text selecting on long-press

By default, long-pressing text in a WebView selects it. To disable this behaviour, give the element you want to disable it for the CSS property `user-select: none`.

Putting `user-select: none` on an element does not disable selecting in inputs and textareas inside that element. Additionally, if the property is set to another value (like `auto`) for a descendant, that descendant will be selectable.

```css
html {
    user-select: none;
}

.selectable {
    user-select: auto;
}
```

#### Remove blue highlights appearing when tapping elements

The blue highlight can be disabled by giving an element the CSS property `-webkit-tap-highlight-color: transparent`. I highly recommend setting this on all elements:

```css
* {
    -webkit-tap-highlight-color: transparent;
}
```

#### Use the HTML template element instead of creating elements manually

[The Content Template Element | MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template)

```html
<template id="appListItemTemplate">
    <div class="app-list-item">
        <span class="label"></span>
        <span class="package-name"></span>
    </div>
</template>
```

```js
const template = document.getElementById('appListItemTemplate');
for (const app of apps)
{
    const item = template.content.firstElementChild.cloneNode(true);
    item.querySelector('.label').innerText = app.label;
    item.querySelector('.package-name').innerText = app.label;
    container.appendChild(item);
}
```

## About

Designed & written by Tored.  
Bridge Launcher is my attempt at making launcher development approachable by reducing dealing with Android to using a simple API. 

- [Discord server](https://discord.gg/Tv23aZrVb8) - I blogpost as I go
- [theothertored@gmail.com](mailto:theothertored@gmail.com) - Contact email
