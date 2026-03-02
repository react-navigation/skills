# React Navigation 8.x Static Config Migration

Use this file only when `@react-navigation/native` is on `8.x`.

## Goal

Convert React Navigation navigators from JSX-based dynamic setup to static configuration while preserving behavior, typing, and deep links.

## When

1. You are migrating screens to the static API in React Navigation 8.x.
2. The navigator's screen list is static and not built at runtime.
3. The navigator doesn't use dynamic variables or props that are not available in static config.

## Prerequisites

- The project is using React Navigation 8.x.

## Root Typing

Use module augmentation of `@react-navigation/core` with the root navigator type:

```tsx
import { createStaticNavigation } from "@react-navigation/native"

const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
  },
})

type RootStackType = typeof RootStack

declare module "@react-navigation/core" {
  interface RootNavigator extends RootStackType {}
}

const Navigation = createStaticNavigation(RootStack)
```

Place the augmentation in the same file where `createStaticNavigation` is used for the root navigator.

## Structure

1. Create a static navigator with `createXNavigator({ screens, groups, ... })`.
2. Each `screens` entry can be a component, a nested static navigator, or a screen config object.
3. `groups` define shared options and conditional rendering using `if`, and contain their own `screens`.
4. Screen config objects accept the same options as the dynamic `Screen` API, plus static-only additions such as `linking` and `if`.
5. When a screen needs a config object, use the navigator's `createXScreen` helper such as `createNativeStackScreen`. Verify helper names per navigator.
6. A screen config `linking` can be a string path or an object with `path`, `parse`, `stringify`, and `exact`.

## Workflow

### 1. Identify static candidates

A navigator is a static candidate if all its screens are known at build time. Look for:

- **Convertible**: fixed `<Stack.Screen>` elements, conditional rendering based on auth or feature flags (use `if` hooks), render callbacks passing extra props (use React context), navigators wrapped in providers (use `layout`)
- **Not convertible**: screen list built from runtime data such as mapping over an API response, screens added or removed based on values that can't be expressed as a hook returning a boolean, navigator props computed at runtime such as dynamic `initialRouteName` or `screenOptions` derived from props or state that aren't available in static config

### 2. Convert navigator JSX to static config

Before:

```tsx
const Stack = createNativeStackNavigator()

function RootStack() {
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen
        name="Profile"
        component={ProfileScreen}
        options={{ title: "My Profile" }}
      />
    </Stack.Navigator>
  )
}
```

After:

```tsx
const RootStack = createNativeStackNavigator({
  screenOptions: { headerShown: false },
  screens: {
    Home: HomeScreen,
    Profile: {
      screen: ProfileScreen,
      options: { title: "My Profile" },
    },
  },
})
```

When a screen needs a config object, use `createXScreen`.

Full screen config shape:

```tsx
createXScreen({
  screen: ScreenComponent,
  options: ({ route, navigation, theme }) => ({
    title: route.name,
  }),
  listeners: ({ route, navigation }) => ({
    focus: () => {},
  }),
  initialParams: {},
  getId: ({ params }) => params.id,
  linking: {
    path: "pattern/:id",
    parse: { id: Number },
    stringify: { id: (value) => String(value) },
    exact: true,
  },
  if: useConditionHook,
  layout: ({ children }) => children,
})
```

Shorthand (component only, no config): `ScreenName: ScreenComponent`

Nested static navigator: `ScreenName: AnotherStaticNavigator`

### 3. Convert nested navigators

Nested dynamic navigators rendered as components become nested config objects.

Before:

```tsx
const Tab = createBottomTabNavigator()

function HomeTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Groups" component={GroupsScreen} />
      <Tab.Screen name="Chats" component={ChatsScreen} />
    </Tab.Navigator>
  )
}

<Stack.Screen name="Home" component={HomeTabs} />
```

After:

```tsx
const HomeTabs = createBottomTabNavigator({
  screens: {
    Groups: GroupsScreen,
    Chats: ChatsScreen,
  },
})

const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeTabs,
  },
})
```

### 4. Convert auth flows and groups

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    Home: {
      if: useIsSignedIn,
      screen: HomeScreen,
    },
    SignIn: {
      if: useIsSignedOut,
      screen: SignInScreen,
    },
  },
})
```

Use `groups` only when you need different shared options, conditional groups, or grouped linking behavior.

### 5. Convert groups

Before:

```tsx
function RootStack() {
  return (
    <Stack.Navigator>
      <Stack.Group screenOptions={{ headerStyle: { backgroundColor: "red" } }}>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Profile" component={ProfileScreen} />
      </Stack.Group>
      <Stack.Group screenOptions={{ presentation: "modal" }}>
        <Stack.Screen name="Settings" component={SettingsScreen} />
      </Stack.Group>
    </Stack.Navigator>
  )
}
```

After:

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
    Profile: ProfileScreen,
  },
  groups: {
    Modal: {
      screenOptions: { presentation: "modal" },
      screens: {
        Settings: SettingsScreen,
      },
    },
  },
})
```

Top-level `screens` and `screenOptions` handle the default group. Use `groups` only when you need separate `screenOptions`, `if` conditions, or `linking` prefixes.

### 6. No extra props on static screens

Before:

```tsx
<Stack.Screen name="Chat">
  {(props) => <ChatScreen {...props} userToken={token} />}
</Stack.Screen>
```

After, use context or a wrapper:

```tsx
const TokenContext = React.createContext("")

function ChatScreen() {
  const token = React.useContext(TokenContext)
  return <Chat token={token} />
}

const RootStack = createNativeStackNavigator({
  screens: {
    Chat: ChatScreen,
  },
})
```

### 7. Move wrappers to `layout`

If the dynamic navigator was wrapped in a provider or layout component, use the navigator-level `layout` property. `layout` renders inside the navigator, so it is not exactly equivalent to wrapping from outside, but it is the closest static alternative.

Before:

```tsx
function RootStack() {
  return (
    <SomeProvider>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
      </Stack.Navigator>
    </SomeProvider>
  )
}
```

After:

```tsx
const RootStack = createNativeStackNavigator({
  layout: ({ children }) => <SomeProvider>{children}</SomeProvider>,
  screens: {
    Home: HomeScreen,
  },
})
```

### 8. Replace the root container

Before:

```tsx
const linking = {
  prefixes: ["https://example.com", "example://"],
  config: {
    screens: {
      Home: "",
      Profile: "profile/:id",
    },
  },
}

function App() {
  return (
    <NavigationContainer linking={linking} theme={MyTheme}>
      <RootStack />
    </NavigationContainer>
  )
}
```

After:

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
    Profile: {
      screen: ProfileScreen,
      linking: "profile/:id",
    },
  },
})

type RootStackType = typeof RootStack

declare module "@react-navigation/core" {
  interface RootNavigator extends RootStackType {}
}

const Navigation = createStaticNavigation(RootStack)

function App() {
  return (
    <Navigation
      linking={{
        prefixes: ["https://example.com", "example://"],
        enabled: "auto",
      }}
      theme={MyTheme}
    />
  )
}
```

The `Navigation` component accepts `linking` with `prefixes` and `enabled`, plus props such as `theme`, `ref`, `onReady`, `onStateChange`, `onUnhandledAction`, and `documentTitle`.

The `enabled: "auto"` option enables automatic deep links for all screens.

Do not pass a `config` object to root `linking`. Screen-level `linking` replaces it.

### 9. Migrate linking carefully

Omit `linking` on a screen when the default kebab-case path is acceptable. If the path is identical to the auto path such as `Details` to `details`, remove the redundant `linking` entry.

Add `linking` for custom paths or when you need path params with `parse` or `stringify`.

Before:

```tsx
const linking = {
  prefixes: ["https://example.com"],
  config: {
    screens: {
      Home: "",
      Profile: {
        path: "user/:id",
        parse: { id: Number },
      },
      Settings: "settings",
    },
  },
}
```

After:

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    Home: {
      screen: HomeScreen,
      linking: "",
    },
    Profile: {
      screen: ProfileScreen,
      linking: {
        path: "user/:id",
        parse: { id: Number },
      },
    },
    Settings: SettingsScreen,
  },
})
```

Linking paths are auto-generated for leaf screens using kebab-case of the screen name. The first leaf screen gets the path `/` unless you set an explicit empty path on another screen.

To control auto-generated linking, pass `enabled` on the root `linking` prop: `enabled: true` generates paths only for screens with explicit `linking`, and `enabled: false` disables linking entirely.

### 10. Preserve deep-link paths

If a screen previously had a custom path such as `linking: "contacts"` and you remove it, the auto path becomes kebab-case of the screen name such as `TabContacts` to `tab-contacts`. This breaks existing URLs and deep links.

Keep explicit `linking` when you need to preserve existing paths.

If screens containing navigators have `linking` set to `""` or `"/"`, it is usually redundant and can be removed.

### 11. Replace screen prop types

Before:

```tsx
function ProfileScreen({
  navigation,
  route,
}: NativeStackScreenProps<RootStackParamList, "Profile">) {
  const id = route.params.id
  navigation.navigate("Home")
}
```

After:

```tsx
import type { StaticScreenProps } from "@react-navigation/native"

function ProfileScreen({ route }: StaticScreenProps<{ id: string }>) {
  const navigation = useNavigation("Profile")
  const id = route.params.id
  navigation.navigate("Home")
}
```

For screens with no params, omit props entirely or use `StaticScreenProps`.

For navigation access, prefer the screen-name-based hooks:

```tsx
const navigation = useNavigation("Profile")
const route = useRoute("Profile")
```

### 12. Remove manual param lists and avoid circular type traps

Remove all hand-written param-list declarations created only to support dynamic typing.

If a param list is absolutely necessary, derive it from the navigator type:

```tsx
type RootStackType = typeof RootStack
type RootStackParamList = StaticParamList<RootStackType>
```

Keep derived param-list aliases only when they are genuinely consumed. If they are unused, delete them.

The common migration failure is:

1. Derive `StaticParamList<typeof Navigator>`.
2. Reuse that param list to type screen props in screens contained by the same navigator.
3. Create a self-reference through imports.

Avoid this by:

- Using `StaticScreenProps` for `route.params`
- Using hooks for navigation access instead of navigator-specific screen-prop aliases
- Keeping any derived root param type close to the root navigator
- Deleting obsolete shared type files when they become empty

### 13. Handle hook-dependent options and styling

- In `screenOptions`, `options`, and `listeners`, use callback arguments such as `theme`, `route`, and `navigation`
- Do not call hooks directly inside those callbacks
- If an icon or child component needs hooks, move the hook usage into a normal React component used by the callback
- If a wrapper or provider needs hooks, move it into `layout`

Example:

```tsx
screenOptions: ({ theme }) => ({
  contentStyle: {
    backgroundColor: theme.colors.background,
  },
})
```

For cases that previously depended on `useSafeAreaInsets`, verify whether built-in behavior already handles the safe area before reproducing custom height math.

### 14. Watch for module-load timing

Static navigator config is created at module load time, not during component render.

Be careful with:

- `translate(...)`
- context-derived values
- feature-flag values read too early

If the value should resolve later, wrap it in a callback:

```tsx
options: () => ({
  tabBarLabel: translate("tabs:home"),
})
```

## Types

Keep the root augmentation next to the root static navigator and reuse it as the single source of truth.

This enables typed `useNavigation("ScreenName")` and `useRoute("ScreenName")` throughout the app. Screen-name-based hooks are much more capable in static config than dynamic.

Params are inferred from linking path patterns such as `user/:userId` inferring `{ userId: string }`. For screens without `linking`, use `StaticScreenProps<ParamType>` to type params explicitly.

Prefer the screen-name-based hooks over manual param lists and navigator-specific screen-prop aliases.

For the root param list, instead of deriving from the navigator type, import it from `@react-navigation/native`: `import type { RootStackParamList } from '@react-navigation/native'`.

If you still need a param-list alias, derive it from the navigator type with `StaticParamList`, export it from the same file as the navigator, and use `import type` for downstream imports to reduce circular dependencies.

Avoid `any`, non-null assertions, and `as` assertions.

## Auth Migration

To migrate conditional screens from dynamic config, use static `if` hooks. This prevents navigating to protected screens when signed out and unmounts auth screens after sign-in, so the back button cannot return to them.

If you previously used `navigationKey` to reset a screen when auth state changes, duplicate the screen in both auth groups. The group name is used for the key, so switching groups resets the screen. For example, declare `Chat` in both the signed-in and signed-out groups instead of using `navigationKey`.

Loading UI should live in the navigator `layout`, not in a `Loading` screen or group. Keep `screens` and `groups` for actual navigable routes only.

```tsx
const AuthLayout = ({ children }: { children: React.ReactNode }) => {
  const isLoading = useIsLoading()

  if (isLoading) {
    return <SplashScreen />
  }

  return <>{children}</>
}
```

### Dynamic Pattern (before)

```tsx
function App() {
  const isSignedIn = useIsSignedIn()

  return (
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
    </Stack.Navigator>
  )
}
```

### Static Pattern (after)

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    Home: {
      if: useIsSignedIn,
      screen: HomeScreen,
    },
    Profile: {
      if: useIsSignedIn,
      screen: ProfileScreen,
    },
    SignIn: {
      if: useIsSignedOut,
      screen: SignInScreen,
    },
    SignUp: {
      if: useIsSignedOut,
      screen: SignUpScreen,
    },
  },
})
```

## Example

### Dynamic (before)

```tsx
type ExampleParamList = {
  Article: { author: string }
  Albums: undefined
}

const Stack = createNativeStackNavigator<ExampleParamList>()

function ArticleScreen({
  navigation,
  route,
}: NativeStackScreenProps<ExampleParamList, "Article">) {
  return <Button onPress={() => navigation.navigate("Albums")} />
}

function AlbumsScreen() {
  return <Albums />
}

export function Example() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen
          name="Article"
          component={ArticleScreen}
          options={({ route }) => ({ title: route.params.author })}
          initialParams={{ author: "Gandalf" }}
        />
        <Stack.Screen name="Albums" component={AlbumsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  )
}
```

### Static (after)

```tsx
function ArticleScreen({
  route,
}: StaticScreenProps<{
  author: string
}>) {
  const navigation = useNavigation("Article")
  return <Button onPress={() => navigation.navigate("Albums")} />
}

function AlbumsScreen() {
  return <Albums />
}

const ExampleNavigator = createNativeStackNavigator({
  screens: {
    Article: {
      screen: ArticleScreen,
      options: ({ route }) => ({ title: route.params.author }),
      initialParams: { author: "Gandalf" },
      linking: "article/:author",
    },
    Albums: AlbumsScreen,
  },
})

type ExampleNavigatorType = typeof ExampleNavigator

declare module "@react-navigation/core" {
  interface RootNavigator extends ExampleNavigatorType {}
}

const Navigation = createStaticNavigation(ExampleNavigator)

export function Example() {
  return <Navigation />
}
```

## Limitations

- Cannot use React hooks such as `useTheme()` directly in `options` or `listeners` callbacks. Use callback arguments such as `theme` instead. `React.use()` can read context in `options` callbacks but may trigger ESLint warnings.
- The screen list is static. Screens cannot be added or removed at runtime. Use `if` hooks for conditional rendering.
- No render callbacks or extra props on screens. Use React context instead.

## Mixing Static and Dynamic

Apps can combine both APIs when needed:

- When migrating incrementally, start from the root navigator since it drives typing. Keep leaf navigators dynamic until converted. Static navigators nested inside dynamic ones lose many benefits such as type inference and auto linking.
- When a leaf navigator genuinely needs runtime-dynamic screens that `if` cannot express.

## Review

1. No `StackScreenProps`, `NativeStackScreenProps`, `BottomTabScreenProps`, or custom screen-prop aliases remain in static screens. Use `StaticScreenProps<ParamType>`.
2. Root typing uses `declare module "@react-navigation/core"` and `RootNavigator`, and the augmentation lives next to the root static navigation.
3. `createStaticNavigation` replaces `NavigationContainer`.
4. Root `linking` only uses `prefixes` and `enabled`. Screen paths live in screen-level `linking`.
5. Linking config is present only where custom paths or params are required. Defaults are kebab-case.
6. No extra props are passed to static screens. Use React context.
7. When a screen needs a config object, use `createXScreen`.
8. Prefer the screen-name-based hooks over navigator-specific generics.
9. Navigator-level `layout` replaces wrapping navigators in providers.
10. No circular type references were introduced.
11. No obsolete shared type files, empty files, or unused exports remain from the old dynamic navigator.
