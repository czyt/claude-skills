# JavaScript/TypeScript SDK Reference

## Installation

```bash
npm install @lazycatcloud/sdk
# or
pnpm install @lazycatcloud/sdk
```

---

## Basic Usage

The JS/TS SDK uses grpc-web to provide services.

```javascript
import { lzcAPIGateway } from "@lazycatcloud/sdk"

// Initialize lzcapi
const lzcapi = new lzcAPIGateway(window.location.origin, false)

// Query all applications
const apps = await lzcapi.pkgm.QueryApplication({ appidList: [] })
console.debug("applications: ", apps)
```

---

## Response Example

```json
{
  "infoList": [
    {
      "appid": "cloud.lazycat.developer.tools",
      "status": 4,
      "version": "0.1.3",
      "title": "LCMD Cloud Developer Tools",
      "description": "",
      "icon": "//lcc.heiyu.space/sys/icons/cloud.lazycat.developer.tools.png",
      "domain": "dev.lcc.heiyu.space",
      "builtin": false,
      "unsupportedPlatforms": []
    }
  ]
}
```

---

## Common API Operations

### Get Application List

```javascript
const apps = await lzcapi.pkgm.QueryApplication({ appidList: [] })
```

### Get Specific Applications

```javascript
const apps = await lzcapi.pkgm.QueryApplication({
  appidList: ["cloud.lazycat.app.myapp"]
})
```

### Check App Status

```javascript
const apps = await lzcapi.pkgm.QueryApplication({
  appidList: ["cloud.lazycat.app.target"]
})

if (apps.infoList.length > 0 && apps.infoList[0].status === 4) {
  console.log("Target app is running")
}
```

---

## Environment Detection

### Detect Client WebShell

When developing frontend apps that run inside LazyCat iOS/Android client, you need to detect the environment:

```js
import base from "@lazycatcloud/sdk/dist/extentions/base"

// Detect platform
const isIOS = base.isIosWebShell()
const isAndroid = base.isAndroidWebShell()
const isClient = isIOS || isAndroid

// Safe call wrapper
export function safeCall(fn, fallback) {
  try {
    return fn ? fn() : fallback
  } catch (error) {
    console.warn("[client-api] call failed", error)
    return fallback
  }
}
```

### Conditional API Usage

```js
import { AppCommon } from "@lazycatcloud/sdk/dist/extentions"
import base from "@lazycatcloud/sdk/dist/extentions/base"

export async function openApp(url, appid) {
  if (base.isClientWebShell()) {
    await AppCommon.LaunchApp(url, appid)
  } else {
    // Browser fallback
    window.location.href = url
  }
}
```

---

## AppCommon Extensions

The `AppCommon` module provides iOS/Android client capabilities:

```js
import { AppCommon } from "@lazycatcloud/sdk/dist/extentions"

// Launch another app
await AppCommon.LaunchApp(url, appid, { forcedRefresh: true })

// Full screen control
await AppCommon.SetFullScreen()
await AppCommon.CancelFullScreen()
const isFull = await AppCommon.GetFullScreenStatus()

// File sharing
await AppCommon.ShareWithFiles(path)
await AppCommon.ShareWithFiles(undefined, [path1, path2])

// Media sharing
await AppCommon.ShareMedia({ ids: ["media-id-1"] })

// Open with other app
await AppCommon.OpenWith(boxName, path, appid)

// iOS only: brightness/volume
await AppCommon.SetScreenBrightness(0.5)
await AppCommon.SetDeviceVolume(0.5)
const brightness = await AppCommon.GetScreenBrightness()
const volume = await AppCommon.GetDeviceVolume()
```

---

## MediaSession (Android Only)

For audio player apps on Android, use MediaSession for lock screen control:

```js
import {
  MediaSession,
  isMediaSessionAvailable,
} from "@lazycatcloud/sdk/dist/extentions/mediasession/index"

if (isMediaSessionAvailable()) {
  await MediaSession.setMetadata({ title: "My Track" })
  await MediaSession.setPlaybackState({ playbackState: "playing" })
  await MediaSession.setActionHandler({ action: "play" }, () => audio.play())
  await MediaSession.setActionHandler({ action: "pause" }, () => audio.pause())
}
```

---

## Newer AppCommon Capabilities (SDK >= 0.1.55x)

Recent versions of `@lazycatcloud/sdk` (currently 0.1.55x) added many client capabilities. Highlights:

### Adjust App Viewport Size (PC desktop client only)

```ts
const result = await AppCommon.SetViewportSize({ width: 1280, height: 720 })

if (!result) {
  // Old desktop client without support, refused, or IPC error (reason logged to console)
}
```

- `true`: viewport/content area resized successfully.
- `false`: unsupported desktop client version, rejected, or IPC failure (reason in console).
- `undefined`: called on a non-PC platform.
- `width`/`height` are absolute CSS pixels of the app content area; no need to know host title bar layout.

### Other Recently Added APIs

```ts
import { AppCommon } from "@lazycatcloud/sdk/dist/extentions"

// Native desktop video player (returns a session with events)
const session = await AppCommon.OpenNativeVideoPlayer({
  playlist: { index: 0, items: [{ id: "1", file: "https://example.com/v.mkv", name: "Episode 1", cover: "https://example.com/cover.jpg", duration: 1320 }] },
  options: { singleton: true },
})
const unsubscribe = session.subscribe((event) => console.log(event.name, event.payload))
// await session.close()

// QR code scanning (mobile clients)
const text = await AppCommon.ScanQrCode()

// Haptic feedback (VibrateType: EFFECT_TICK / EFFECT_CLICK / EFFECT_HEAVY_CLICK / EFFECT_DOUBLE_CLICK)
await AppCommon.Vibrate("EFFECT_CLICK")

// Runtime capability probing
const capabilities = await AppCommon.GetCapabilitys()

// Client permission requests (album / contacts / BLE / microphone / camera)
const status = await AppCommon.GetClientAuthorizationStatus("camera")
if (!status?.authorized) await AppCommon.RequestClientAuthorization("camera")

// Biometric lock for the light app
const info = await AppCommon.GetBiometricInfo()
const auth = await AppCommon.AuthenticateBiometric("Unlock to continue")
```

> Always guard these calls: check the environment first (`base.isIosWebShell()` / `base.isAndroidWebShell()` / capability probing via `GetCapabilitys`), and provide browser fallbacks.

---

## Frontend Integration

For web applications running inside LazyCat:

```javascript
// In your Vue/React/etc app
import { lzcAPIGateway } from "@lazycatcloud/sdk"

// Use window.location.origin for automatic endpoint detection
const api = new lzcAPIGateway(window.location.origin, false)

export async function getAppInfo(appid) {
  const result = await api.pkgm.QueryApplication({ appidList: [appid] })
  return result.infoList[0]
}
```

---

## Best Practices

### 1. Lazy Loading

Load SDK only when needed in frontend applications:

```javascript
// Lazy import
const { lzcAPIGateway } = await import("@lazycatcloud/sdk")
```

### 2. Error Handling

```javascript
try {
  const apps = await lzcapi.pkgm.QueryApplication({ appidList: [] })
  // Handle success
} catch (error) {
  console.error("Failed to query applications:", error)
  // Handle error gracefully
}
```

### 3. Connection Reuse

Initialize the API gateway once and reuse:

```javascript
// Create once, use everywhere
let api = null

export function getAPI() {
  if (!api) {
    api = new lzcAPIGateway(window.location.origin, false)
  }
  return api
}
```