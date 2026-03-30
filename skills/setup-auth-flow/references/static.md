# React Navigation Auth Flow: Static Config

Use this file only when the navigator uses static config.

## Goal

Set up signed-in and signed-out navigation with static groups while keeping loading outside the navigation tree, preserving reset behavior for shared screens, and handling deep links to protected screens when needed.

## When

1. The app uses `createXNavigator({ screens, groups, ... })` or `createStaticNavigation`.
2. Auth state is available through a boolean hook such as `useIsSignedIn`.
3. The app needs different route sets for signed-in and signed-out users.
4. The app may need to recover a deep link that points to a protected screen after login.

## Structure

1. Keep splash or bootstrap UI outside the navigator, before `<NavigationContainer>` or `<Navigation>`.
2. Put signed-in and signed-out routes in separate `groups`.
3. Use `if` hooks to show only the group that matches the current auth state.
4. Duplicate any shared screen in both groups if it should reset when auth state changes.
5. Keep public and protected screens in the same navigator tree only when they are actual navigable routes.
6. If a protected deep link should be restored after sign-in, set `UNSTABLE_routeNamesChangeBehavior: 'lastUnhandled'` on 7.x or `routeNamesChangeBehavior: 'lastUnhandled'` on 8.x.

## Workflow

### 1. Gate loading first

Auth bootstrap or token restoration should happen before rendering navigation.

```tsx
const App = () => {
  const isLoading = useAuthBootstrap();

  if (isLoading) {
    return <SplashScreen />;
  }

  return <Navigation />;
};
```

### 2. Split signed-in and signed-out routes

Use groups for each auth state.

```tsx
const RootStack = createNativeStackNavigator({
  UNSTABLE_routeNamesChangeBehavior: 'lastUnhandled',
  groups: {
    SignedIn: {
      if: useIsSignedIn,
      screens: {
        Home: HomeScreen,
        Profile: ProfileScreen,
        Help: HelpScreen,
      },
    },
    SignedOut: {
      if: useIsSignedOut,
      screens: {
        SignIn: SignInScreen,
        SignUp: SignUpScreen,
        Help: HelpScreen,
      },
    },
  },
});
```

On 8.x, rename `UNSTABLE_routeNamesChangeBehavior` to `routeNamesChangeBehavior`.

### 3. Reset shared screens by duplicating them

If a route should clear its state when the user signs in or out, place it in both groups instead of relying on `navigationKey`.

Use this for routes like `Help`, `Settings`, or other screens that should remain available but start fresh after an auth transition.

If you need a reusable post-auth route, use the same route name in both groups and let the group key force a remount.

### 4. Restore deep links after auth

If the user opens a protected deep link while signed out, the navigator can remember the missing route and continue to it after sign-in when `UNSTABLE_routeNamesChangeBehavior: 'lastUnhandled'` is enabled in 7.x or `routeNamesChangeBehavior: 'lastUnhandled'` is enabled in 8.x.

Use this when the app should send the user to the target screen after the sign-in flow completes instead of dropping them on the default signed-in screen.

## Notes

- Do not put splash screens inside the navigator tree.
- Do not use a group for non-navigable loading UI.
- Prefer a single navigator with conditions over swapping whole navigators so login/logout transitions remain smooth.
- Keep this skill focused on auth gating and route structure, not on deep linking or TypeScript setup.

## Verification

- Confirm the splash screen disappears before navigation renders.
- Confirm signed-out screens are not reachable when signed in, and signed-in screens are not reachable when signed out.
- Confirm shared screens remount when auth changes.
- Confirm the back button does not return to auth screens after sign-in.
- If `UNSTABLE_routeNamesChangeBehavior` is enabled, confirm a protected deep link opens the target screen after sign-in.
