# Static Linking

Use this file when the navigator uses static config.

## Detect

Treat the navigator as static when the app uses `createXNavigator({ screens, groups, ... })` or `createStaticNavigation`.

## Goal

Use React Navigation's static path generation, then override only the screens, groups, or prefixes that need custom behavior.

## Behavior

- In React Navigation 7.x, static deep linking is opt-in through `linking={{ enabled: 'auto', prefixes: [...] }}` on `Navigation`.
- In React Navigation 8.x, static deep linking is on by default. The `linking` prop on the static `Navigation` component is narrowed to root-path behavior, and screen or group `linking` drives the actual route mapping.
- Leaf screens get paths from their screen names automatically.
- Screen-level `linking` overrides the generated path for that screen.
- Group-level `linking` applies a shared prefix to nested screens inside the group.
- `exact: true` stops a nested screen from inheriting the parent path.

## Setup

1. Start with the static navigator tree.
2. On 7.x, add `prefixes` only for the schemes or domains the app should accept.
3. On 8.x, keep URL acceptance aligned with the platform setup and use screen/group linking for the route map.
4. Add `linking.path` to the screens that need custom paths.
5. Use `path: ''` for the homepage route.
6. Use `parse` and `stringify` when params need custom serialization.
7. Use `*` for a catch-all screen when you need a not-found route.

```tsx
const RootStack = createNativeStackNavigator({
  groups: {
    Main: {
      linking: {
        path: 'app',
      },
      screens: {
        Home: {
          screen: HomeScreen,
          linking: {
            path: '',
          },
        },
        Profile: {
          screen: ProfileScreen,
          linking: {
            path: 'user/:id',
            parse: {
              id: Number,
            },
          },
        },
      },
    },
    Modals: {
      screens: {
        Settings: {
          screen: SettingsScreen,
          linking: {
            path: 'settings',
          },
        },
      },
    },
  },
});
```

```tsx
const Navigation = createStaticNavigation(RootStack);

function App() {
  return (
    <Navigation
      fallback={<Text>Loading...</Text>}
      linking={{
        // 7.x static config can accept prefixes here.
        prefixes: ['myapp://', 'https://app.example.com'],
      }}
    />
  );
}
```

## Caveats

- Nested paths are relative to their parent unless you use `exact: true`.
- Path generation only applies cleanly to leaf screens; nested navigators still need explicit path handling when you want something other than the default hierarchy.
- If the app still uses Expo Go or a custom scheme, keep the runtime prefixes from the platform-specific file in sync with the static config on 7.x. On 8.x, keep the platform URL setup aligned with the screen-level path map instead.
- Do not use the static config file to solve native URL registration problems; that belongs in the Expo/native file.

## Verification

- Open the homepage path and confirm it lands on the expected root screen.
- Open a custom leaf path and confirm params are parsed correctly.
- Open a nested path and confirm the parent/child path composition is correct.
- Open an unknown path and confirm the not-found route or fallback behavior is correct.
- Test on web and native if the app supports both.

## Common Pitfalls

- Assuming 8.x static config still needs `enabled: 'auto'`.
- Forgetting that nested screen paths are relative by default.
- Matching the path but forgetting to parse or stringify params.
- Adding prefixes that do not match the Expo or native setup.
