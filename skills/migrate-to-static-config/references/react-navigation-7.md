# React Navigation 7.x Static Config Migration

Use this file only when `@react-navigation/native` is on `7.x`.

## Prerequisites

- The project is using React Navigation 7.x.
- Before migrating, check if `@react-navigation/*` packages are on the latest published 7.x version:
  - Run `npm view package-name@latest version` for each `@react-navigation` package to check.
  - If versions are outdated, recommend updating and explain why (newer 7.x versions added static config support and fixes). If the user declines or updates are blocked by project constraints, proceed with current versions and note any features that may be unavailable.

## Official reference

Fetch [llms.txt](https://reactnavigation.org/llms.txt) for a table of contents of all React Navigation documentation. During the migration, when you encounter an API, pattern, or type not fully covered in this reference, find the relevant link in llms.txt and fetch that specific page. Common mappings:
- Custom navigators or routers -- fetch "Custom navigators" and "Custom routers" pages
- Static config API details -- fetch "Static configuration" page
- Type errors or TypeScript issues -- fetch "TypeScript" page
- Screen options for a specific navigator -- fetch that navigator's API page (e.g., "Stack Navigator")

## Structure

1. Create a static navigator with `createXNavigator({ screens, groups, ... })`.
2. Each `screens` entry can be a component, a nested static navigator, or a screen config object.
3. `groups` define shared options and conditional rendering using `if`, and contain their own `screens`.
4. Screen config objects accept the same options as the dynamic `Screen` API, plus static-only additions such as `linking` and `if`.
5. When a screen needs a config object, use a plain screen config object.
6. A screen config `linking` can be a string path or an object with `path`, `parse`, `stringify`, and `exact`.

## Workflow

### Step 0: Check the parent navigator first

**Before classifying or editing any navigator, check whether its parent navigator is static or dynamic.** This is a prerequisite gate — do not skip it.

1. Find the parent navigator that renders this navigator as a `<Screen>` component.
2. If the parent uses JSX-based dynamic config (`<Stack.Navigator>` with `<Stack.Screen>` children, hooks for screenOptions, conditional rendering via JSX expressions), it is dynamic.
3. **If the parent is dynamic, classify this navigator as "keep dynamic"** unless the user explicitly wants to migrate bottom-up. Static navigators nested inside dynamic ones lose automatic linking and TypeScript type inference at the boundary (see "Mixing Static and Dynamic APIs" below).
4. If the user wants to proceed anyway, use the "Dynamic root navigator, static nested navigator" pattern: call `.getComponent()` on the static child and `createPathConfigForStaticNavigation()` for linking.
5. If you are migrating the root navigator, this step does not apply — proceed to classification.

### Classify the navigator

A navigator is a static candidate if all its screens are known at build time. Classify it before editing code:

- **Direct static migration**
  - A fixed set of `<Stack.Screen>`, `<Tabs.Screen>`, or similar screen elements declared explicitly in JSX
  - Nested navigators whose own screen lists are already static
  - Conditional screens or groups controlled by auth state, feature flags, or other boolean conditions
  - Navigator-level `screenOptions`, `initialRouteName`, and group options that do not depend on component-local props or hooks
  - Screen options, params, IDs, and linking config that are already known in module scope
- **Static migration with adaptation**
  - Render callbacks used only to pass extra props or wrappers to a screen
  - Navigators wrapped in providers or components that use hooks to compute navigator-level props
  - Factory functions that generate navigators from a fixed screen list or a fixed set of options
- **Keep dynamic**
  - Screen lists built from runtime data, such as mapping over API responses, server-driven config, or data not known statically
  - Navigation structure that depends on async data before the full route tree can be known
  - Child navigators whose parent navigator must stay dynamic and cannot represent the child as a static screen entry

When a navigator matches both "adaptation" and "keep dynamic" criteria, "keep dynamic" takes precedence.

Start the migration from the root navigator and work downwards. If the root navigator is not a static candidate, abort the migration unless the user explicitly wants to keep the root dynamic and migrate only nested navigators.

### Identify custom navigators

A custom navigator uses the `useNavigationBuilder` hook. Before migration, ensure it uses same patterns for its types as official docs:

```tsx
import * as React from 'react';
import {
  View,
  Text,
  Pressable,
  type StyleProp,
  type ViewStyle,
  StyleSheet,
} from 'react-native';
import {
  createNavigatorFactory,
  CommonActions,
  type DefaultNavigatorOptions,
  type NavigatorTypeBagBase,
  type ParamListBase,
  type StaticConfig,
  type TabActionHelpers,
  type TabNavigationState,
  TabRouter,
  type TabRouterOptions,
  type TypedNavigator,
  useNavigationBuilder,
  type NavigationProp,
} from '@react-navigation/native';

type MyNavigationConfig = {
  // Additional props accepted by the view
};

type MyNavigationOptions = {
  // Supported screen options
};

type MyNavigationEventMap = {
  // Map of event name and the type of data
};

// The type of the navigation object for each screen
type MyNavigationProp<
  ParamList extends ParamListBase,
  RouteName extends keyof ParamList = keyof ParamList,
  NavigatorID extends string | undefined = undefined,
> = NavigationProp<
  ParamList,
  RouteName,
  NavigatorID,
  TabNavigationState<ParamList>,
  MyNavigationOptions,
  MyNavigationEventMap
> &
  TabActionHelpers<ParamList>;

// The props accepted by the component is a combination of 3 things
type Props = DefaultNavigatorOptions<
  ParamListBase,
  string | undefined,
  TabNavigationState<ParamListBase>,
  MyNavigationOptions,
  MyNavigationEventMap,
  MyNavigationProp<ParamListBase>
> &
  TabRouterOptions &
  MyNavigationConfig;

function TabNavigator({ tabBarStyle, contentStyle, ...rest }: Props) {
  /* implementation of the navigator */
}

// Types required for type-checking the navigator
type MyTabTypeBag<ParamList extends {}> = {
  ParamList: ParamList;
  State: TabNavigationState<ParamList>;
  ScreenOptions: MyNavigationOptions;
  EventMap: MyNavigationEventMap;
  NavigationList: {
    [RouteName in keyof ParamList]: MyNavigationProp<ParamList, RouteName>;
  };
  Navigator: typeof TabNavigator;
};

// The factory function with overloads for static and dynamic configuration
export function createMyNavigator<
  const ParamList extends ParamListBase,
  const NavigatorID extends string | undefined = undefined,
  const TypeBag extends NavigatorTypeBagBase = {
    ParamList: ParamList;
    NavigatorID: NavigatorID;
    State: TabNavigationState<ParamList>;
    ScreenOptions: MyNavigationOptions;
    EventMap: MyNavigationEventMap;
    NavigationList: {
      [RouteName in keyof ParamList]: MyNavigationProp<
        ParamList,
        RouteName,
        NavigatorID
      >;
    };
    Navigator: typeof TabNavigator;
  },
  const Config extends StaticConfig<TypeBag> = StaticConfig<TypeBag>,
>(config?: Config): TypedNavigator<TypeBag, Config> {
  return createNavigatorFactory(TabNavigator)(config);
}
```

Then, it can use the same static config API as official navigators:

```tsx
const MyNavigator = createMyNavigator({
  screens: {
    Home: HomeScreen,
    Profile: {
      screen: ProfileScreen,
      options: { title: 'My Profile' },
    },
  },
});
```

The actual implementation of the navigator is not relevant to the migration. The only relevant part is the navigator function (e.g. `createMyNavigator`) and whether it accepts configuration object.

If it doesn't accept a config object, update it to use the `createNavigatorFactory` and navigator API patterns shown above before migration. If it already uses the same patterns, there are no changes needed to the navigator implementation for static config migration.

#### Custom router options in static config

If the custom navigator accepts additional router options beyond the standard set (e.g., `sidebarScreen`, `persistentScreens`, custom layout modes), these options cannot be expressed in the static config object directly. The static config API passes `screens`, `groups`, `screenOptions`, `screenListeners`, and `initialRouteName` to the navigator -- not custom router options.

Options for handling custom router options:
1. **Static values** -- Set them as default values in the navigator implementation itself.
2. **Dynamic values via `.with()`** -- Pass them as navigator props inside a `.with()` wrapper.
3. **Keep dynamic** -- If the custom router options must be computed from hooks or parent context that cannot be accessed via `.with()` + `useRoute()`, keep the navigator dynamic.

When using `.with()` to pass custom router options, the flow is: the `.with()` wrapper passes props to `<Navigator />` → the navigator component receives them → it forwards them to `useNavigationBuilder` options → `useNavigationBuilder` passes non-standard options to the router factory. This means any prop you pass to `<Navigator />` inside `.with()` will reach the custom router, as long as the navigator component forwards it.

If the navigator component does NOT forward the prop to `useNavigationBuilder` (e.g., it consumes it for rendering only), you may need to modify the navigator to thread it through. If neither `.with()` nor navigator modification can deliver the option to the router, keep the navigator dynamic.

### Convert navigator JSX to static config

Convert the existing navigator first, then introduce screen config objects only where a screen needs options, listeners, params, IDs, linking, or `if`.

Before:

```tsx
const Stack = createNativeStackNavigator();

function MyStack() {
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen
        name="Profile"
        component={ProfileScreen}
        options={{ title: 'My Profile' }}
      />
    </Stack.Navigator>
  );
}
```

After:

```tsx
const MyStack = createNativeStackNavigator({
  screenOptions: { headerShown: false },
  screens: {
    Home: HomeScreen,
    Profile: {
      screen: ProfileScreen,
      options: { title: 'My Profile' },
    },
  },
});
```

Full screen config shape:

```tsx
const MyStack = createNativeStackNavigator({
  screens: {
    Example: {
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
        path: 'pattern/:id',
        parse: { id: Number },
        stringify: { id: (value) => String(value) },
        exact: true,
      },
      if: useConditionHook,
      layout: ({ children }) => children,
    },
  },
});
```

Shorthand (component only, no config): `ScreenName: ScreenComponent`

Nested static navigator: `ScreenName: AnotherStaticNavigator`

### Convert nested navigators

Nested dynamic navigators rendered as components become nested config objects.

Before:

```tsx
const Tab = createBottomTabNavigator();

function HomeTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Groups" component={GroupsScreen} />
      <Tab.Screen name="Chats" component={ChatsScreen} />
    </Tab.Navigator>
  );
}

function RootStack() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="Home" component={HomeTabs} />
    </Stack.Navigator>
  );
}
```

After:

```tsx
const HomeTabs = createBottomTabNavigator({
  screens: {
    Groups: GroupsScreen,
    Chats: ChatsScreen,
  },
});

const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeTabs,
  },
});
```

### Convert groups

Before:

```tsx
function RootStack() {
  return (
    <Stack.Navigator>
      <Stack.Group screenOptions={{ headerStyle: { backgroundColor: 'red' } }}>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Profile" component={ProfileScreen} />
      </Stack.Group>
      <Stack.Group screenOptions={{ presentation: 'modal' }}>
        <Stack.Screen name="Settings" component={SettingsScreen} />
      </Stack.Group>
    </Stack.Navigator>
  );
}
```

After:

```tsx
const RootStack = createNativeStackNavigator({
  groups: {
    Card: {
      screenOptions: { headerStyle: { backgroundColor: 'red' } },
      screens: {
        Home: HomeScreen,
        Profile: ProfileScreen,
      },
    },
    Modal: {
      screenOptions: { presentation: 'modal' },
      screens: {
        Settings: SettingsScreen,
      },
    },
  },
});
```

Top-level `screens` and `screenOptions` handle the default group.

Use `groups` when you need different shared options, conditional groups, grouped linking, or to logically group screens if the dynamic config already had such groups.

### Convert auth flows

To migrate conditional screens from dynamic config, use static `if` hooks. The `if` property takes a user-defined hook that returns a boolean such as `useIsSignedIn` or `useIsSignedOut`.

This prevents navigating to protected screens when signed out and unmounts auth screens after sign-in, so the back button cannot return to them.

If you previously used `navigationKey` to reset a screen when auth state changes, duplicate the screen in both auth groups. The group name is used for the key, so switching groups resets the screen. For example, declare `Help` in both the signed-in and signed-out groups instead of using `navigationKey`.

Loading UI should live outside the navigation tree, meaning outside `<NavigationContainer>` / `<Navigation>`, not in a `Loading` screen or group. Keep `screens` and `groups` for actual navigable routes only.

Use `.with()` for wrappers around a mounted navigator, not for boot or loading gates that should happen before rendering `<Navigation>`.

```tsx
const App = () => {
  const isLoading = useIsLoading();

  if (isLoading) {
    return <SplashScreen />;
  }

  return <Navigation />;
};
```

Before:

```tsx
function App() {
  const isSignedIn = useIsSignedIn();

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
      <Stack.Screen
        navigationKey={isSignedIn ? 'signed-in' : 'signed-out'}
        name="Help"
        component={HelpScreen}
      />
    </Stack.Navigator>
  );
}
```

After:

```tsx
const RootStack = createNativeStackNavigator({
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

### Use `.with()` for wrappers, providers, and dynamic navigator props

`.with()` renders a regular React component. It can use any React hooks (including `useEffect`, `useCallback`, custom hooks), return early (e.g., `return null` for loading guards in nested navigators), and render arbitrary JSX (multiple nested wrappers, event handlers, refs, animated views). The `{ Navigator }` argument is the configured navigator component — render it wherever you would render `<Stack.Navigator>`.

If the dynamic navigator is rendered in a component that uses hooks for navigator-level behavior, or has wrappers around the mounted navigator, use `.with()` to provide this wrapper. This applies to navigator-level props such as `initialRouteName`, `backBehavior`, `screenOptions`, and `screenListeners` that are derived dynamically.

#### Wrapping with a provider and dynamic props and options

Before:

```tsx
function MyStack() {
  const someValue = useSomeHook();

  return (
    <SomeProvider>
      <Stack.Navigator screenOptions={{ title: someValue }}>
        <Stack.Screen name="Home" component={HomeScreen} />
      </Stack.Navigator>
    </SomeProvider>
  );
}
```

After:

```tsx
const MyStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
  },
}).with(({ Navigator }) => {
  const someValue = useSomeHook();

  return (
    <SomeProvider>
      <Navigator screenOptions={{ title: someValue }} />
    </SomeProvider>
  );
});
```

#### Using props based on parent route

Before:

```tsx
function MyStack({ route }) {
  return (
    <Stack.Navigator screenOptions={{ title: route.params.title }}>
      <Stack.Screen name="Home" component={HomeScreen} />
    </Stack.Navigator>
  );
}
```

After:

```tsx
const MyStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
  },
}).with(({ Navigator }) => {
  const route = useRoute();

  return <Navigator screenOptions={{ title: route.params.title }} />;
});
```

#### Per-screen dynamic options via `screenOptions` callback

If each screen has different options, use a `screenOptions` callback and switch on `route.name`.

Before:

```tsx
function MyStack() {
  const getSomething = useSomeHook();

  return (
    <SomeProvider>
      <Stack.Navigator>
        <Stack.Screen
          name="Home"
          component={HomeScreen}
          options={{ title: getSomething('First') }}
        />
        <Stack.Screen
          name="Profile"
          component={ProfileScreen}
          options={{ title: getSomething('Second') }}
        />
      </Stack.Navigator>
    </SomeProvider>
  );
}
```

After:

```tsx
const MyStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
    Profile: ProfileScreen,
  },
}).with(({ Navigator }) => {
  const getSomething = useSomeHook();

  return (
    <SomeProvider>
      <Navigator
        screenOptions={({ route }) => {
          switch (route.name) {
            case 'Home':
              return {
                title: getSomething('First'),
              };
            case 'Profile':
              return {
                title: getSomething('Second'),
              };
            default:
              return {};
          }
        }}
      />
    </SomeProvider>
  );
});
```

#### Dynamic `initialParams` from hooks

If a screen's `initialParams` are computed from hooks or URL state at mount time, use `.with()` to compute them and pass via the `screenOptions` callback or a context provider.

Before:

```tsx
function MyStack() {
  const computedId = useComputedId();

  return (
    <Stack.Navigator>
      <Stack.Screen
        name="Detail"
        component={DetailScreen}
        initialParams={{ id: computedId }}
      />
    </Stack.Navigator>
  );
}
```

After:

```tsx
const MyStack = createNativeStackNavigator({
  screens: {
    Detail: DetailScreen,
  },
}).with(({ Navigator }) => {
  const computedId = useComputedId();

  return <Navigator screenOptions={{ initialParams: { id: computedId } }} />;
});
```

If different screens need different computed `initialParams`, use a context provider in `.with()` and read the context inside each screen component.

#### Merging a static per-screen options map with dynamic base options

If the codebase defines a static map of per-screen options and merges them with hook-derived base options at runtime, use a `screenOptions` callback inside `.with()`:

```tsx
const OPTIONS_PER_SCREEN: Record<string, object> = {
  Settings: { animationTypeForReplace: 'pop' },
  Profile: { gestureDirection: 'vertical' },
};

const MyStack = createNativeStackNavigator({
  screens: {
    Settings: SettingsScreen,
    Profile: ProfileScreen,
  },
}).with(({ Navigator }) => {
  const baseOptions = useBaseScreenOptions();

  return (
    <Navigator
      screenOptions={({ route }) => ({
        ...baseOptions,
        ...OPTIONS_PER_SCREEN[route.name],
      })}
    />
  );
});
```

#### Convert render callbacks for screens

Static config doesn't support render callbacks on screens.

For additional props passed to the screen component, move the data to React context and provide it via `.with()`.

Passing additional props via context:

Before:

```tsx
<Stack.Screen name="Chat">
  {(props) => <ChatScreen {...props} userToken={token} />}
</Stack.Screen>
```

After:

```tsx
const TokenContext = React.createContext('');

function ChatScreen() {
  const token = React.useContext(TokenContext);

  return <Chat token={token} />;
}

const MyStack = createNativeStackNavigator({
  screens: {
    Chat: ChatScreen,
  },
}).with(({ Navigator }) => {
  const token = useToken();

  return (
    <TokenContext.Provider value={token}>
      <Navigator />
    </TokenContext.Provider>
  );
});
```

For wrappers around the screen component, move the wrapper to the screen's `layout`.

Before:

```tsx
<Stack.Screen name="Profile">
  {(props) => (
    <SomeWrapper>
      <ProfileScreen {...props} />
    </SomeWrapper>
  )}
</Stack.Screen>
```

After:

```tsx
const MyStack = createNativeStackNavigator({
  screens: {
    Profile: {
      screen: ProfileScreen,
      layout: ({ children }) => <SomeWrapper>{children}</SomeWrapper>,
    },
  },
});
```

For refs passed to the screen component, use context and wrap the screen in a component that forwards the ref.

Before:

```tsx
<Stack.Screen name="Profile">
  {(props) => <ProfileScreen {...props} ref={profileRef} />}
</Stack.Screen>
```

After:

```tsx
const MyStack = createNativeStackNavigator({
  screens: {
    Profile: {
      screen: () => {
        const profileRef = React.useContext(ProfileRefContext);

        return <ProfileScreen ref={profileRef} />;
      },
    },
  },
}).with(({ Navigator }) => {
  const profileRef = React.useRef();

  return (
    <ProfileRefContext.Provider value={profileRef}>
      <Navigator />
    </ProfileRefContext.Provider>
  );
});
```

If multiple of these patterns are used on the same screen, use appropriate combinations of context and layout.

#### Migrating navigator factories

A factory function that generates multiple navigators from a shared template (e.g., a function that takes a screen map and returns a configured navigator component) is not a custom navigator in the `useNavigationBuilder` sense — it is a wrapper-generator.

To migrate a factory to static config:

1. Convert each factory invocation to a separate `createXNavigator({ screens: ... })` call.
2. Extract shared wrapper logic into a reusable `.with()` callback.

```tsx
// Shared wrapper for all factory-generated navigators
function withModalWrapper({
  Navigator,
}: {
  Navigator: React.ComponentType<any>;
}) {
  const screenOptions = useModalScreenOptions();

  return (
    <View style={styles.modalContainer}>
      <Navigator screenOptions={screenOptions} />
    </View>
  );
}

// Each factory call becomes a static navigator with the shared wrapper
const SettingsModal = createNativeStackNavigator({
  screens: {
    Profile: ProfileScreen,
    Preferences: PreferencesScreen,
  },
}).with(withModalWrapper);

const ReportsModal = createNativeStackNavigator({
  screens: {
    ReportList: ReportListScreen,
    ReportDetail: ReportDetailScreen,
  },
}).with(withModalWrapper);
```

If the factory iterates over a static object to build screens (e.g., `Object.keys(screenMap)`), the screens ARE known at build time — this is a static candidate despite the iteration pattern.

#### Migrating `getComponent` lazy loading

Static config uses a `screen` component and doesn't support `getComponent`. Use a custom utility to lazily render the screen:

```tsx
const lazyScreen = <T extends React.ComponentType<any>>(
  getComponent: () => T,
) => {
  return function LazyScreen(props: React.ComponentProps<T>) {
    const Component = getComponent();

    return <Component {...props} />;
  };
};
```

Place this utility in a shared file such as `utils/lazyScreen.ts` following the pattern of other shared utilities in the codebase.

This utility preserves synchronous loading semantics. Do not use `React.lazy(() => import(...))` for `getComponent` migration -- `React.lazy` requires async `import()` and introduces Suspense fallback flashes for screens that previously loaded instantly via synchronous `require()`. The `lazyScreen` utility above avoids this by calling the factory synchronously during render.

Then, replace `getComponent` with the lazy screen:

Before:

```tsx
<Stack.Screen
  name="Settings"
  getComponent={() => require('./SettingsScreen').default}
/>
```

After:

```tsx
const MyStack = createNativeStackNavigator({
  screens: {
    Settings: {
      screen: lazyScreen<typeof import('./SettingsScreen').default>(
        () => require('./SettingsScreen').default,
      ),
    },
  },
});
```

### Migrate screen-level linking

Use screen-level `linking` to replace the old root `linking.config.screens` structure.

Omit `linking` on a screen when the default kebab-case path is acceptable. If the path is identical to the auto path such as `Details` to `details`, remove the redundant `linking` entry.

Add `linking` for custom paths or when you need path params with `parse` or `stringify`.

Before:

```tsx
const linking = {
  prefixes: ['https://example.com'],
  config: {
    screens: {
      Home: '',
      Profile: {
        path: 'user/:id',
        parse: { id: Number },
      },
      Settings: 'settings',
    },
  },
};
```

After:

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    Home: {
      screen: HomeScreen,
      linking: '', // explicit root path; omit if this is the first leaf screen or the initialRouteName
    },
    Profile: {
      screen: ProfileScreen,
      linking: {
        path: 'user/:id',
        parse: { id: Number },
      },
    },
    Settings: SettingsScreen,
  },
});
```

Linking paths are auto-generated for leaf screens using kebab-case of the screen name. The first leaf screen, or the `initialRouteName` if set, gets the path `/` unless you set an explicit empty path on another screen.

To control auto-generated linking, pass `enabled` on the root `linking` prop: `enabled: 'auto'` generates paths for all leaf screens, and `enabled: false` disables linking entirely.

If a screen previously had a custom path such as `linking: 'contacts'` and you remove it, the auto path becomes kebab-case of the screen name such as `TabContacts` to `tab-contacts`. This breaks existing URLs and deep links. Keep explicit `linking` when you need to preserve existing paths.

If screens containing navigators have `linking` set to `''` or `'/'`, it is usually redundant and can be removed.

Keep TypeScript param typing on the screen component with `StaticScreenProps`. Screen-level `linking` config is for URL parsing and serialization only.

### Update types

#### Getting navigation and route access

Use `StaticScreenProps` to type the screen's `route` prop.

Use the default `useNavigation()` type provided through the global `ReactNavigation.RootParamList` augmentation for navigator-agnostic navigation calls.

Prefer the screen `route` prop over `useRoute` when available. Use `useNavigationState` separately when you need navigation state.

Before:

```tsx
function ProfileScreen({
  navigation,
  route,
}: NativeStackScreenProps<MyStackParamList, 'Profile'>) {
  const id = route.params.id;
  navigation.navigate('Home');
}
```

After:

```tsx
type ProfileScreenProps = StaticScreenProps<{
  id: string;
}>;

function ProfileScreen({ route }: ProfileScreenProps) {
  const navigation = useNavigation();
  const id = route.params.id;

  navigation.navigate('Home');
}
```

Use `StaticScreenProps` to type the `route` prop. If you need navigator-specific APIs such as `push`, `pop`, `openDrawer`, or `setOptions`, you can manually annotate `useNavigation`, but this is not type-safe and should be kept to a minimum.

If you need a navigator-specific navigation object:

```tsx
type RootStackParamList = StaticParamList<typeof RootStack>;

type ProfileNavigationProp = NativeStackNavigationProp<
  RootStackParamList,
  'Profile'
>;

const navigation = useNavigation<ProfileNavigationProp>();
```

#### Remove manual param lists

Remove all hand-written param-list declarations created only to support dynamic typing.

If a param list is absolutely necessary, derive it from the navigator type:

```tsx
type SomeStackType = typeof SomeStack;
type SomeStackParamList = StaticParamList<SomeStackType>;
```

If a static navigator nests a dynamic navigator, annotate the dynamic navigator screen with `StaticScreenProps<NavigatorScreenParams<...>>` so the nesting is reflected in the root param list.

For the root navigator, keep the single source of truth in the `RootParamList` augmentation shown below.

#### Custom screen prop types

If the codebase uses custom screen prop types from a custom navigator (e.g., `PlatformStackScreenProps<ParamList, ScreenName>` instead of `NativeStackScreenProps`), migrate them alongside the standard types:

1. Replace the custom screen prop type with `StaticScreenProps` for param typing.
2. If the custom type provides navigator-specific navigation methods, use `useNavigation()` with a manual type annotation as a temporary bridge.
3. Update all screen files that reference the old custom type.

If many screen files (10+) reference the custom type, consider a codemod or find-and-replace to update them in bulk rather than manually.

Avoid circular dependencies by:

- Using `StaticScreenProps` for screen params instead of shared hand-written param lists
- Using the default `useNavigation()` type where possible instead of navigator-specific aliases
- Deleting obsolete shared type files when they become empty

#### Root type augmentation

Place the global `RootParamList` augmentation next to the root static navigator. This is the single source of truth for the default types used by `useNavigation`, `Link`, refs, and related APIs.

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    // ...
  },
});

type RootStackParamList = StaticParamList<typeof RootStack>;

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

#### Typing params

Use `StaticScreenProps` to annotate route params, including screens that use `linking`.

A path such as `user/:userId` defines URL parsing and serialization. Keep the TypeScript param type on the screen component with `StaticScreenProps`.

If the params are not strings, use `parse` and `stringify` in the `linking` config:

```tsx
type ArticleScreenProps = StaticScreenProps<{
  date: Date;
}>;

function ArticleScreen({ route }: ArticleScreenProps) {
  return <Article date={route.params.date} />;
}

const RootStack = createNativeStackNavigator({
  screens: {
    Article: {
      screen: ArticleScreen,
      linking: {
        path: 'article/:date',
        parse: {
          date: (date: string) => new Date(date),
        },
        stringify: {
          date: (date: Date) => date.toISOString(),
        },
      },
    },
  },
});
```

The runtime parsing comes from `linking`. The compile-time param type comes from `StaticScreenProps`.

Avoid `any`, non-null assertions, and `as` assertions.

Before:

```tsx
type MyStackParamList = {
  Article: { author: string };
  Albums: undefined;
};

const Stack = createNativeStackNavigator<MyStackParamList>();

function ArticleScreen({
  navigation,
  route,
}: NativeStackScreenProps<MyStackParamList, 'Article'>) {
  return <Button onPress={() => navigation.navigate('Albums')} />;
}

function AlbumsScreen() {
  return <Albums />;
}

export function Example() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen
          name="Article"
          component={ArticleScreen}
          options={({ route }) => ({ title: route.params.author })}
          initialParams={{ author: 'Gandalf' }}
        />
        <Stack.Screen name="Albums" component={AlbumsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

After:

```tsx
import {
  createStaticNavigation,
  type StaticParamList,
  type StaticScreenProps,
  useNavigation,
} from '@react-navigation/native';

type ArticleScreenProps = StaticScreenProps<{
  author: string;
}>;

function ArticleScreen({ route }: ArticleScreenProps) {
  const navigation = useNavigation();

  return (
    <Button
      title={route.params.author}
      onPress={() => navigation.navigate('Albums')}
    />
  );
}

function AlbumsScreen() {
  return <Albums />;
}

const RootStack = createNativeStackNavigator({
  screens: {
    Article: {
      screen: ArticleScreen,
      options: ({ route }) => ({ title: route.params.author }),
      initialParams: { author: 'Gandalf' },
      linking: 'article/:author',
    },
    Albums: AlbumsScreen,
  },
});

type RootStackParamList = StaticParamList<typeof RootStack>;

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

const Navigation = createStaticNavigation(RootStack);

export function Example() {
  return <Navigation />;
}
```

### Replace the root container

Replace `NavigationContainer` with `createStaticNavigation(RootStack)` once the root static navigator, screen-level linking, and root typing are in place. Then pass container-level props to the generated `Navigation` component.

Before:

```tsx
const linking = {
  prefixes: ['https://example.com', 'example://'],
  config: {
    screens: {
      Home: '',
      Profile: 'profile/:id',
    },
  },
};

function App() {
  return (
    <NavigationContainer linking={linking} theme={MyTheme}>
      <RootStack />
    </NavigationContainer>
  );
}
```

After:

```tsx
import { createStaticNavigation } from '@react-navigation/native';

const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
    Profile: {
      screen: ProfileScreen,
      linking: 'profile/:id',
    },
  },
});

const Navigation = createStaticNavigation(RootStack);

function App() {
  return (
    <Navigation
      linking={{
        prefixes: ['https://example.com', 'example://'],
        enabled: 'auto',
      }}
      theme={MyTheme}
    />
  );
}
```

Keep screen path details on individual screens.

The `Navigation` component returned by `createStaticNavigation` cannot take a full `linking.config` object. Put per-screen paths in screen-level `linking`, and use the root `linking` prop only for container-level settings and root-level linking options.

Only configuration from the `screens` property needs to be moved to screen-level `linking`. Container-level linking options such as `prefixes`, custom `getStateFromPath`, `getPathFromState` and any other properties can still be passed to the root-level `linking` prop.

All the props that were previously passed to `NavigationContainer` such as `theme`, `fallback`, and `onReady` can be passed to the `Navigation` component without changes except for `children`, and require changes to `linking` as described above.

## Mixing Static and Dynamic APIs

Use mixed static/dynamic trees only as a fallback when full static migration is not possible.

Prefer a static root navigator. Once any part of the tree remains dynamic, automatic linking and automatic TypeScript types stop at that boundary. Handle linking and typing for the mixed boundary manually.

### Static root navigator, dynamic nested navigator

Use this fallback when a parent navigator can be migrated but a nested navigator cannot.

- Keep the parent navigator static
- Keep the dynamic navigator under a screen in the static parent navigator
- Define linking for the dynamic child screens manually in the parent screen's `linking.screens`
- Type the parent screen params with `StaticScreenProps<NavigatorScreenParams<...>>`

```tsx
type FeedParamList = {
  Latest: undefined;
  Popular: undefined;
};

type FeedScreenProps = StaticScreenProps<
  NavigatorScreenParams<FeedParamList>
>;

function FeedScreen(_: FeedScreenProps) {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Latest" component={LatestScreen} />
      <Tab.Screen name="Popular" component={PopularScreen} />
    </Tab.Navigator>
  );
}

const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
    Feed: createNativeStackScreen({
      screen: FeedScreen,
      linking: {
        path: 'feed',
        screens: {
          Latest: 'latest',
          Popular: 'popular',
        },
      },
    }),
  },
});
```

### Dynamic root navigator, static nested navigator

Use this fallback only when a parent navigator cannot be migrated and must remain dynamic.

- Migrate the nested navigator to the static API
- Use `.getComponent()` on the static navigator to get a screen component for the dynamic parent
- Derive params with `StaticParamList<typeof StaticNavigator>` and use `NavigatorScreenParams<...>` in the dynamic parent's param list
- Generate linking config with `createPathConfigForStaticNavigation(StaticNavigator)` and place it in:
  - The `linking.config` of `NavigationContainer` if the parent dynamic navigator is the root navigator
  - The `linking.screens` of the screen in static grandparent navigator of the dynamic parent if the parent dynamic navigator is nested

```tsx
const FeedTabs = createBottomTabNavigator({
  screens: {
    Latest: LatestScreen,
    Popular: PopularScreen,
  },
});

const FeedScreen = FeedTabs.getComponent();

type FeedTabsParamList = StaticParamList<typeof FeedTabs>;

type RootStackParamList = {
  Home: undefined;
  Feed: NavigatorScreenParams<FeedTabsParamList>;
};

const feedScreens = createPathConfigForStaticNavigation(FeedTabs);

const linking = {
  prefixes: ['https://example.com', 'example://'],
  config: {
    screens: {
      Home: '',
      Feed: {
        path: 'feed',
        screens: feedScreens,
      },
    },
  },
};
```

### Gotchas

#### Platform-specific option keys

If the codebase uses custom navigator types that extend screen options with platform-specific keys (e.g., a `web:` or `native:` key inside `screenOptions`), these are not standard React Navigation options. They will work in static config as long as the custom navigator's TypeScript types accept them. When passing platform-specific options via `.with()`'s `screenOptions` prop, ensure the custom navigator component handles the platform-specific keys the same way it does in dynamic config — the `.with()` wrapper passes `screenOptions` through to the navigator unchanged.

#### Module-load timing

Static navigator config is created at module load time, not during component render.

Be careful with:

- `translate(...)`
- context-derived values
- feature-flag values read too early

If the value should resolve later, wrap it in a callback:

```tsx
options: () => ({
  tabBarLabel: translate('tabs:home'),
});
```

## Limitations

- Cannot use React hooks such as `useTheme()` directly in `options` or `listeners` callbacks. Use callback arguments such as `theme` instead. `React.use()` (React 19+) can read context in `options` callbacks but may trigger ESLint warnings.
- The screen list is static. Screens cannot be added or removed at runtime. Use `if` hooks for conditional rendering.
- No render callbacks on screens. Move extra props to React context and wrappers to `layout`.

## Review

1. All `@react-navigation/*` packages were updated to the latest published 7.x versions before migration.
2. No `NativeStackScreenProps`, `BottomTabScreenProps`, or custom screen-prop aliases remain. Use `useNavigation()` for access, the screen `route` prop when available, and `StaticScreenProps` for params.
3. `RootParamList` augmentation in `ReactNavigation` lives next to the root static navigator.
4. `createStaticNavigation` replaces `NavigationContainer`.
5. Root `linking` contains container-level settings such as `prefixes` and `enabled`. Screen paths live in screen-level `linking`.
6. Linking config is present only where custom paths or params are required. Defaults are kebab-case.
7. Every screen with params uses `StaticScreenProps`. Screen-level `linking` is used for URL parsing and serialization.
8. No render callbacks remain on screens. Extra props use React context and wrappers use `layout`.
9. Any previous `getComponent` screens now use custom utility
10. No hand-written param lists remain unless derived via `StaticParamList`.
11. No hooks are called directly in `screenOptions`, `options`, or `listeners` callbacks.
12. Loading or boot UI lives outside `<Navigation>`.
13. No circular type references or obsolete shared type files remain from the old dynamic setup.
