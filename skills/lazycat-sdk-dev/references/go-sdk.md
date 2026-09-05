# Go SDK Reference

## Installation

```bash
go get -u gitee.com/linakesi/lzc-sdk/lang/go
```

## Import Paths

```go
import (
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/common"
    "gitee.com/linakesi/lzc-sdk/lang/go/sys"
)
```

---

## Core Pattern: API Gateway

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

### APIGateway Available Services

| Field | Client Type | Purpose |
|-------|-------------|---------|
| `gw.Users` | `common.UserManagerClient` | User management |
| `gw.Devices` | `common.EndDeviceServiceClient` | End device (client) management |
| `gw.Box` | `common.BoxServiceClient` | Box control (LED, power) |
| `gw.HClients` | `common.HClientServiceClient` | Client device binding management |
| `gw.PkgManager` | `sys.PackageManagerClient` | Application management |
| `gw.Message` | `common.MessageServiceClient` | In-app messages |
| `gw.Version` | `sys.VersionInfoServiceClient` | System version info |
| `gw.Permisions` | `common.PermissionManagerClient` | Permission management |
| `gw.FileTransfer` | `common.FileTransferServiceClient` | File transfer |
| `gw.PeripheralDevice` | `common.PeripheralDeviceServiceClient` | Peripheral devices |
| `gw.ISCSIService` | `common.ISCSIServiceClient` | iSCSI storage |
| `gw.AccessControler` | `sys.AccessControlerServiceClient` | Access control |
| `gw.Btrfs` | `sys.BtrfsUtilClient` | Btrfs operations |
| `gw.DirMonitor` | `sys.DirMonitorClient` | Directory monitoring |
| `gw.TvOS` | `sys.TvOSClient` | TV devices |

---

## HClient: Client Device Binding Management

`gw.HClients` manages the binding between logged-in clients (HClient, i.e. 手机/桌面客户端实例) and logical client devices (`HClientDevice`). Useful for implementing device list management, renaming, and binding fixes in your own apps.

### Service Methods

| Method | Purpose |
|--------|---------|
| `ListHClients(uid)` | List all client instances of a user |
| `ListHClientDevices(uid)` | List all logical client devices of a user |
| `GetHClientDeviceCandidates(uid, hclientID)` | Get candidate devices when a client's device ID changed (dedupe assist) |
| `SetHClientDeviceBinding(uid, hclientID, hclientDeviceID)` | Bind a client instance to a logical device |
| `SetHClientDeviceRemarkName(uid, hclientDeviceID, remarkName)` | Set device remark name |
| `DeleteHClient(uid, hclientID)` | Delete a client instance |
| `DeleteHClientDevice(uid, hclientDeviceID)` | Delete a logical client device |

> Note: `HClientDevice` no longer carries a `device_api_url` field (removed in recent SDK versions). Use `EndDevice.device_api_url` from `gw.Devices.ListEndDevices` when you need a Device API endpoint.

### Example: List Client Devices and Update Binding

```go
import (
    "context"
    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
)

func ListClientDevices(ctx context.Context, uid string) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    reply, err := gw.HClients.ListHClientDevices(ctx, &common.ListHClientDevicesRequest{Uid: uid})
    if err != nil {
        return err
    }
    for _, dev := range reply.Devices {
        // dev.Id, dev.RemarkName, dev.IsOnline, dev.LastLoginAt
        // dev.Hclients: the client instances bound to this device
        fmt.Printf("Device %s (%s) online=%v\n", dev.Id, dev.GetRemarkName(), dev.IsOnline)
    }
    return nil
}

func BindClientToDevice(ctx context.Context, uid, hclientID, hclientDeviceID string) error {
    gw, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return err
    }
    defer gw.Close()

    _, err = gw.HClients.SetHClientDeviceBinding(ctx, &common.SetHClientDeviceBindingRequest{
        Uid:             uid,
        HclientId:       hclientID,
        HclientDeviceId: hclientDeviceID,
    })
    return err
}
```

---

## User Management

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

## Device Management

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

## Box Control

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

## Application Management

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

## User Context in HTTP Handlers

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

## Go backend notification

后端发送通知适用于：后端已知道用户 `uid` 和目标客户端 `uniqueDeivceId`，并运行在可读取 SDK 默认应用证书路径的轻应用容器内。`package.yml` 必须声明 `user.notify`，要求 lzcos v1.6.0+。

```bash
go get -x gitee.com/linakesi/lzc-sdk@master
go mod tidy
```

```go
package notificationexample

import (
    "context"
    "errors"
    "fmt"
    "net/url"
    "strings"
    "time"

    gohelper "gitee.com/linakesi/lzc-sdk/lang/go"
    "gitee.com/linakesi/lzc-sdk/lang/go/common"
    "gitee.com/linakesi/lzc-sdk/lang/go/localdevice"
    "google.golang.org/grpc"
    "google.golang.org/grpc/metadata"
)

const notificationTimeout = 15 * time.Second

type NotificationPayload struct {
    Title       string
    Body        string
    DeeplinkURL string
}

func SendNotificationToDevice(ctx context.Context, uid, deviceID string, payload NotificationPayload) error {
    ctx, cancel := context.WithTimeout(ctx, notificationTimeout)
    defer cancel()

    gateway, err := gohelper.NewAPIGateway(ctx)
    if err != nil {
        return fmt.Errorf("create lzc api gateway: %w", err)
    }
    defer gateway.Close()

    device, err := findOnlineDevice(ctx, gateway, uid, deviceID)
    if err != nil {
        return err
    }
    return notifyDevice(ctx, device.GetDeviceApiUrl(), payload)
}

func findOnlineDevice(ctx context.Context, gateway *gohelper.APIGateway, uid, deviceID string) (*common.EndDevice, error) {
    reply, err := gateway.Devices.ListEndDevices(ctx, &common.ListEndDeviceRequest{Uid: uid})
    if err != nil {
        return nil, fmt.Errorf("list devices: %w", err)
    }
    for _, device := range reply.GetDevices() {
        if device.GetUniqueDeivceId() != deviceID {
            continue
        }
        if !device.GetIsOnline() || strings.TrimSpace(device.GetDeviceApiUrl()) == "" {
            return nil, errors.New("target device is offline or unavailable")
        }
        return device, nil
    }
    return nil, errors.New("device not found")
}

func notifyDevice(ctx context.Context, deviceAPIURL string, payload NotificationPayload) error {
    parsedURL, err := url.Parse(deviceAPIURL)
    if err != nil {
        return fmt.Errorf("parse device api url: %w", err)
    }
    if parsedURL.Host == "" {
        return errors.New("device api url has no host")
    }

    cred, err := gohelper.BuildClientCredOption(gohelper.CAPath, gohelper.APPKeyPath, gohelper.APPCertPath)
    if err != nil {
        return fmt.Errorf("build device tls credentials: %w", err)
    }

    authConn, err := grpc.DialContext(ctx, parsedURL.Host, grpc.WithBlock(), cred)
    if err != nil {
        return fmt.Errorf("dial device api for auth: %w", err)
    }
    token, err := gohelper.RequestAuthToken(ctx, authConn)
    _ = authConn.Close()
    if err != nil {
        return fmt.Errorf("request device auth token: %w", err)
    }

    conn, err := grpc.DialContext(ctx, parsedURL.Host, grpc.WithBlock(), cred)
    if err != nil {
        return fmt.Errorf("dial device api: %w", err)
    }
    defer conn.Close()

    req := &localdevice.NotifyRequest{Title: payload.Title, Body: payload.Body}
    if payload.DeeplinkURL != "" {
        req.DeeplinkUrl = &payload.DeeplinkURL
    }

    ctx = metadata.AppendToOutgoingContext(ctx, "lzc_dapi_auth_token", token.Token)
    _, err = localdevice.NewNotificationServiceClient(conn).Notify(ctx, req)
    return err
}
```

```go
err := notificationexample.SendNotificationToDevice(ctx, uid, deviceID, notificationexample.NotificationPayload{
    Title:       "新消息",
    Body:        "你有一条来自轻应用的新消息",
    DeeplinkURL: "lzc://client/app/open?appId=cloud.lazycat.app.photo&path=/",
})
```

验证：

```bash
go mod tidy
go test ./...
```

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

---

## HTTP Headers Reference

LazyCat injects these headers into HTTP requests:

| Header | Description | Example |
|--------|-------------|---------|
| `x-hc-user-id` | Current user ID | `lazycat` |
| `x-hc-user-role` | User role | `admin`, `user` |
| `x-hc-device-id` | Device ID | `device-123` |
| `x-hc-device-version` | Device version | `1.4.1` |
