# React Navigation Auth Flow: Dynamic Config

Use this file only when the navigator uses JSX-based dynamic config.

## Goal

Set up signed-in and signed-out navigation with conditional screen trees while keeping loading outside the navigation tree, using `navigationKey` where a shared screen must reset, and handling protected deep links when needed.

## When

1. The app uses `<Stack.Navigator>`, `<Tab.Navigator>`, or similar JSX-based navigation.
2. Auth state is available through a boolean hook such as `useIsSignedIn`.
3. The app needs different route sets for signed-in and signed-out users.
4. The app may need to recover a deep link that points to a protected screen after login.

## Structure

1. Keep splash or bootstrap UI outside the navigator, before `<NavigationContainer>`.
2. Render signed-in and signed-out screen sets conditionally.
3. Use `navigationKey` on shared screens when the screen should reset after auth changes.
4. Keep protected screens out of the signed-out branch.
5. Use `UNSTABLE_routeNamesChangeBehavior: 'lastUnhandled'` on 7.x or `routeNamesChangeBehavior: 'lastUnhandled'` on 8.x if the app needs to remember a protected deep link until auth completes.

## Workflow

### 1. Gate loading first

Restore auth state before rendering the navigator.

```tsx
const App = () => {
  const isLoading = useAuthBootstrap();

  if (isLoading) {
    return <SplashScreen />;
  }

  return <Navigation />;
};
```

### 2. Render auth-specific screen trees

Use conditional screen trees for signed-in and signed-out users.

Prefer one navigator with conditional screens over two separate navigators. That keeps the auth transition inside the same navigation state and avoids losing transition behavior.

```tsx
function Navigation() {
  const isSignedIn = useIsSignedIn();

  return (
    <NavigationContainer>
      <Stack.Navigator
        // Add UNSTABLE_routeNamesChangeBehavior (7.x) or routeNamesChangeBehavior (8.x)
        // here if you need to preserve
        // an unhandled deep link through the auth flow.
      >
        {isSignedIn ? (
          <>
            <Stack.Screen name="Home" component={HomeScreen} />
            <Stack.Screen name="Profile" component={ProfileScreen} />
          </>
        ) : (
          <>
            <Stack.Screen name="SignIn" component={SignInScreen} />
            <Stack.Screen name="SignUp" component={SignUpScreen} />
            <Stack.Screen name="ResetPassword" component={ResetPasswordScreen} />
          </>
        )}
        <Stack.Screen
          name="Help"
          component={HelpScreen}
          navigationKey={isSignedIn ? 'user' : 'guest'}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### 3. Reset shared screens when auth changes

If a screen should be recreated when the auth state flips, render it outside the auth branches and give it a `navigationKey` that changes with the auth branch.

```tsx
<Stack.Navigator>
  {isSignedIn ? (
    <>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Profile" component={ProfileScreen} />
    </>
  ) : (
    <>
      <Stack.Screen name="SignIn" component={SignInScreen} />
      <Stack.Screen name="SignUp" component={SignUpScreen} />
    </>
  )}
  <Stack.Screen
    name="Help"
    component={HelpScreen}
    navigationKey={isSignedIn ? 'user' : 'guest'}
  />
</Stack.Navigator>
```

If several shared screens should reset together, wrap them in a `Stack.Group` and put `navigationKey` on the group instead of each screen.

### 4. Restore deep links after auth

If the user opens a protected deep link while signed out, use `UNSTABLE_routeNamesChangeBehavior: 'lastUnhandled'` on 7.x or `routeNamesChangeBehavior: 'lastUnhandled'` on 8.x so React Navigation can continue to the target route after sign-in.

Use this when the user should end up on the intended protected screen instead of the default signed-in screen.

## Notes

- Do not put splash screens inside the navigator tree.
- Prefer a single navigator with conditions over swapping whole navigators so login/logout transitions remain smooth.
- Keep this skill focused on auth gating and route structure, not on deep linking or TypeScript setup.

## Verification

- Confirm the splash screen disappears before navigation renders.
- Confirm signed-out screens are not reachable when signed in, and signed-in screens are not reachable when signed out.
- Confirm shared screens remount when auth changes.
- Confirm the back button does not return to auth screens after sign-in.
- If `UNSTABLE_routeNamesChangeBehavior` or `routeNamesChangeBehavior` is enabled, confirm a protected deep link opens the target screen after sign-in.
