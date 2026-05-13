# Overview

The SimpliFi Virtual Card SDK allows you to securely display sensitive card information (PAN, CVV) and manage PINs within your application. To ensure PCI-DSS compliance, these details are rendered directly from SimpliFi servers via a secure web page.

**Base URL:** `https://{env}-virtualcard.simplifipay.com/simplifi-sdk/`

### Parameters through postMessage

The SDK accepts credentials via postMessage. The message is delivered to the secure page using the browser `postMessage` API after the iframe/web view has loaded. Send a JSON object to the `contentWindow` once the `load` event has fired:

**Payload**

| Field        | Type   | Required | Description |
|------------- | ------ | -------- | ----------- |
| **token**    | String | Yes      | Bearer token obtained from the [Auth API](https://bayzat-docs.simplifipay.com/login-to-generate-jwt-token-31439162e0). |
| **cardID**   | String | Yes      | The unique 36-character ID of the card to display. |
| **userID**   | String | Yes      | The unique 36-character ID of the cardholder. |
| **action**   | String | Yes      | The operation to perform (see list below). |

**Available Actions**

| Action               | Description |
| -------------------- | ----------- |
| **view_card_detail** | Reveal Card Number & CVV |
| **set_pin**          | Set a new PIN for Physical Card |
| **activate_card**    | Activate a new physical Card |

## Mobile Integration (React Native)
To integrate the SimpliFi Virtual Card SDK into a React Native application, you should use the official **react-native-webview package** (or a similar web view component). The SDK works by loading a secure web page directly from SimpliFi servers via a secure web page to ensure PCI-DSS compliance.

- Install Dependency
  Install the package in your project: `npm install react-native-webview`
- Build the URL

```javascript
const constructUrl = (env) =>{
  return `https://${env}-virtualcard.simplifipay.com/simplifi-sdk/`;
};
```

## Render WebView Component
Use the `WebView` component from the package, passing the constructed URL to the `source` prop and post the config. The secure page relies on JavaScript to render data, which the `react-native-webview` component generally supports by default.

```javascript
import React from 'react';
import { View, StyleSheet } from 'react-native';
import { WebView } from 'react-native-webview';
const MyVirtualCardScreen = () => {
  const webViewRef = useRef(null);
  // **Replace with your actual dynamic values**
  const env = 'prod';
  const action = 'view_card_detail'; // or set_pin, activate_card
  const cardUrl = constructUrl(env);
  const config = {
    token: "YOUR_ACCESS_TOKEN",
    cardId: "YOUR_CARD_UUID",
    userId: "YOUR_USER_ID",
    action,
  };
  
  // Runs inside the WebView once the page is loaded. Dispatches a post`message`
  // event that the SimpliFi SDK page listens for via window.addEventListener("message", ...).
  const handleLoadEnd = () =>
  {
    const payload = Buffer.from(JSON.stringify(config)).toString("base64");
      const injected = `
        window.postMessage(atob('${payload}'), '*');
        true;
      `;
      webViewRef.current?.injectJavaScript(injected);
  };
  
  return (
    <View style={styles.container}>
      <WebView
        ref={webViewRef}
        source={{ uri: cardUrl }}
        style={styles.webview}
        onLoadEnd={handleLoadEnd}
        javaScriptEnabled
      />
    </View>
  );
};
const styles = StyleSheet.create({
	container: {
		flex: 1,
	},
	webview: {
		flex: 1,
		minHeight: 600
	}
});
export default MyVirtualCardScreen;
```

## Web Integration
To integrate this into a website (React, Vue, or plain HTML), use a standard **HTML iFrame**.
1. **Create Element:** Add an `<iframe>` tag to your page structure.
1. **Styling:** Ensure the iframe has sufficient height (approx. 600px) and width (100%) to display the card visually properly. No additional headers or authentication logic is needed inside the browser code; the token in the URL handles validation.
1. **Set Source:** Set the `src` attribute of the iframe to the URL constructed above.
1. **Build Config JSON:** Add params (token, cardId, userId, action)
1. **Send POST Message:** Send the post message after the iframe loads.

```javascript
iframe.onload = () => {
  const message = JSON.stringify({
    token: "YOUR_ACCESS_TOKEN",
    cardId: "YOUR_CARD_UUID",
    userId: "YOUR_USER_ID",
    action: "view_card_detail" //or any other actions defined above
  });
  iframe.contentWindow.postMessage(message, cardUrl);
};
```
