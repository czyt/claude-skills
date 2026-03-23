---
name: lazycat-sdk-dev
description: LazyCat SDK development skill for building applications with Go and JavaScript/TypeScript SDKs. Use when developing apps that need to interact with LazyCat microservice system APIs, query app lists, manage devices, handle file pickers, or use minidb. Triggers when code imports @lazycatcloud/sdk, gitee.com/linakesi/lzc-sdk, or when user asks about LazyCat SDK, device management, app queries, or system integration.
---

# LazyCat SDK Development

This skill helps you develop applications that interact with the LazyCat microservice system using the official SDKs.

## Supported Languages

- **JavaScript/TypeScript** - `@lazycatcloud/sdk` (npm)
- **Go** - `gitee.com/linakesi/lzc-sdk/lang/go`

More languages coming soon: Rust, Dart, Cpp, Java, Python, Ruby, C#, PHP, Objective-C, Kotlin.

## Extensions

LazyCat also provides these extension libraries:

| Package | Purpose |
|---------|---------|
| `@lazycatcloud/minidb` | Small MongoDB-like database for serverless apps |
| `@lazycatcloud/lzc-file-pickers` | File picker for LazyCat storage integration |

---

## Go SDK

### Installation

```bash
go get -u gitee.com/linakesi/lzc-sdk/lang/go
```

### Import Paths

```go
import (
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/common"
    "gitee.com/linakesi/lzc-sdk/lang/go/sys"
)
```

### Core Pattern: API Gateway

The API Gateway is the main entry point for all SDK operations:

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
)

func doSomething(ctx context.Context) error {
    // Create API Gateway
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()  // Always close when done!

    // Use gw.Users, gw.Box, gw.PkgManager, gw.Devices...
    return nil
}
```

**Important:** Always call `defer gw.Close()` to release resources.

---

## Go SDK: User Management

### Query User Info

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/common"
)

func GetUserInfo(ctx context.Context, userID string) (*common.UserInfo, error) {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return nil, err
    }
    defer gw.Close()

    userInfo, err := gw.Users.QueryUserInfo(ctx, &common.UserID{Uid: userID})
    if err != nil {
        return nil, err
    }

    // Available fields:
    // - userInfo.Uid       - User ID
    // - userInfo.Nickname  - Display name
    // - userInfo.Avatar    - Avatar URL
    // - userInfo.Email     - Email address

    return userInfo, nil
}
```

### User Info Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `Uid` | string | User ID |
| `Nickname` | string | Display name |
| `Avatar` | string | Avatar URL |
| `Email` | string | Email address |

---

## Go SDK: Device Management

### List Devices

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/common"
)

func ListOnlineDevices(ctx context.Context, uid string) ([]string, error) {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return nil, err
    }
    defer gw.Close()

    request := &common.ListEndDeviceRequest{Uid: uid}
    devices, err := gw.Devices.ListEndDevices(ctx, request)
    if err != nil {
        return nil, err
    }

    var online []string
    for _, d := range devices.Devices {
        if d.IsOnline {
            online = append(online, d.Name)
        }
    }
    return online, nil
}
```

### Device Info Fields

| Field | Type | Description |
|-------|------|-------------|
| `Name` | string | Device name |
| `IsOnline` | bool | Online status |
| `DeviceId` | string | Device ID |

---

## Go SDK: Box Control

### Query Box Info

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
)

func GetBoxInfo(ctx context.Context) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    boxInfo, err := gw.Box.QueryInfo(ctx, nil)
    if err != nil {
        return err
    }

    // Available fields:
    // - boxInfo.PowerLed  - LED status (true=on, false=off)
    // - other device info...

    return nil
}
```

### Control Power LED

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/common"
)

func SetLEDStatus(ctx context.Context, on bool) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    _, err = gw.Box.ChangePowerLed(ctx, &common.ChangePowerLedRequest{
        PowerLed: on,
    })
    return err
}

func ToggleLED(ctx context.Context) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    // Get current status
    boxInfo, err := gw.Box.QueryInfo(ctx, nil)
    if err != nil {
        return err
    }

    // Toggle
    newStatus := !boxInfo.PowerLed
    _, err = gw.Box.ChangePowerLed(ctx, &common.ChangePowerLedRequest{
        PowerLed: newStatus,
    })
    return err
}
```

### Shutdown / Reboot

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/common"
)

func Reboot(ctx context.Context) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    _, err = gw.Box.Shutdown(ctx, &common.ShutdownRequest{
        Action: common.ShutdownRequest_Reboot,
    })
    return err
}

func PowerOff(ctx context.Context) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    _, err = gw.Box.Shutdown(ctx, &common.ShutdownRequest{
        Action: common.ShutdownRequest_Poweroff,
    })
    return err
}
```

---

## Go SDK: Application Management

### Query Applications

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/sys"
)

type AppInfo struct {
    AppID          string
    Title          string
    Icon           string
    Version        string
    Status         string
    InstanceStatus string
    MultiInstance  bool
    Builtin        bool
}

func ListInstalledApps(ctx context.Context) ([]AppInfo, error) {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return nil, err
    }
    defer gw.Close()

    resp, err := gw.PkgManager.QueryApplication(ctx, &sys.QueryApplicationRequest{})
    if err != nil {
        return nil, err
    }

    var apps []AppInfo
    for _, info := range resp.InfoList {
        // Filter: only installed apps
        if info.Status != sys.AppStatus_Installed {
            continue
        }

        // Filter: skip builtin/preinstalled apps
        if info.Builtin != nil && *info.Builtin {
            continue
        }

        app := AppInfo{
            AppID:          info.Appid,
            Status:         info.Status.String(),
            InstanceStatus: info.InstanceStatus.String(),
            MultiInstance:  info.MultiInstance,
        }

        if info.Title != nil {
            app.Title = *info.Title
        }
        if info.Icon != nil {
            app.Icon = *info.Icon
        }
        if info.Version != nil {
            app.Version = *info.Version
        }
        if info.Builtin != nil {
            app.Builtin = *info.Builtin
        }

        apps = append(apps, app)
    }

    return apps, nil
}

func GetAppStatus(ctx context.Context, appID string) (*sys.AppInfo, error) {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return nil, err
    }
    defer gw.Close()

    resp, err := gw.PkgManager.QueryApplication(ctx, &sys.QueryApplicationRequest{
        AppidList: []string{appID},
    })
    if err != nil {
        return nil, err
    }

    if len(resp.InfoList) == 0 {
        return nil, fmt.Errorf("app %s not found", appID)
    }

    return resp.InfoList[0], nil
}
```

### App Status Enums

```go
// App Status
sys.AppStatus_Installed      // App is installed
sys.AppStatus_NotInstalled   // App is not installed

// Instance Status
sys.InstanceStatus_Status_Running   // App is running
sys.InstanceStatus_Status_Starting  // App is starting
sys.InstanceStatus_Status_Stopped   // App is stopped
```

### Resume / Pause Application

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/sys"
)

func ResumeApp(ctx context.Context, appID, userID string) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    // Check if already running
    resp, err := gw.PkgManager.QueryApplication(ctx, &sys.QueryApplicationRequest{
        AppidList: []string{appID},
    })
    if err != nil {
        return err
    }

    if len(resp.InfoList) == 0 {
        return fmt.Errorf("app %s not found", appID)
    }

    appInfo := resp.InfoList[0]

    // Skip if already running
    if appInfo.InstanceStatus == sys.InstanceStatus_Status_Running {
        return nil
    }

    // Resume the app
    _, err = gw.PkgManager.Resume(ctx, &sys.AppInstance{
        Appid: appID,
        Uid:   userID,
    })
    return err
}

func PauseApp(ctx context.Context, appID, userID string) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    _, err = gw.PkgManager.Pause(ctx, &sys.AppInstance{
        Appid: appID,
        Uid:   userID,
    })
    return err
}
```

---

## Go SDK: User Context in HTTP Handlers

### Extract User Info from Headers

LazyCat injects user information into HTTP request headers:

```go
import (
    "github.com/gin-gonic/gin"
)

type BasicInfo struct {
    UserID        string
    UserRole      string
    DeviceID      string
    DeviceVersion string
}

func ExtractBasicInfo(c *gin.Context) BasicInfo {
    return BasicInfo{
        UserID:        c.GetHeader("x-hc-user-id"),
        UserRole:      c.GetHeader("x-hc-user-role"),
        DeviceID:      c.GetHeader("x-hc-device-id"),
        DeviceVersion: c.GetHeader("x-hc-device-version"),
    }
}

// Usage in handler
func MyHandler(c *gin.Context) {
    info := ExtractBasicInfo(c)
    if info.UserID == "" {
        c.JSON(401, gin.H{"error": "Unauthorized"})
        return
    }
    // ...
}
```

### Add User Context to gRPC Calls

For SDK calls that require user context, use `metadata.AppendToOutgoingContext`:

```go
import (
    "context"
    "google.golang.org/grpc/metadata"
)

func DoSomethingForUser(ctx context.Context, userID string) error {
    // Add user ID to outgoing context
    ctx = metadata.AppendToOutgoingContext(ctx, "x-hc-user-id", userID)

    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    // SDK calls will now include the user context
    resp, err := gw.PkgManager.QueryApplication(ctx, &sys.QueryApplicationRequest{})
    // ...
}
```

### Complete Handler Example (Gin)

```go
package handlers

import (
    "context"
    "net/http"

    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/common"
    "github.com/gin-gonic/gin"
    "google.golang.org/grpc/metadata"
)

func GetUserInfo(c *gin.Context) {
    ctx := c.Request.Context()

    // Get user ID from header
    userID := c.GetHeader("x-hc-user-id")
    if userID == "" {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Unauthorized"})
        return
    }

    // Add to context for SDK calls
    ctx = metadata.AppendToOutgoingContext(ctx, "x-hc-user-id", userID)

    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        c.AbortWithError(http.StatusInternalServerError, err)
        return
    }
    defer gw.Close()

    userInfo, err := gw.Users.QueryUserInfo(ctx, &common.UserID{Uid: userID})
    if err != nil {
        c.JSON(http.StatusOK, gin.H{
            "userId":   userID,
            "nickname": userID,  // Fallback to user ID
        })
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "userId":   userInfo.Uid,
        "nickname": userInfo.Nickname,
        "avatar":   userInfo.Avatar,
    })
}
```

---

## JavaScript/TypeScript SDK

### Installation

```bash
npm install @lazycatcloud/sdk
# or
pnpm install @lazycatcloud/sdk
```

### Basic Usage

The JS/TS SDK uses grpc-web to provide services.

```javascript
import { lzcAPIGateway } from "@lazycatcloud/sdk"

// Initialize lzcapi
const lzcapi = new lzcAPIGateway(window.location.origin, false)

// Query all applications
const apps = await lzcapi.pkgm.QueryApplication({ appidList: [] })
console.debug("applications: ", apps)
```

### Response Example

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

### Common API Operations

#### Get Application List

```javascript
const apps = await lzcapi.pkgm.QueryApplication({ appidList: [] })
```

#### Get Specific Applications

```javascript
const apps = await lzcapi.pkgm.QueryApplication({
  appidList: ["cloud.lazycat.app.myapp"]
})
```

---

## Using minidb Extension

`@lazycatcloud/minidb` is a small MongoDB-like database for serverless applications.

### Installation

```bash
npm install @lazycatcloud/minidb
```

### Usage Pattern

```javascript
import { Minidb } from "@lazycatcloud/minidb"

// Initialize database
const db = new Minidb("myapp")

// Insert document
await db.insert({ name: "item1", value: 100 })

// Query documents
const items = await db.find({ name: "item1" })

// Update document
await db.update({ name: "item1" }, { $set: { value: 200 } })

// Delete document
await db.remove({ name: "item1" })
```

---

## Using File Pickers Extension

`@lazycatcloud/lzc-file-pickers` allows your app to open files from LazyCat storage.

### Installation

```bash
npm install @lazycatcloud/lzc-file-pickers
```

### Usage Pattern

```javascript
import { pickFiles } from "@lazycatcloud/lzc-file-pickers"

// Open file picker
const files = await pickFiles({
  multiple: true,
  accept: [".pdf", ".doc", ".docx"]
})

// Handle selected files
for (const file of files) {
  console.log("Selected file:", file.path, file.name)
}
```

---

## Common Use Cases

### 1. App Status Checking

Check if another app is installed and running:

```javascript
const apps = await lzcapi.pkgm.QueryApplication({
  appidList: ["cloud.lazycat.app.target"]
})

if (apps.infoList.length > 0 && apps.infoList[0].status === 4) {
  console.log("Target app is running")
}
```

### 2. Device Management

List and manage connected devices:

```go
devices, _ := lzcapi.Devices.ListEndDevices(ctx, &common.ListEndDeviceRequest{})

for _, d := range devices.Devices {
    fmt.Printf("Device: %s, Online: %v\n", d.Name, d.IsOnline)
}
```

### 3. Cross-App Communication

Apps can communicate using service discovery:

```yaml
# In manifest.yml
services:
  myapp:
    environment:
      - TARGET_API=http://target-app.lzcapp:8080
```

### 4. LED Control with Status Check

```go
func SmartLEDControl(ctx context.Context, targetStatus bool) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    // Get current status
    boxInfo, err := gw.Box.QueryInfo(ctx, nil)
    if err != nil {
        return err
    }

    // Only change if different
    if boxInfo.PowerLed != targetStatus {
        _, err = gw.Box.ChangePowerLed(ctx, &common.ChangePowerLedRequest{
            PowerLed: targetStatus,
        })
    }
    return err
}
```

---

## Best Practices

### 1. Connection Management

Always close the API Gateway when done:

```go
gw, err := gohelper.NewAPIGateway(ctx)
if err != nil {
    return err
}
defer gw.Close()  // CRITICAL!
```

### 2. User Context

Always add user context for operations that need user-specific data:

```go
ctx = metadata.AppendToOutgoingContext(ctx, "x-hc-user-id", userID)
```

### 3. Error Handling

Handle SDK initialization errors gracefully:

```go
gw, err := gohelper.NewAPIGateway(ctx)
if err != nil {
    log.Error().Err(err).Msg("Failed to create API gateway")
    // Graceful degradation
    return fallbackBehavior()
}
```

### 4. Connection Reuse

Initialize API gateway once per operation, not per SDK call:

```go
// Good: One gateway for multiple operations
func DoMultipleThings(ctx context.Context) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    gw.Users.QueryUserInfo(...)
    gw.Box.QueryInfo(...)
    gw.PkgManager.QueryApplication(...)
    return nil
}
```

### 5. Context Usage

Use proper context for timeout and cancellation:

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

gw, err := gohelper.NewAPIGateway(ctx)
```

### 6. Lazy Loading (Frontend)

Load SDK only when needed in frontend applications:

```javascript
// Lazy import
const { lzcAPIGateway } = await import("@lazycatcloud/sdk")
```

---

## HTTP Headers Reference

LazyCat injects these headers into HTTP requests:

| Header | Description | Example |
|--------|-------------|---------|
| `x-hc-user-id` | Current user ID | `lazycat` |
| `x-hc-user-role` | User role | `admin`, `user` |
| `x-hc-device-id` | Device ID | `device-123` |
| `x-hc-device-version` | Device version | `1.4.1` |

---

## References

- **Official Docs**: https://developer.lazycat.cloud
- **SDK Repository**: https://gitee.com/linakesi/lzc-sdk
- **npm Packages**: https://www.npmjs.com/search?q=%40lazycatcloud