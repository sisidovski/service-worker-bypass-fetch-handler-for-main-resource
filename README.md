# Explainer: ServiceWorkerBypassFetchHandlerForMainResource

## Authors

-  [sisidovski](https://github.com/sisidovski)
-  [KenjiBaheux](https://github.com/KenjiBaheux)

## Participate

-  https://github.com/sisidovski/service-worker-bypass-fetch-handler-for-main-resource/issues


## Introduction
ServiceWorker is a popular API involved on 15% of page loads. This API offers many advanced features such as fetch handling, push notification, background sync, etc. The ServiceWorker’s fetch handling capability allows developers to intercept and control the resource fetch requests via “fetch” event handler. This enables web apps to be reliable even while offline and delivers faster loading experiences by using its own CacheStorage. However, the cost of starting up a ServiceWorker is non-negligible. In the worst case, it can take up to hundreds of milliseconds to bootstrap the ServiceWorker itself and process fetch event handlers.

To improve the performance of ServiceWorkers in the fetch handling scenarios, we’d like to introduce the “ServiceWorker fetch fast-path”, which is a way to bypass fetch handlers when appropriate. Normally, the ServiceWorker activation is triggered before sending a request if the resource is in the ServiceWorker’s scope. In other words, the bootstrap process typically blocks the actual request regardless of whether the resource actually needs to be handled by handlers or not. This is unfortunate in the cases where the fetch handlers do not need to handle the resource. What if there was a way to tell the browser in advance which resource requests can bypass the fetch handler? This is what the “fetch fast-path” is about. With this feature, the request will immediately happen without waiting for the ServiceWorker bootstrap and will bypass the fetch handlers. The ServiceWorker bootstrap is started at the same time as the request happens, but it doesn’t intercept the request/response in the fetch handler. As a result, the browser can mitigate the bootstrap cost and the fetch handler execution.


## Goals

Offset the ServiceWorker bootstrap latency on websites that don’t need to have their main resource requests handled by the ServiceWorker’s fetch handler. 

## Non-goals

For this initial performance-gathering experiment, bypassing the service worker for subresources is a non-goal.

## Related APIs
Unlike [the navigation preload API](https://w3c.github.io/ServiceWorker/#navigationpreloadmanager), the proposed feature doesn’t wait for the navigation response in fetch handlers.


## Origin Trial

We (Google Chrome team) will start the origin trial, which allows developers to opt-in the feature on their sites. Regarding the introduction and basic registration process, please refer to [this page](https://developer.chrome.com/docs/web-platform/origin-trials/).

Once a token string is generated from [the dashboard](https://developer.chrome.com/origintrials), developers need to set the HTTP response header to their ServiceWorker files.

```
Origin-Trial: TOKEN_GOES_HERE
```

One important thing to mention here is that the header has to be added to **the ServiceWorker script, not to the page**. HTML meta tags are not accepted, and the feature will take effect after the ServiceWorker registration.

For local testing, you can enable this feature by flipping the `Bypass Service Worker Fetch Handler for main resource` flag from chrome://flags.

Normally ServiceWorker bootstrap happens before triggering a navigation request when the registered ServiceWorker is inactive and the url to navigate is in its scope. This is a required process to make sure the ServiceWorker fetch handler captures the request and response. But when the `ServiceWorkerBypassFetchHandler` feature is enabled, the browser will trigger the navigation request immediately without waiting for the ServiceWorker bootstrap. The browser will still start the ServiceWorker right after sending a navigation request. Note that the response for the navigation request will not be captured in the ServiceWorker fetch handler at all. On the other hand, subresources will be captured in the fetch handler as usual.

## Goal of the origin trial

The purpose of the Origin Trial is to gather data on whether this gives good performance or not. If that experiment shows good results, we will consider expanding this approach to specific subresources, probably via [the declarative routing API]((https://github.com/w3c/ServiceWorker/issues/1373)) or similar. We don’t yet have a concrete idea for the proper API surface to enable this feature. Any feedback or API proposals are welcomed: for the main resource case, and for the subresource scenarios.

## Previous efforts

ServiceWorker Subresource Filter ([issue on github]((https://github.com/w3c/ServiceWorker/issues/1584)) / [blink-dev thread](https://groups.google.com/a/chromium.org/g/blink-dev/c/GtR3POPyefM)), which was trialed by engineers at Facebook was intended to solve a similar problem. ServiceWorker Subresource Filter was an effort to reduce the overhead introduced by fetch handlers for subresource loadings. As a result, Facebook did not observe any conclusive performance difference. On the other hand, ServiceWorkerBypassFetchHandler is focused on mitigating ServiceWorker’s bootstrap latency and the fetch handler overhead associated to handling main resources (navigation requests). We believe that this scenario is more impactful on the loading performance.
