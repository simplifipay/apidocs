# Overview

The SimpliFi Virtual Card SDK allows you to securely display sensitive card information (PAN, CVV) and manage PINs within your application. To ensure PCI-DSS compliance, these details are rendered directly from SimpliFi servers via a secure web page.

**Base URL:** `https://{env}-virtualcard.simplifipay.com/simplifi-sdk/`

### Parameters through postMessage

The SDK accepts credentials via `postMessage`. The message is delivered securely to the SDK page using the browser `postMessage` API after the SDK has fully loaded and signalled readiness.

**The Handshake**

1. Host loads the SDK URL in an iframe or WebView
1. SDK mounts and sends `SIMPLIFI_SDK_READY` to the host
1. Host receives `SIMPLIFI_SDK_READY` and sends `SIMPLIFI_SDK_CONFIG` with credentials
1. SDK validates the config and sends `SIMPLIFI_SDK_CONFIG_ACK` (accepted) or `SIMPLIFI_SDK_CONFIG_ERROR` (rejected)
1. On flow completion, SDK sends `SDK_FLOW_RESULT`

> **Important:** Never send credentials before receiving `SIMPLIFI_SDK_READY`. The SDK may not have mounted yet and the message will be lost.

**Config Payload**

Sent to the SDK after it signals readiness via `SIMPLIFI_SDK_READY`.

| Field        | Type   | Required | Description |
|------------- | ------ | -------- | ----------- |
| **token**    | String | Yes      | Bearer token obtained from the [Auth API](https://bayzat-docs.simplifipay.com/login-to-generate-jwt-token-31439162e0). |
| **cardID**   | String | Yes      | The unique 36-character ID of the card to display. |
| **userID**   | String | Yes      | The unique 36-character ID of the cardholder. |
| **action**   | String | Yes      | The operation to perform (see list below). |

**Available Actions**

| Action               | Description |
| -------------------- | ----------- |
| **view_card_detail** | Reveal Card Number and CVV |
| **set_pin**          | Set a new PIN for Physical Card |
| **activate_card**    | Activate a new physical Card |

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

| eventName                   | source            | Description |
| --------------------------- | ----------------- | ----------- |
| `SIMPLIFI_SDK_CONFIG`       | `simplifi-parent` | Send config payload to SDK |

**Config message**
```json
{
  "source": "simplifi-parent",
  "eventName": "SIMPLIFI_SDK_CONFIG",
  "payload": {
    "token": "YOUR_TOKEN",
    "cardId": "YOUR_CARD_UUID",
    "userId": "YOUR_USER_UUID",
    "action": "view_card_detail"
  }
}
```

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

All three parts below are required. Missing any one will break the integration.

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

  // Required: inject bridge after page loads so SDK's window.postMessage
  // events are forwarded into SimplifiSDKChannel above
  .setNavigationDelegate(NavigationDelegate(
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

### Step 3 — Load the SDK

Always append `?platform=webview` to the URL. This tells the SDK it is running inside a WebView and to use `SimplifiSDKChannel` instead of `window.parent`. Without this param the SDK will not send any events.

```dart
controller.loadRequest(Uri.parse('$sdkUrl?platform=webview'));
```

Call this after the controller is fully configured in Step 2.

### Step 4 — Handle incoming SDK messages

The SDK sends four events. You must handle all of them.

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
        // SDK rejected the config.
        // data['message']   → human-readable reason
        // data['errorCode'] → machine-readable code
        // Show an error state to the user.
        break;

      case 'SDK_FLOW_RESULT':
        // A card flow completed.
        // data['status']  → 'SUCCESS' or 'FAILURE'
        // data['flow']    → 'ACTIVATE_CARD' | 'SET_PIN'
        // data['message'] → human-readable result
        break;
    }
  } catch (_) {}
}
```

### Step 5 — Send config to the SDK

Only called from inside the `SIMPLIFI_SDK_READY` handler. Never send config before the SDK signals ready — the message will be lost.
Use a flag to ensure config is sent only once per session.

```dart
bool _configSent = false;

Future<void> _sendConfig() async {
  if (_configSent) return;
  _configSent = true;

  final msg = jsonEncode({
    'source':    'simplifi-parent',  // must be exactly this string
    'eventName': 'SIMPLIFI_SDK_CONFIG',
    'payload': {
      'token':  token,   // Bearer token from Auth API
      'cardId': cardId,  // 36-char card UUID
      'userId': userId,  // 36-char user UUID
      'action': action,  // 'view_card_detail' | 'set_pin' | 'activate_card'
    },
  });

  await controller.runJavaScript('window.postMessage($msg, "$sdkOrigin");');
}
```

### Step 6 — Timeout handling (Optional)

The SDK should respond within a few seconds. Add a timeout in case the page fails to load or the WebView is blocked.

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
To integrate this into a website (React, Vue, or plain HTML), use a standard **HTML iFrame**.
1. **Create Element:** Add an `<iframe>` tag to your page structure.
1. **Styling:** Ensure the iframe has sufficient height (approx. 600px) and width (100%) to display the card visually properly. No additional headers or authentication logic is needed inside the browser code; the token in the URL handles validation.

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
          cardId: 'YOUR_CARD_UUID',
          userId: 'YOUR_USER_ID',
          action: 'view_card_detail' //or any other actions defined above
        }
      }), sdkOrigin);
    }

    if (data.eventName === 'SDK_FLOW_RESULT') {
      console.log(data.flow, data.status);
    }
  } catch (_) {}
});
```

## Security Notes

- Always set `targetOrigin` to the SDK origin when calling `postMessage` — never `"*"`.
- Always validate `e.origin` on all incoming messages before processing.
- `?platform=webview` is required for Flutter — without it the SDK will not dispatch any events in a WebView context.
- Never log the `token` field from the config payload.
