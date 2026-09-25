# SimpliFi KYC SDK Integration Guide

# Overview

The SimpliFi KYC SDK lets you launch identity verification (via the Mawarid provider) from inside your app without building any verification UI yourself. The host loads a secure SimpliFi page (iframe/popup on web, WebView on native), sends it a config over `postMessage`, and receives a single `SDK_FLOW_RESULT` once verification succeeds or fails. The intro, document review, provider handoff, and polling screens are all handled by the SDK.

**Base URL:** `https://{env}-virtualcard.simplifipay.com/simplifi-sdk/`

### Parameters through postMessage

The SDK accepts credentials via `postMessage`, delivered after the SDK has fully loaded and signalled readiness — same handshake used by the rest of the Virtual Card SDK.

**The Handshake**

1. Host loads the SDK URL in an iframe, popup, or WebView
1. SDK mounts and sends `SIMPLIFI_SDK_READY` to the host
1. Host receives `SIMPLIFI_SDK_READY` and sends `SIMPLIFI_SDK_CONFIG` with credentials
1. SDK validates the config and sends `SIMPLIFI_SDK_CONFIG_ACK` (accepted) or `SIMPLIFI_SDK_CONFIG_ERROR` (rejected)
1. On flow completion, SDK sends `SDK_FLOW_RESULT`

> **Important:** Never send credentials before receiving `SIMPLIFI_SDK_READY`. The SDK may not have mounted yet and the message will be lost.

> **Important:** Send config **once**, at the very start. Do not resend it later in the flow.

**Config Payload**

Sent to the SDK after it signals readiness via `SIMPLIFI_SDK_READY`.

| Field         | Type   | Required | Description |
|-------------- | ------ | -------- | ----------- |
| **token**     | String | Yes      | Admin-scoped JWT for this specific user, minted server-side by **your backend** via the [Auth API](https://ss-docs.simplifipay.com/login-to-generate-sdk-admin-jwt-token-43566163e0). This is not the customer's own session/login token — the customer never generates or sees it directly. |
| **userID**    | String | Yes      | The unique 36-character ID of the user to verify. |
| **action**    | String | Yes      | Must be `initiate_kyc`. |

**Available Actions**

| Action              | Description |
| ------------------- | ----------- |
| **initiate_kyc**    | Launch identity verification |

## Message Protocol

The SDK and host communicate via `postMessage`. All messages are JSON objects.

### SDK → Host

| eventName                   | source            | Description |
| --------------------------- | ----------------- | ----------- |
| `SIMPLIFI_SDK_READY`        | `simplifi-sdk`    | SDK loaded and waiting for config |
| `SIMPLIFI_SDK_CONFIG_ACK`   | `simplifi-sdk`    | Config accepted, SDK initialising |
| `SIMPLIFI_SDK_CONFIG_ERROR` | `simplifi-sdk`    | Config rejected — check `errorCode` and `message` |
| `SDK_FLOW_RESULT`           | `simplifi-sdk`    | Flow completed — check `status` (`SUCCESS`/`FAILURE`) and `flow` |

### Host → SDK

| eventName             | source            | Description |
| --------------------- | ----------------- | ----------- |
| `SIMPLIFI_SDK_CONFIG` | `simplifi-parent` | Send config payload to SDK |

**Config message**
```json
{
  "source": "simplifi-parent",
  "eventName": "SIMPLIFI_SDK_CONFIG",
  "payload": {
    "token": "YOUR_TOKEN",
    "userId": "YOUR_USER_UUID",
    "action": "initiate_kyc"
  }
}
```

**`SDK_FLOW_RESULT` for KYC**
```json
{
  "source": "simplifi-sdk",
  "eventName": "SDK_FLOW_RESULT",
  "flow": "INITIATE_KYC",
  "status": "SUCCESS",
  "action": "initiate_kyc",
  "userID": "YOUR_USER_UUID",
  "message": "Your identity verification was submitted successfully.",
  "timestamp": "2026-09-17T12:00:00.000Z"
}
```
On failure, `status` is `"FAILURE"` and `errorCode`/`httpStatus` may be present alongside `message`.

Keep the SDK's iframe/WebView mounted and listening for messages until `SDK_FLOW_RESULT` arrives.

## Flutter Integration

### Dependency

```yaml
dependencies:
  webview_flutter: ^4.0.0
```

### Step 1 — Define SDK URL and origin

```dart
final String sdkUrl = 'https://{env}-virtualcard.simplifipay.com/simplifi-sdk/';
final String sdkOrigin = '${Uri.parse(sdkUrl).scheme}://${Uri.parse(sdkUrl).host}';
```

`sdkOrigin` scopes all postMessage communication to the SDK domain only. Derive it from `sdkUrl` — never hardcode it separately.

### Step 2 — Set up WebViewController

All parts below are required. Missing any one will break the integration.

```dart
late final WebViewController controller;

controller = WebViewController()
  // Required: enable JavaScript
  .setJavaScriptMode(JavaScriptMode.unrestricted)

  // Required: register named channel — SDK sends all messages through this
  .addJavaScriptChannel(
    'SimplifiSDKChannel',
    onMessageReceived: _onMessage,
  )

  // Required: allow navigation outside the SDK's own domain
  .setNavigationDelegate(NavigationDelegate(
    onNavigationRequest: (request) => NavigationDecision.navigate,

    // Required: re-inject the bridge after every page load, not just the first
    onPageFinished: (_) async {
      await controller.runJavaScript('''
        window.addEventListener("message", function(e) {
          if (e.origin !== "$sdkOrigin") return;
          try {
            var d = typeof e.data === "string" ? e.data : JSON.stringify(e.data);
            SimplifiSDKChannel.postMessage(d);
          } catch(err) {}
        });
      ''');
    },
  ));
```

> **Note:** Don't clear cookies, `sessionStorage`, or local storage for this WebView between navigations — doing so mid-flow will strand the user.

### Step 3 — Load the SDK

Always append `?platform=webview` to the URL. This tells the SDK it is running inside a WebView and to use `SimplifiSDKChannel` instead of `window.parent`/`window.open`. Without this param, KYC's provider handoff will not work in a WebView and no events will be sent.

```dart
controller.loadRequest(Uri.parse('$sdkUrl?platform=webview'));
```

Call this after the controller is fully configured in Step 2.

### Step 4 — Handle incoming SDK messages

```dart
void _onMessage(JavaScriptMessage message) {
  try {
    final data = jsonDecode(message.message) as Map<String, dynamic>;

    // Always verify source before processing
    if (data['source'] != 'simplifi-sdk') return;

    switch (data['eventName']) {
      case 'SIMPLIFI_SDK_READY':
        // SDK has mounted and is waiting for config.
        // Call _sendConfig() now — never before this event.
        _sendConfig();
        break;

      case 'SIMPLIFI_SDK_CONFIG_ACK':
        // SDK accepted the config and is initialising.
        // Hide your loading spinner here.
        break;

      case 'SIMPLIFI_SDK_CONFIG_ERROR':
        // SDK rejected the config payload sent in Step 5.
        // data['message']   → human-readable reason
        // data['errorCode'] → 'INVALID_SDK_CONFIG'
        // Show an error state to the user.
        break;

      case 'SDK_FLOW_RESULT':
        // The KYC flow completed.
        // data['status']  → 'SUCCESS' or 'FAILURE'
        // data['flow']    → 'INITIATE_KYC'
        // data['message'] → human-readable result
        //
        // On FAILURE, check data['errorCode'] first:
        //   50002 → the user is already verified. Show your own "verified"
        //           state rather than an error message.
        //   any other value → show data['message'] to the user.
        break;
    }
  } catch (_) {}
}
```

### Step 5 — Send config to the SDK

Only called from inside the `SIMPLIFI_SDK_READY` handler. Never send config before the SDK signals ready — the message will be lost.

```dart
bool _configSent = false;

Future<void> _sendConfig() async {
  if (_configSent) return;
  _configSent = true;

  final msg = jsonEncode({
    'source':    'simplifi-parent',   // must be exactly this string
    'eventName': 'SIMPLIFI_SDK_CONFIG',
    'payload': {
      'token':     token,             // Admin-scoped JWT minted server-side via the SimpliFi Auth API
      'userId':    userId,            // 36-char user UUID
      'action':    'initiate_kyc',
    },
  });

  await controller.runJavaScript('window.postMessage($msg, "$sdkOrigin");');
}
```

### Step 6 — Timeout handling (Optional)

The SDK should signal `SIMPLIFI_SDK_READY` within a few seconds of the initial load. Because KYC involves multiple redirects afterward, don't apply this same short timeout to the whole flow — only to the initial readiness handshake.

```dart
Timer? _readyTimeout;

// Start timer before loading SDK (Step 3)
_readyTimeout = Timer(const Duration(seconds: 10), () {
  if (_configSent) return;
  // SDK never became ready — show error state to user
});

// Cancel timer when SIMPLIFI_SDK_CONFIG_ACK is received (Step 4)
_readyTimeout?.cancel();
```

## Web Integration

To integrate this into a website (React, Vue, or plain HTML), use a standard **HTML iframe**.

1. **Create Element:** Add an `<iframe>` tag to your page structure.
1. **Styling:** Ensure the iframe has sufficient height (approx. 600px) and width (100%).
1. **Popups:** The KYC flow opens the provider in a popup window. Make sure your page doesn't block popups triggered by a user gesture inside the iframe (most browsers allow this by default since it follows a click on "Get started" inside the SDK).

```html
<iframe id="simplifi-sdk-frame"
        style="width: 100%; height: 600px; border: none;"
        allow="clipboard-write">
</iframe>
```

```javascript
const sdkUrl    = 'https://{env}-virtualcard.simplifipay.com/simplifi-sdk/';
const sdkOrigin = new URL(sdkUrl).origin;
const iframe    = document.getElementById('simplifi-sdk-frame');

iframe.src = sdkUrl;

window.addEventListener('message', (e) => {
  if (e.origin !== sdkOrigin) return;
  try {
    const data = typeof e.data === 'string' ? JSON.parse(e.data) : e.data;
    if (data.source !== 'simplifi-sdk') return;

    if (data.eventName === 'SIMPLIFI_SDK_READY') {
      iframe.contentWindow.postMessage(JSON.stringify({
        source:    'simplifi-parent',
        eventName: 'SIMPLIFI_SDK_CONFIG',
        payload:   {
          token: 'YOUR_ACCESS_TOKEN',
          userId: 'YOUR_USER_ID',
          action: 'initiate_kyc',
        }
      }), sdkOrigin);
    }

    if (data.eventName === 'SDK_FLOW_RESULT') {
      console.log(data.flow, data.status);
    }
  } catch (_) {}
});
```

> **If the popup is blocked:** the flow still completes, but the user ends up on a SimpliFi-hosted success screen ("You can close this window") instead of returning to your site directly.

## Security Notes

- Always set `targetOrigin` to the SDK origin when calling `postMessage` — never `"*"`.
- Always validate `e.origin` on all incoming messages before processing.
- `?platform=webview` is required for Flutter/native — without it the KYC handoff cannot complete in a WebView context.
- Never log the `token` field from the config payload.
