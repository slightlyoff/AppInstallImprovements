<h2>App Install Extensions Explained</h2>

## What's All This About?

Modern web applications can engage users as deeply as native/desktop apps. The W3C Manifest Specification provides a route for enabling UAs to offer "installation" of web content. Despite the promise of Manifests and Service Workers, many aspects of appyness are currently missing from the web platform's de-facto OS integrations.

In particular, web content that is treated by as a top-level app by an OS is still missing:

  - The ability for apps to understand that they've been installed
  - API surface area in Service Workers to direct navigation requests to applications and detect that some `Client` instances are launched in "app mode"
  - Control from Service Workers regarding requests that should be launched in "regular" tabs (e.g., how does an App decide which navigations are "outside" the app?)
  - Agreement between UAs about what combination of URL scopes, Service Worker registrations, and manifest locations constitute "an app"
  - Metadata about how to offer associated "native" apps to users

This document explores each of those challenges, proposes sketches for solutions, and explores challenges.

This is presented in a single repository to avoid peppering the various existing efforts (Manifests, Service Workers, DOM Events) with seemingly unrelated proposals, although we expect that the outcome will be changes to each of those specs and not a new spec/effort.

### Offering Related Applications

Some platforms support non-web-applications; e.g.

  - [Windows Apps](https://dev.windows.com/en-us)
  - [Android Apps](http://developer.android.com/index.html)
  - [iOS Apps](https://developer.apple.com/library/ios/referencelibrary/GettingStarted/RoadMapiOS/)
  - [Chrome Apps](https://developer.chrome.com/apps/first_app)
  - [Firefox OS Packaged Apps](https://developer.mozilla.org/en-US/Marketplace/Options/Packaged_apps)

We observe that many developers rely on converting users to these "native" app platforms (for reasons that are contested). _That_ it happens pervasively is enough reason for some vendors to [invent](https://msdn.microsoft.com/en-us/library/ie/hh781489(v=vs.85).aspx) [one-off meta-tag-based solutions](https://developer.apple.com/library/ios/documentation/AppleApplications/Reference/SafariWebContent/PromotingAppswithAppBanners/PromotingAppswithAppBanners.html).

These system-provided banners are beneficial to the overall web ecosystem as they can potentially reduce the amount of "slam door" interstitial prompting that websites do to users.

Downsides to this proliferation of meta-tag based solutions include:

 - Placement in the `<head>` of documents, which is incredibly sensitive from a page performance standpoint
 - Repetition per-native platform. This exacerbates the issues related to precious first-packet bytes in much the same way that the sea-of-icon-tags problem did.

The [solution](https://developer.chrome.com/multidevice/android/installtohomescreen) for icons has been a [uniform manifest](https://w3c.github.io/manifest/).

We propose unifying the manifest to support native app install banners. Given a page:

```html
<!DOCTYPE html>
<!-- https://example.com/index.html -->
<html>
  <head>
    <link rel="manifest" href="/manifest.json">
    <script defer>
      if ('serviceWorker' in navigator) {
        navigator.serviceWorker.register("/sw.js").catch(
          /* ... */
        );
      }
    </script>
  </head>
  <body> ... </body>
</html>
```

We propose manifest extensions which enable the UA to decide which application to offer:

```json
{
  "name": "Google I/O 2015",
  "short_name": "I/O 2015",
  "start_url": "index.html",
  "related_applications": [
    {
      "platform": "android",
      "location": "<play store url>",
    },
    {
      "platform": "ios",
      "location": "<ios store url>",
    },
    {
      "platform": "web"
    },
    {
      "location": "...",
      "platform": "..."
    },
  ],
  "icons": [ {
    "src": "images/icon-144x144.png",
    "sizes": "144x144",
    "type": "image/png",
    "density": "3.0"
  }],
  "display": "standalone"
}
```

If no `"related_applications"` list is provided and the site meets the installability criteria (see below), the web version can be assumed to be the preference. No `location` is required for `"web"` entry as the manifest itself can be considered to describe the web app and the `"start_url"` entry describes the Apps primary entrypoint.

In some instances, the web version might not be installable (e.g. because of system restrictions on the UA in creating the necessary UI and control surface area). Further, it's obvious that there will be instances where an iOS or Android application won't be available. To accommodate all of these situations, the list is ordered based on developer preference. If and `"android"` or `"iOS"` entry occurs before the `"web"` (or default `"web"`) entry and is appropriate for the UA to offer, an "install banner" might be shown, using the information provided in the manifest to bootstrap the offer process.

If a `"web"` version is not explicitly provided, it is assumed to be the last (lowest priority, default) entry.

This system could scale better, be less intrusive, and be less confusing for developers dealing with multiple application than the current hodge-podge of app install hints and expensive header entries.

### Controlling Installation

The [Manifest format](https://w3c.github.io/manifest/) doesn't provide mechanisms for detecting (and responding to) the app installation process or for understanding, once launched, that an app is in "app mode".

Understanding what (if any) API to provide related to prompting is complicated. For instance, Chrome 42 is going to show Web App install prompts when a conforming app is visited multiple times in the space of a few weeks and _never_ on the first visit to a site. Such criteria make explicit requesting API untennable.

Having weighed the concerns about spammyness of APIs that would implicitly or explicitly force a prompt to appear for users, it seems the _most_ reasonable way for a site to know if it (or a related application) is eligible for installation is for the system to inform it that such a prompt is being offered. An event in a document may be a reasonable way to model this:

```js
window.addEventListener("beforeinstallprompt", function(e) {
  // log the platforms provided as options in an install prompt
  console.log(e.platforms); // e.g., ["web", "android", "windows"]
  e.userChoice.then(function(platform, outcome) {
    console.log(platform); // the platform of the app the user took an action on
    console.log(outcome); // either "installed", "dismissed", etc.
  }, handleError);
});
```

The rejection handler is used to communicate exceptions (out of disk, etc.).

A user choosing to dismiss the prompt without installing is provided to the success handler via the outcome value. This can be one of:

  - `"installed"`: indicates that the installation process was successfully started
  - `"dismissed"`: indicates that the user dismissed the prompt, rejecting installation

The event and promise together allow websites to record important analytics such as click through and conversion rates.

We further hope that such an API can evolve to handle _delaying_ display of a prompt -- for instance to avoid interrupting users at an inopportune point in a workflow or to allow sites to show in-page explanation of the feature before the prompt is displayed. Such an "install later" capability can be modeled using the same event:

```js
var isTooSoon = true;
window.addEventListener("beforeinstallprompt", function(e) {
  if (isTooSoon) {
    e.preventDefault(); // Prevents prompt display
    // Prompt later instead:
    setTimeout(function() {
      isTooSoon = false;
      e.prompt(); // Throws if called more than once or default not prevented
    }, 10000);
  }

  // The event was re-dispatched in response to our request
  // ...
});
```

### App Extent & Service Worker Interaction

Many application platforms provide for [intercepting a subset of URL navigations](https://developer.chrome.com/apps/manifest/url_handlers) or [system actions that lead to navigations](http://developer.android.com/guide/components/intents-filters.html). This has some overlap with [registering protocol handlers](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/registerProtocolHandler), but we can safely consider `http://` and `https://` to be unique.

Web applications do not, today, contain a single view of what set of URLs specify them as an "app". One obvious option is to consider the set of URLs which a [Service Worker](http://www.html5rocks.com/en/tutorials/service-worker/introduction/) is responsible for handling to be "the app", but applications may want ways to suggest that they don't "own" the navigation and that it should be handled in the default way (e.g., loading a document in a tab instead of navigating inside an application window). Integration of Service Workers into Manifests is an un-yet [unresolved issue](https://github.com/w3c/manifest/issues/161), meaning the easiest answer ("just use whatever the registration in the manifest indicates") isn't sufficient either.

We propose API to allow an app to dispose of inbound navigations in whatever way it sees fit:

```js
// https://example.com/sw.js
onfetch = function(e) {
  // Check to see if it's a request for end-user documentation
  var requestUrl = new URL(request.url);
  if (requestUrl.pathname.indexOf("/docs") == 0) {
    // Send to a regular tab
    e.default({ target: "_new" });
  }
  e.waitUntil(clients.matchAll({ type: "window" }).then(
    function(windows) {
      // Check to see if we've already got an open app window:
      var window;
      windows.forEach(function(w) {
        if (w.url == e.request.url) {
          window = w;
        }
      });
      if (w) {
        return w.focus();
      } else {
        // Straw-man syntax for setting a preferred display type
        e.default({ display: "standalone" });
      }
    }
  ));
};
```

No affordance is provided for _outbound_ link mediation as, in general, this is the fast-path to creating violated user expectations and poor experiences (e.g., "in app browsers" that totally balls up TLS, etc.).

### Strawman: Non-App Control Surfaces

We note that many "native" app platforms provide alternative UI front-ends; e.g. [Android Widgets](http://developer.android.com/design/patterns/widgets.html), [Windows Live Tiles](https://msdn.microsoft.com/en-us/library/windows/apps/hh202948%28v=vs.105%29.aspx), and [iOS App Extensions](https://developer.apple.com/library/mac/documentation/General/Conceptual/ExtensibilityPG/NotificationCenter.html). Each of these system creates alternative entry points into application functionality displayed in non-primary App UI contexts (it can be argued that Notifications are similar, but for now we ignore them).

The question arises: what if those UI surfaces could be populated by web applications? Some systems allow developers to provide multiple widgets/tiles per application and allow users to select their placement. To accomodate this, we explore a parallel to `"start_url"` which indicates URLs for multiple alternative entry points:

```json
{
  "name": "Google I/O 2015",
  "short_name": "I/O 2015",
  "start_url": "index.html",
  "widgets": [
    {
      "location": "tile.html",
      "type": "tile"
    },
    {
      "location": "widget.html",
      "type": "widget"
    },
    {
      "location": "lock_screen_control.html",
      "type": "lockscreen"
    },
  ],
  "icons": [ {
    "src": "images/icon-144x144.png",
    "sizes": "144x144",
    "type": "image/png",
    "density": "3.0"
  }],
  "display": "standalone"
}
```

To be surfaced to users, the URLs in question must (of course) match the Service Worker scope for the application.

### Brand Colors and Launch Icons

Many application platforms provide the ability for an app to show some sort of startup screen. Web apps tend to be launched in tabs today -- either as new sub-processes which can be cheaply cloned or in-process -- in an environment where initial rendering can safely considered to be "cheap".

Web apps that launch from the homescreen or start menu may not have such advantages. They may, perhaps, require startup of a browser and renderer process, initialization of GPU contexts, script context creation and application parsing, etc.

Web applications that must pay the perceptual price for browser startup when the user taps on their icon are therefore at a UX disadvantage to their "native" counterparts because they cannot control what first gets painted in those initial milliseconds -- even if the content from which they will eventually render is entirely local (e.g., via Service Worker).

Certain native platforms provide hooks for presenting [images](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/MobileHIG/LaunchImages.html) or otherwise provide very-early launch control over paint. iOS has even extended the "launch image" concept [to the web](https://developer.apple.com/library/mac/documentation/AppleApplications/Reference/SafariWebContent/ConfiguringWebApplications/ConfiguringWebApplications.html).

It is jarring to present a white (or default-color) screen on tap only to have the application draw a different full-screen `background-color` half a second later. To prevent this, we think it useful to allow the Manifest to declare optional colors and images to use at launch.

There is precedent. Configuration is possible of the status-bar area of mobile OSes. iOS provides this capability to the web using the [`apple-mobile-web-app-status-bar-style`](https://developer.apple.com/library/safari/documentation/AppleApplications/Reference/SafariHTMLRef/Articles/MetaTags.html). In addition, the shipping [`theme-color`](https://github.com/whatwg/meta-theme-color), [`brand-color`](https://groups.google.com/a/chromium.org/d/msg/blink-dev/nzRY-h_-_ig/KR3XWn73tDoJ) and [`msapplication-navbutton-color`](https://msdn.microsoft.com/en-us/library/ie/gg491732%28v=vs.85%29.aspx) meta tags indicate significant interest among vendors in providing at some configurability. Indeed, there is a [related issue open on the Manifest spec today](https://github.com/w3c/manifest/issues/225).

We propose `"theme_color"` and `"start_image"` extensions:

```json
{
  "name": "Google I/O 2015",
  "short_name": "I/O 2015",
  "start_url": "index.html",
  "icons": [ {
    "src": "images/icon-144x144.png",
    "sizes": "144x144",
    "type": "image/png",
    "density": "3.0"
  }],
  "display": "standalone",
  "theme_color": "#db5945",
  "start_image": "images/start-image.png"
}
```

This proposal assumes that the `"theme_color"` entry will be used to both configure initial full-bleed painting for the app as well as configure icon tray color. It is not yet clear to us if there is a desire to configure them independently. If there is, we might imagine a separate `"start_background_color"` property, although the name `"theme_color"` seems to alleviate this concern somewhat.

### Strawman: App Install Criteria

The Chrome team has identified a list of criteria that we equate with "appiness". As runtimes begin to allow and offer [installability for web applications](http://updates.html5rocks.com/2014/11/Support-for-installable-web-apps-with-webapp-manifest-in-chrome-38-for-Android), we have an interest in ensuring that users do not experience poor-quality sites in their "apps" list. Quality is subjective, but we're interested in a subset of quality criteria that can be automated and broadly communicated but which also correlate to maintained and modern, appy content.

The following straw-man criteria seem to fit that bill:

  - Served at a Secure Origin
    - Functionally this means requiring TLS
  - Registration of a Service Worker which
    - Does not contain parse errors
    - Controls at least the current document and the `start_url` of the manifest (see below)
    - Registers a non-empty onfetch event handler
  - The presence of a Web Manifest which includes at least the following fields
    - `short_name` and `name`:
      - The descriptive names to use for display, e.g. under a homescreen icon and hover text
    - `icons`:
      - A non-empty list with at least one icon in with enough density to look good on modern displays. For now that's at least one icon in PNG format that is at least 144x144px.
    - `start_url`:
      - The location to launch the app at if it isn't navigated to some other way. In essence, a "homescreen" location.
      - The `start_url` must be within the registration scope of the Service Worker for the application.
    - `service_worker`:
      - Optional if the installing site already has a Service Worker that matches both the installing page and `start_url`. A property bag that lists the location and scope of the Service Worker to use for the application.
  - Responsive breakpoints & viewport meta tags
    - The viewport declaration must be appropriate for mobile design, e.g.:
      ```<meta name="viewport" content="width=device-width, initial-scale=1">```
    - Responsive CSS breakpoints are the most at-risk criteria as it might not be appropriate, e.g., for games

Runtimes may add additional criteria when deciding if and when to offer appy features (including offers of installation). For instance, Chrome will not prompt users to "keep" applications unless the user has visited multiple times in the past few weeks.

Similarly, we can envision improving the quality of "app" detection by implementing a server or client-side analysis of Service Workers to determine to what extent applications will respond meaningfully offline, above and beyond the simple presences of an `onfetch` handler.

#### The Rubric

##### Secure Origins

Good apps don't give your data to third parties without your knowledge. Our ability to trust your behavior as a full-screen application hinges on us already having some confidence that the removal of security indicators won't egregiously degrade the user experience.

Chrome reserves the right to show URL-bar + security indicators should the security posture of the app degrade; e.g. through the use of Mixed Content on a site that was previously served over entirely secure connections.

##### Service Workers

It isn't an app if it doesn't start when you tap.

That means all the time, which requires that the app at least boot up to an "offline" screen that's provided by the app and not the browser. Service Workers are the preferred way of providing offline functionality and are therefore required.

##### Web Manifests

We require enough metadata to ensure that we can provide meaningful UI to users who choose to install or "keep" the application in their top-level launcher UI surfaces.

##### Responsive Breakpoints & Viewport

The site must be making some effort to work well on mobile. Apps aren't good citizens unless they do the work to adapt to the viewport.

Browsers bend over backwards to adapt to legacy content, but for content we wish to offer to users as "first class", there shouldn't be any tap-to-zoom required. The app should simply work well.
