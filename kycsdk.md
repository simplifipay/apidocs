# SimpliFi KYC SDK Integration Guide

# Overview

The SimpliFi KYC SDK lets you launch identity verification (via the Mawarid provider) from inside your app without building any verification UI yourself. The host loads a secure SimpliFi page (iframe/popup on web, WebView on native), sends it a config over `postMessage`, and receives an `SDK_FLOW_RESULT` when verification succeeds or fails. The details review, provider handoff, processing, and result screens are all handled by the SDK.

**What the user sees**

```
Review details → Provider (ID scan + selfie) → Processing → Identity verified → OTP → Set MPIN → Done
```

After **Identity verified**, the user can tap **Secure my account** to set an MPIN (a one-time code is sent first) or **Skip for now**. Users who already have an MPIN go straight from Processing to Done.

**Prerequisites**

- The user exists and has a valid phone number that can receive a one-time code.
- The user's `kycStatus` is `READY_FOR_KYC` or `RESUBMIT`. An already-approved user gets a `FAILURE` result with error code `50002` (shown to them as "You're already verified").
- Your company has the Mawarid KYC configuration enabled.
- The provider's callback URL is registered to the SDK origin and ends in `/kyc-callback`.

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

**Config Payload**

Sent to the SDK after it signals readiness via `SIMPLIFI_SDK_READY`.

| Field         | Type   | Required | Description |
|-------------- | ------ | -------- | ----------- |
| **token**     | String | Yes      | Either an admin-scoped JWT for this specific user, minted server-side by **your backend** via the [Auth API](https://ss-docs.simplifipay.com/login-to-generate-sdk-admin-jwt-token-43566163e0), or the customer's own regular session/login token. The admin-scoped JWT is valid for **5 minutes**, so you must also handle [token refresh](#token-refresh). A customer token must belong to the same user as `userId`. |
| **userId**    | String | Yes      | The unique 36-character ID of the user to verify. |
| **action**    | String | Yes      | Must be `initiate_kyc`. |
| **primaryColor**, **accentColor**, **backgroundColor**, **textColor** | String | No | Hex colours (for example `#2E7DFC`) to match your brand. Defaults to the SimpliFi theme. |
| **borderRadius** | String | No | CSS radius for buttons and cards, for example `12px`. |
| **returnUrl** | String | No | Web only. Where the user lands if the provider redirect has to use the full page instead of a popup. |

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
| `SDK_FLOW_RESULT`           | `simplifi-sdk`    | Verification result — check `status` (`SUCCESS`/`FAILURE`) and `flow` |
| `SDK_REQUEST_FRESH_TOKEN`   | `simplifi-sdk`    | The token expired — reply with `SDK_FRESH_TOKEN_RESULT` (see [Token refresh](#token-refresh)) |

### Host → SDK

| eventName             | source            | Description |
| --------------------- | ----------------- | ----------- |
| `SIMPLIFI_SDK_CONFIG` | `simplifi-parent` | Send config payload to SDK |
| `SDK_FRESH_TOKEN_RESULT` | `simplifi-parent` | Reply to `SDK_REQUEST_FRESH_TOKEN` with a fresh token |

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
  "message": "Your identity has been verified.",
  "timestamp": "2026-09-17T12:00:00.000Z"
}
```
On failure, `status` is `"FAILURE"` and `errorCode`/`httpStatus` may be present alongside `message`.

`SUCCESS` is sent as soon as the verification is approved. It means the user's identity is verified, not that they are finished: if they have no MPIN, the SDK keeps running for the OTP and MPIN steps. **Do not close the iframe/WebView on `SUCCESS`.** Let the user dismiss it from the final screen, or close it when they navigate away. Keep it mounted and listening for messages until then.

### Token refresh

The admin-scoped token expires after 5 minutes, which is shorter than a verification (the user leaves for the provider and comes back). When it has expired, the SDK sends `SDK_REQUEST_FRESH_TOKEN`. Your host must mint a new admin-scoped token server-side for the **same user** and reply within 15 seconds. Without a handler, the user sees "Couldn't refresh your session" after returning from the provider.

SDK → host:
```json
{ "source": "simplifi-sdk", "eventName": "SDK_REQUEST_FRESH_TOKEN", "requestId": "…" }
```

Host → SDK (send `error` instead of `token` if you cannot mint one):
```json
{ "source": "simplifi-parent", "eventName": "SDK_FRESH_TOKEN_RESULT", "requestId": "…", "token": "NEW_TOKEN" }
```

The SDK rejects a token that belongs to a different user. A customer token usually lasts much longer, so this reply is rarely needed with one.

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

**Android requirements.** The provider scans the user's ID with the camera and lets them upload photos, so on Android:

- Declare `android.permission.CAMERA` and `android.permission.INTERNET` in `AndroidManifest.xml`.
- Request the camera permission at runtime and grant the WebView's permission request.
- Support the file chooser, for example with `AndroidWebViewController.setOnShowFileSelector`. Without it the upload buttons do nothing.

**How the provider redirect works.** In a WebView the SDK does not open a popup: it redirects the whole page to the provider, then back to the SDK's `/kyc-callback` on the **same origin as the SDK**. Do not intercept those navigations, and allow the provider's host and the SDK origin if your WebView restricts navigation.

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

      case 'SDK_REQUEST_FRESH_TOKEN':
        // Token expired. Mint a new one server-side for the same user
        // and reply with _sendFreshToken(data['requestId']).
        break;

      case 'SDK_FLOW_RESULT':
        // The verification finished.
        // data['status']  → 'SUCCESS' or 'FAILURE'
        // data['flow']    → 'INITIATE_KYC'
        // data['message'] → human-readable result
        // On SUCCESS keep the WebView open: the user may still set an MPIN.
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

### Step 6 — Reply to token refresh requests

```dart
Future<void> _sendFreshToken(String requestId) async {
  final freshToken = await mintAdminToken(userId); // your backend call
  final msg = jsonEncode({
    'source':    'simplifi-parent',
    'eventName': 'SDK_FRESH_TOKEN_RESULT',
    'requestId': requestId,
    'token':     freshToken,
  });
  await controller.runJavaScript('window.postMessage($msg, "$sdkOrigin");');
}
```

### Step 7 — Timeout handling (Optional)

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

    if (data.eventName === 'SDK_REQUEST_FRESH_TOKEN') {
      // mint a new admin-scoped token for the same user on your backend, then:
      iframe.contentWindow.postMessage(JSON.stringify({
        source:    'simplifi-parent',
        eventName: 'SDK_FRESH_TOKEN_RESULT',
        requestId: data.requestId,
        token:     'NEW_TOKEN',
      }), sdkOrigin);
    }

    if (data.eventName === 'SDK_FLOW_RESULT') {
      console.log(data.flow, data.status);
    }
  } catch (_) {}
});
```

> **If the popup is blocked:** the SDK falls back to redirecting the whole frame to the provider and returns to the SDK, or to your `returnUrl` if you set one. If the user closes the popup without finishing, the SDK shows a "Verification window closed" screen with a retry button.

## Security Notes

- Always set `targetOrigin` to the SDK origin when calling `postMessage` — never `"*"`.
- Always validate `e.origin` on all incoming messages before processing.
- `?platform=webview` is required for Flutter/native — without it the KYC handoff cannot complete in a WebView context.
- Never log the `token` field from the config payload.
