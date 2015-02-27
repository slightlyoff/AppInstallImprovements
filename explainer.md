<h2>App Install Extensions Explained</h2>

## What's All This About?

Modern web applications can engage users as deeply as native/desktop apps. The W3C Manifest Specification provides a route for enabling UAs to offer "installation" of web content. Despite the promise of Manifests and Service Workers, many aspects of appyness are currently missing from the web platform's de-facto OS integrations.

In particular, web content that is treated by as a top-level app by an OS is still missing:

  - The ability for apps to understand that they've been installed
  - API surface area in Service Workers to direct navigation requests to applications and detect that some `Client` instances are launched in "app mode"
  - Control from Service Workers regarding requests that should be launched in "regular" tabs (e.g., how does an App decide which navigations are "outside" the app?)
  - Ways for content to detect that it has been launched in such a mode (e.g., feature-detectable flag or media query)
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
      "location": <play store url>,
    },
    {
      "platform" "ios",
      "location": <ios store url>,
    },
    {
      "platform": "web"
    },
    {
      "location": ...,
      "platform": ...
    },
  ],
  "icons": [ {
    "src": "images/touch/google-touch-icon-144x144.png",
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

Having weighed the concerns about spammyness of APIs that would implicitly or explicitly force a prompt to appear for users, it seems the _most_ reasonable way for a site to know if it (or a related application) is being installed is for the system to inform it that such a prompt is being offered. An event in a document may be a reasonable way to model this:

```js
window.addEventListener("beforeinstallprompt", function(e) {
  // log out the platform being prompted for
  console.log(e.platform); // e.g., "web", "android", "windows", etc.
  e.userChoice.then(function(outcome) {
    console.log(outcome); // either "installed", "dismissed", etc.
  }, handleError);
});
```

The rejection handler is used to communicate exceptions (out of disk, etc.).

A user choosing to dismiss the prompt without installing is provided to the success handler via the outcome value. This can be one of:

  - `"installed"`: indicates that the installation process was successfully started
  - `"dismissed"`: indicates that the user dismissed the prompt, rejecting installation

We further hope that such an API can evolve to handle _delaying_ display of a prompt -- for instance to avoid interrupting users at an inopportune point in a workflow. Such an "install later" capability can be modeled using the same event:

```js
var isTooSoon = true;
window.addEventListener("beforeinstallprompt", function(e) {
  if (isTooSoon) {
    e.preventDefault(); // Prevents prompt display
    // Prompt later instead:
    setTimeout(function() {
      isTooSoon = false;
      window.dispatchEvent(e); // Shows prompt
    }, 10000);
  }

  // The event was re-dispatched in response to our request
  // ...
});
```

<!--
### Understanding Launch Mode



### App Extent & Service Worker Interaction

-->

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
    "src": "images/touch/google-touch-icon-144x144.png",
    "sizes": "144x144",
    "type": "image/png",
    "density": "3.0"
  }],
  "display": "standalone"
}
```

To be surfaced to users, the URLs in question must (of course) match the Service Worker scope for the application.

<!--
### Strawman: App Install Criteria

-->
