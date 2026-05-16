# PWA: App Shell, Background Sync, Web Push

## Quick Reference

| Capability | Mechanism | Requires SW? |
|---|---|---|
| Installability | Web App Manifest + HTTPS + SW | Yes |
| App Shell | Service Worker pre-caches shell HTML/JS/CSS | Yes |
| Background Sync | `SyncManager` API via Service Worker | Yes |
| Web Push | Push API + Notifications API + Service Worker | Yes |
| Standalone mode | `display: standalone` in manifest | No (just manifest) |

---

## What Is This?

A Progressive Web App (PWA) is a web app that delivers app-like capabilities through web standards: it's installable, works offline, can receive push notifications, and can sync data in the background. None of these require a native app store.

The foundation is three things:
1. **HTTPS** — required by Service Workers and most PWA APIs
2. **Web App Manifest** — tells the browser how to install and display the app
3. **Service Worker** — the runtime that makes offline, push, and sync possible

```json
// manifest.json
{
  "name": "My App",
  "short_name": "App",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0070f3",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

```html
<link rel="manifest" href="/manifest.json">
```

> **Check yourself:** What three criteria must be met before Chrome shows the "Add to Home Screen" prompt?

---

## Why PWAs Exist

Native apps have distribution (app stores), offline capability, push notifications, and hardware access. Web apps have reach (no install barrier) and deployability (no review process). PWAs try to close the gap: the web app gets native capabilities without requiring a native app wrapper.

The business case: every additional step in installation (App Store → download → open) loses a significant fraction of users. A PWA can be used immediately and installed later, reducing funnel friction while still delivering key native features.

---

## App Shell Architecture

### What it is

The App Shell is the minimal HTML, CSS, and JavaScript needed to render the UI scaffolding — navigation, layout, loading states — before any dynamic content loads. It's cached aggressively via the Service Worker so the shell loads instantly from cache on repeat visits, even offline.

```
App Shell (cached):           Dynamic Content (network):
┌─────────────────────┐       ┌──────────────┐
│ Header              │   +   │ API data     │
│ Navigation          │       │ Images       │
│ Layout skeleton     │       │ User content │
└─────────────────────┘       └──────────────┘
     Instant from cache            Network (or IndexedDB fallback)
```

### Implementation

Cache the shell on Service Worker install:

```js
const SHELL_CACHE = 'app-shell-v1';
const SHELL_ASSETS = ['/', '/app.js', '/styles.css', '/offline.html'];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(SHELL_CACHE).then(cache => cache.addAll(SHELL_ASSETS))
  );
  self.skipWaiting();
});
```

Serve the shell cache-first, API data network-first:

```js
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // Cache-first for shell assets
  if (SHELL_ASSETS.includes(url.pathname)) {
    event.respondWith(
      caches.match(event.request).then(cached => cached ?? fetch(event.request))
    );
    return;
  }

  // Network-first for API calls
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(
      fetch(event.request)
        .then(async response => {
          const cache = await caches.open('api-v1');
          cache.put(event.request, response.clone());
          return response;
        })
        .catch(() => caches.match(event.request))
    );
    return;
  }
});
```

### Shell vs. SSR

The App Shell pattern is best for SPAs. SSR apps cache the server-rendered HTML directly (full page cache), which is different from caching a static shell that receives data separately. Streaming SSR with Service Workers can combine both: cache the shell, stream dynamic content.

---

## Background Sync

Covered in depth in Phase 7.2 (Service Workers), but the PWA context adds:

### User-facing reliability

The key user experience proposition: "Take an action offline, trust it will complete." A form submission while offline registers a sync tag; the browser fires the sync event when online, completing the submission.

```js
// In the page
async function submitForm(data) {
  await db.put('pending-submissions', data);
  const reg = await navigator.serviceWorker.ready;

  if ('sync' in reg) {
    await reg.sync.register('submit-form');
  } else {
    // Fallback: try to submit now, may fail if offline
    await doSubmit(data);
  }
}
```

### Periodic Background Sync

Allows the SW to wake up periodically to refresh content — like a native news app that fetches new articles in the background:

```js
// Register (requires permission)
const reg = await navigator.serviceWorker.ready;
await reg.periodicSync.register('refresh-content', { minInterval: 24 * 60 * 60 * 1000 }); // 24h

// In sw.js
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'refresh-content') {
    event.waitUntil(refreshContent());
  }
});
```

Periodic Background Sync is Chrome-only, requires `periodic-background-sync` permission, and the interval is browser-enforced (not guaranteed to fire at exact intervals).

> **Check yourself:** Why does Background Sync fire in the Service Worker rather than in the page?

---

## Web Push

Web Push lets your server send notifications to users even when your app isn't open. The stack involves:
1. **Push API** — browser mechanism for subscribing to push messages
2. **Push Service** — browser vendor's infrastructure (Google FCM for Chrome, Mozilla's for Firefox)
3. **VAPID keys** — authentication proving the push message comes from your server
4. **Notifications API** — displaying the actual notification

### Flow

```
1. User grants notification permission
2. Browser subscribes to the vendor's Push Service → gives you a PushSubscription (endpoint + keys)
3. Your server stores the PushSubscription
4. Your server sends a push message (encrypted, signed with VAPID) to the Push Service endpoint
5. Push Service delivers to browser (even with no open tab)
6. Browser wakes up Service Worker → fires 'push' event
7. SW displays notification
```

### Subscribing

```js
// Request notification permission
const permission = await Notification.requestPermission();
if (permission !== 'granted') return;

// Subscribe to push
const reg = await navigator.serviceWorker.ready;
const subscription = await reg.pushManager.subscribe({
  userVisibleOnly: true, // must be true — no silent pushes in browsers
  applicationServerKey: urlBase64ToUint8Array(PUBLIC_VAPID_KEY),
});

// Send subscription to your server
await fetch('/api/push-subscribe', {
  method: 'POST',
  body: JSON.stringify(subscription),
  headers: { 'Content-Type': 'application/json' },
});
```

### Receiving in the Service Worker

```js
self.addEventListener('push', (event) => {
  const data = event.data?.json();

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192.png',
      badge: '/badge.png',
      data: { url: data.url },
    })
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  event.waitUntil(
    clients.openWindow(event.notification.data.url)
  );
});
```

### Server-side (sending)

The server uses the `web-push` npm library to send encrypted messages to the Push Service:

```js
// Node.js server
const webpush = require('web-push');

webpush.setVapidDetails(
  'mailto:admin@example.com',
  process.env.VAPID_PUBLIC_KEY,
  process.env.VAPID_PRIVATE_KEY
);

await webpush.sendNotification(storedSubscription, JSON.stringify({
  title: 'New message',
  body: 'You have a new message from Alice',
  url: '/messages',
}));
```

### `userVisibleOnly: true`

All browser implementations require `userVisibleOnly: true` on push subscriptions. This means every push message must result in a visible notification — silent background pushes are blocked. This is a privacy and battery protection measure.

---

## Install Prompt

Chrome shows the "Add to Home Screen" prompt automatically when:
1. HTTPS
2. Valid Web App Manifest (with name, icons, start_url, display)
3. Service Worker registered with a fetch handler

You can defer and control the prompt:

```js
let deferredPrompt;
window.addEventListener('beforeinstallprompt', (e) => {
  e.preventDefault();
  deferredPrompt = e; // stash it
  showInstallButton(); // show your own UI
});

// When user clicks your install button
installButton.addEventListener('click', async () => {
  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice;
  if (outcome === 'accepted') trackInstall();
  deferredPrompt = null;
});

// After install completes
window.addEventListener('appinstalled', () => {
  hideInstallButton();
});
```

Detect if running in standalone mode (installed):

```js
if (window.matchMedia('(display-mode: standalone)').matches || navigator.standalone) {
  // Running as installed PWA, not in browser tab
}
```

---

## Gotchas

**1. iOS Safari has significant PWA limitations**
iOS Safari supports basic Service Worker caching and offline. It does not support Web Push (added in iOS 16.4, but only for home-screen-installed PWAs and with additional restrictions). Periodic Background Sync is not supported. Storage may be limited to 50MB per origin in some configurations. Always test on iOS for PWAs.

**2. `userVisibleOnly: true` is mandatory — silent push doesn't work**
Browsers enforce that every push event results in a notification. If your Service Worker's push handler doesn't call `showNotification()`, Chrome may display a generic "site has been updated" notification. Design your push payloads to always be user-visible.

**3. Push subscriptions expire**
`PushSubscription` endpoints can expire or be invalidated (user uninstalls the browser, resets notifications). Your server should handle 410 Gone responses from the Push Service by deleting the stored subscription. Sending to an expired subscription is a waste of resources.

**4. `beforeinstallprompt` is Chrome-only**
Firefox and Safari have their own install flows and don't fire this event. Build your install UI defensively — show a generic "Add to Home Screen" instruction on non-Chrome browsers.

**5. App Shell ≠ working offline**
Caching the shell doesn't make your app fully offline-capable. API responses, dynamic data, and images also need caching strategies. A blank shell with a spinner offline is not a good user experience. Plan offline UI states explicitly.

**6. Service Worker updates break cached shell**
When you deploy a new version, old cached shell assets may be served to users who haven't reloaded. Version your caches (`app-shell-v2`) and clean up old caches in `activate`. Include a reload prompt when a new SW activates.

---

## Interview Questions

**Q (High): What makes an app a PWA and what capabilities does a Service Worker enable for it?**

Answer: A PWA is a web app that meets the criteria for installability (HTTPS, valid Web App Manifest, registered Service Worker with a fetch handler) and uses web APIs to deliver native-like capabilities. The Service Worker is the runtime enabler: it intercepts network requests (enabling offline via the Cache API), handles Background Sync events (deferred network requests that complete when online), and handles Push events (receiving messages from your server even with no open tab to show notifications). Without a Service Worker, you can't do offline, background sync, or push — you just have an installable web page.

**Q (High): Walk through the Web Push flow from subscription to notification display.**

Answer: The user grants notification permission. The page calls `pushManager.subscribe()`, which registers with the browser vendor's Push Service (Google FCM, Mozilla, etc.) and returns a `PushSubscription` containing an endpoint URL and encryption keys. The page sends this subscription to your server. When you want to send a push: your server encrypts the payload with the subscription's keys, signs it with your VAPID private key, and POSTs it to the Push Service endpoint. The Push Service delivers the message to the browser, even with no open tab. The browser wakes up the Service Worker and fires a `push` event. The SW calls `registration.showNotification()`, displaying the notification. `userVisibleOnly: true` is required — silent pushes are blocked.

**Q (High): What is the App Shell architecture and when is it appropriate?**

Answer: The App Shell is the minimal, static HTML/CSS/JS that renders the app's UI scaffold — navigation, layout, loading states — before any dynamic content loads. It's pre-cached by the Service Worker during install and served cache-first on repeat visits. Dynamic content (API data, user content) loads separately from the network. This delivers instant first-meaningful-paint on repeat visits and keeps the app usable offline (with the shell visible and a fallback message for network data). It's appropriate for SPAs where the shell is stable across navigations. It's less appropriate for SSR apps where the full HTML differs per page, or for content-driven sites where the dynamic content IS the primary experience.

**Q (Medium): Why is `userVisibleOnly: true` required in push subscriptions?**

Answer: Browsers mandate that every push message results in a visible notification to the user. `userVisibleOnly: true` is the declaration of this constraint. The reason: silent background pushes could be used for covert tracking — sending a push to check if the user's browser is online, or to trigger background computation without the user knowing. It also protects battery life. Without this constraint, apps could send dozens of silent push messages per day for telemetry. If your Service Worker's push handler fails to show a notification, Chrome will display a generic fallback notification — it won't silently let you skip it.

**Q (Low): What happens to Web Push subscriptions when a user changes browsers or reinstalls?**

Answer: The `PushSubscription` endpoint is tied to the browser vendor's Push Service and the browser's local keying material. If the user reinstalls the browser, clears browser data, or the subscription expires, the endpoint becomes invalid. Your server will receive a `410 Gone` (or `404`) response when trying to push to an expired endpoint. The server must handle this by deleting the stored subscription. Users who reinstall will need to re-subscribe — there's no way to migrate a push subscription across browser reinstalls. Track active subscriptions and re-prompt for push permission on re-install scenarios.

---

## Self-Assessment

- [ ] List the three criteria Chrome needs before showing the install prompt
- [ ] Draw the Web Push flow (subscribe → server sends → notification appears) from memory
- [ ] Describe what the App Shell pattern caches and what it leaves to the network
- [ ] Explain why `userVisibleOnly: true` is required and what happens if you don't show a notification
- [ ] Describe two ways iOS Safari differs from Chrome in its PWA support

---
*Next: Offline-first Architecture Patterns — the higher-level design patterns for building apps that treat the network as an enhancement rather than a requirement, including sync strategies, conflict resolution, and local-first principles.*
