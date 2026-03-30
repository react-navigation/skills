# Dynamic Linking

Use this file when the navigator uses JSX and `NavigationContainer`.

## Detect

Treat the navigator as dynamic when the app renders `<NavigationContainer>`, `<Stack.Navigator>`, `<Tab.Navigator>`, or similar JSX-based navigation trees.

## Goal

Configure the container-level `linking` prop, map paths to the navigator tree, and keep the URL prefixes aligned with the platform setup.

## Behavior

- The `prefixes` list defines which schemes and domains the app should accept.
- `config.screens` mirrors the navigator tree and maps each route name to a path.
- `parse`, `stringify`, and `exact` control param conversion and nested matching.
- `fallback` controls what the user sees while React Navigation resolves the initial URL.
- `prefixes` is mostly a native concern; on web the browser host is inferred from the current site URL.
- If the app uses Expo Go, feed the runtime URL from `Linking.createURL('/')` rather than hard-coding a scheme.

## Setup

1. Build the `prefixes` array from the platform-specific scheme or domain.
2. Pass the linking object to `NavigationContainer`.
3. Mirror the navigator tree in `config.screens`.
4. Add nested `screens` objects for nested navigators.
5. Add `parse`, `stringify`, and `exact` where route matching needs it.
6. Use `fallback` if the app needs a loading UI while the initial URL is resolved.

```tsx
const linking = {
  prefixes: ['myapp://', 'https://app.example.com'],
  config: {
    screens: {
      Home: '',
      Profile: {
        path: 'user/:id',
        parse: {
          id: Number,
        },
      },
      Settings: {
        path: 'settings',
      },
      NotFound: '*',
    },
  },
};

function App() {
  return (
    <NavigationContainer linking={linking} fallback={<Text>Loading...</Text>}>
      <RootNavigator />
    </NavigationContainer>
  );
}
```

```tsx
const linking = {
  prefixes: ['myapp://', 'https://app.example.com'],
  config: {
    screens: {
      HomeTabs: {
        path: 'home',
        screens: {
          Feed: '',
          Profile: 'profile',
        },
      },
    },
  },
};
```

## Caveats

- `config.screens` has to reflect nested navigators accurately, or deep links will resolve to the wrong branch.
- If you use Expo Go, the prefix should come from `Linking.createURL('/')` so the scheme matches the runtime environment.
- If your app supports universal or app links, the same domain must appear in the platform setup and the React Navigation prefixes.
- In 8.x, the `prefixes` option is optional if you only want to accept the default matching behavior, but it is still the right place to restrict accepted URLs.

## Verification

- Open a custom scheme URL and confirm the correct screen opens.
- Open a nested URL and confirm the nested navigator receives the right route.
- Test query strings and route params and confirm they arrive as screen params.
- Test a cold start and a warm start.
- Test on web if the app supports web URLs.

## Common Pitfalls

- Mapping a route in `config.screens` that does not exist in the actual navigator tree.
- Forgetting nested `screens` entries for nested navigators.
- Leaving out `fallback` and showing a blank screen during initial URL resolution.
- Using the wrong prefix for Expo Go versus standalone or native builds.
