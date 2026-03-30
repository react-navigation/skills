# Expo Deep Linking

Use this file when the app is Expo-managed or the native URL scheme is configured through Expo app config.

## Detect

Treat the repo as Expo when you find `app.json` or `app.config.*` with an `expo` block, or when the project depends on Expo tooling such as `expo`, `expo-router`, or `expo-linking`.

## Goal

Register the app URL scheme in Expo, derive the runtime prefix with `expo-linking`, and keep the native config and React Navigation prefixes aligned.

## Setup

1. Add a `scheme` to the Expo app config.
2. Install `expo-linking` and use `Linking.createURL('/')` instead of hard-coding the runtime prefix.
3. Add your HTTPS domain to the linking prefixes if you use universal links.
4. Configure iOS associated domains and Android intent filters in Expo config when you want the app to open from universal links or app links.
5. Rebuild the app after changing scheme, entitlements, or intent filters.

```json
{
  "expo": {
    "scheme": "myapp",
    "ios": {
      "associatedDomains": ["applinks:app.example.com"]
    },
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "category": ["BROWSABLE", "DEFAULT"],
          "data": [
            {
              "scheme": "https",
              "host": "app.example.com"
            }
          ]
        }
      ]
    }
  }
}
```

```tsx
import * as Linking from 'expo-linking';

const prefix = Linking.createURL('/');

const linking = {
  prefixes: [prefix, 'https://app.example.com'],
};
```

## Caveats

- `scheme` only applies to standalone or dev-client builds. Expo Go uses a different runtime URL, which is why `Linking.createURL('/')` is required.
- Universal links need both the Expo app config and a matching website association setup.
- `prefixes` is a React Navigation concern; keep the Expo-native config and the linking config in sync, but do not duplicate the same setup logic in both places.
- If you change `scheme`, associated domains, or intent filters, reinstall the app or rebuild the native binary before testing.

## Verification

- Open a custom scheme on iOS and Android and confirm the app launches into the expected screen.
- Test a universal link or app link and confirm the app opens instead of the browser.
- Test both a cold start and a warm start; the initial URL and an already-running app should both resolve.
- If the app runs in Expo Go, verify the runtime prefix comes from `Linking.createURL('/')` and not a hard-coded scheme.

## Common Pitfalls

- Hard-coding `exp://` or a standalone-only URL.
- Adding a domain to `prefixes` without configuring the app association files.
- Forgetting to rebuild after editing Expo-native link settings.
- Configuring the prefix correctly but not forwarding the native link into React Navigation in the linking file.
