# Explainer: ServiceWorkerBypassFetchHandler

## Authors

-  [sisidovski](https://github.com/sisidovski)

## Participate

-  https://github.com/sisidovski/service-worker-bypass-fetch-handler/issues

## Introduction

ServiceWorker is a unique and primitive API. The API allows developers to intercept and control the resource fetch requests via "fetch" event handler. This enables web apps to be reliable even offline and give faster loading experiences by using its own CacheStorage. However, the cost of ServiceWorker is not cheap. As the worst case, it takes hundreds milliseconds to bootstrap the ServiceWorker itself and process fetch event handlers. Adoption-wise, we see [15%](https://chromestatus.com/metrics/feature/timeline/popularity/990) of all Chrome page loads are controlled by ServiceWorker.

To solve this issue, we'd like to introduce the ServiceWorker "fetch" fast-path, which is a way to bypass fetch handlers. Normally, the ServiceWorker activation is triggered before sending a request if the resource is in the ServiceWorker's scope. In other words, the bootstrap process always blocks the actual request regardless of whether the resource actually needs to be handled by handlers or not. But what if the fetch handlers do not effectively handle the resource and there is a way to tell the browser in advance which resources are bypassable? By enabling this feature, the request immediately happens without waiting for the ServiceWorker bootstrap and bypasses fetch handlers. The ServiceWorker bootstrap is started at the same time as the request happens, but it doesn't interrupt request/response in the fetch handler. By enabling this feature, the browser can hide the bootstrap cost and the fetch handler execution for all the network requests.

## Goals

Websites can opt-in to allow main resource requests to bypass ServiceWorker fetch handlers. 

## Non-goals

Unlike [the navigation preload](https://w3c.github.io/ServiceWorker/#navigationpreloadmanager), this feature doesn't wait for the navigation response in fetch handlers.

For this initial performance-gathering experiment, bypassing the service worker for subresources is a non-goal.

## Initial experiment

We (Google Chrome team) will start the origin trial, which allows developers to opt-in the feature on their sites. Regarding the introduction and basic registration process, please refer to [this page](https://developer.chrome.com/docs/web-platform/origin-trials/).

Once a token string is generated from [the dashboard](https://developer.chrome.com/origintrials), developers need to set the HTTP response header to their ServiceWorker files.

```
Origin-Trial: TOKEN_GOES_HERE
```

One important thing to mention here is that the header has to be added to **the ServiceWorker script, not to the page**. This is the limitation to run the origin trial on ServiceWorker. HTML meta tags are not accepted, and the feature will take effect after the ServiceWorker registration.

To enable the bypass for all service workers when running Chrome locally, enable `BypassServiceWorkerFetchHandlersForMainResource` from chrome://flags.

Normally ServiceWorker bootstrap happens before triggering a navigation request when the registered ServiceWorker is inactive and the url to navigate is in its scope. This is a required process to make sure the ServiceWorker fetch handler captures the request and response. But when the `ServiceWorkerBypassFetchHandler` feature is enabled, the browser triggers a navigation request immediately without waiting for the ServiceWorker bootstrap. The browser starts the ServiceWorker right after sending a navigation request, but that is not captured in the ServiceWorker fetch handler at all. Subresources are captured in the fetch handler as usual.

## Eventual goal

The purpose of the initial experiment is to gather data on whether this gives good performance or not. If that experiment shows good results, we want to expand to subresources, probably via [the declarative routing API](https://github.com/w3c/ServiceWorker/issues/1373) or similar. That will be the eventual final form of the API, not an `Origin-Trial: ...` header. We still don't have a concrete API surface to enable the feature yet. Any feedback or API proposals are welcomed.

## Previous efforts

ServiceWorker Subresource Filter ([issue on github](https://github.com/w3c/ServiceWorker/issues/1584) / [blink-dev thread](https://groups.google.com/a/chromium.org/g/blink-dev/c/GtR3POPyefM)), which we used to work on to solve similar problems. ServiceWorker Subresource Filter was an effort to reduce the overhead introduced by fetch handlers for subresource loadings. As a result, we did not observe any significant performance difference. On the other hand, ServiceWorkerBypassFetchHandler is focused on bypassing ServiceWorker's boot and the fetch handler overhead especially for main resources (navigation requests). And we expect bypassing the bootstrap time for main resources is more impactful on the loading performance.
