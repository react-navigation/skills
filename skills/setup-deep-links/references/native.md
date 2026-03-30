# Native Deep Linking

Use this file when the app is bare React Native or custom native code owns the platform link handling.

## Detect

Treat the repo as native when you find `ios/` and `android/` projects, native app delegates or activities, and no Expo-managed app config driving the scheme.

## Goal

Make the native app open the configured scheme or universal/app links, then pass those URLs through the React Native linking system so React Navigation can resolve them.

## Setup

1. Register a custom URL scheme in the native app.
2. Configure iOS universal links or Android app links if you need HTTPS links.
3. Forward incoming URLs from the native entry points into React Native's linking integration.
4. Use the same scheme and domains in the React Navigation linking prefixes.
5. Rebuild the native app after any scheme, entitlements, or intent-filter change.

```objc
// iOS example: forward URL openings to React Native's linking manager.
- (BOOL)application:(UIApplication *)application
            openURL:(NSURL *)url
            options:(NSDictionary<UIApplicationOpenURLOptionsKey, id> *)options
{
  return [RCTLinkingManager application:application openURL:url options:options];
}
```

```xml
<!-- Android example: intent-filter for a custom scheme or https link -->
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="https" android:host="app.example.com" />
</intent-filter>
```

## Caveats

- A working React Navigation config does not help if the native layer never forwards the URL.
- Universal links and app links also require website association files on the server side.
- If the app uses scenes or a custom activity setup, make sure the URL handling path is covered in every entry point that can receive links.
- Keep the native registration and the React Navigation prefixes aligned, especially if you support both a custom scheme and HTTPS links.

## Verification

- Launch the app from a custom scheme on iOS and Android.
- Launch the app from a universal link or app link and confirm it opens the app instead of the browser.
- Test a cold start and a warm start.
- Check the initial URL and the foreground event path both reach the app.

## Common Pitfalls

- Registering the scheme only in React Navigation but not in native code.
- Updating the native manifest or app delegate without reinstalling the app.
- Forgetting to forward `openURL`/intent handling to the React Native linking layer.
- Using a domain in React Navigation that the native app is not actually authorized to open.
