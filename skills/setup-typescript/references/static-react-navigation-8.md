# React Navigation 8.x Static TypeScript

Use this file only when `@react-navigation/native` is on `8.x` and the navigator uses static config.

## Goal

Type static navigators using the 8.x root navigator model so screens, hooks, `Link`, refs, and linking stay aligned without manual duplication.

## Check First

1. Confirm the project is on React Navigation `8.x`.
2. Confirm the navigator is static, for example `createXNavigator({ screens: ... })` or `createStaticNavigation`.
3. Check `tsconfig.json` has `strict: true` or `strictNullChecks: true` and `moduleResolution: "bundler"`.
4. Identify any nested navigator that is still dynamic.

## Workflow

### 1. Type screen props first

Use `StaticScreenProps` for each screen component.

```tsx
import type { StaticScreenProps } from '@react-navigation/native';

type ProfileProps = StaticScreenProps<{
  userId: string;
}>;

function ProfileScreen({ route }: ProfileProps) {
  return <Text>{route.params.userId}</Text>;
}
```

Keep the route param type close to the screen and use `undefined` for screens without params.

### 2. Declare the root navigator type

The 8.x static API uses module augmentation on `@react-navigation/core`.

```tsx
import type { StaticParamList } from '@react-navigation/native';

const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
    Profile: ProfileScreen,
  },
});

type RootStackType = typeof RootStack;
type RootStackParamList = StaticParamList<typeof RootStack>;

declare module '@react-navigation/core' {
  interface RootNavigator extends RootStackType {}
}
```

Use `StaticParamList<typeof RootStack>` as a local helper when you need a reusable route map for refs, tests, or shared helpers. The root navigator augmentation is what enables default inference.

### 3. Use typed hooks by screen name

Once the root navigator is augmented, hooks infer types from the screen name.

```tsx
function ProfileScreen() {
  const route = useRoute('Profile');
  const navigation = useNavigation('Profile');
  const focusedRouteName = useNavigationState(
    'Profile',
    (state) => state.routes[state.index].name
  );

  return <Text>{route.params.userId}</Text>;
}
```

Use the unnamed form only in reusable components that truly need root-wide types. Keep manual `useNavigation<...>()` annotations to a minimum and use them only when you need navigator-specific methods such as `push` or `pop`.

### 4. Use screen helpers when a config object needs stronger inference

When a static screen entry needs options, linking, listeners, or other screen config, prefer the navigator-specific screen helper such as `createNativeStackScreen`.

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    Profile: createNativeStackScreen({
      screen: ProfileScreen,
      linking: {
        path: 'profile/:userId',
      },
    }),
  },
});
```

If you use a `path` with params, keep the param type aligned with the path pattern. Use `parse`, `stringify`, and `exact` when you need custom conversion or matching.

### 5. Handle mixed static and dynamic trees

If a nested navigator is still dynamic, type it explicitly with `NavigatorScreenParams` and a screen prop type. `StaticParamList<typeof RootStack>` cannot infer screens declared only in JSX.

```tsx
import type {
  NavigatorScreenParams,
  StaticScreenProps,
} from '@react-navigation/native';

type SettingsStackParamList = {
  General: undefined;
  Account: { userId: string };
};

type SettingsProps = StaticScreenProps<
  NavigatorScreenParams<SettingsStackParamList>
>;
```

If the nested navigator is static too, keep it static and let the root type inference handle it.

### 6. Keep linking and params in sync

The root navigator type also informs `Link`, `ref`, and `linking`. Keep the route map, path patterns, and navigation helpers consistent.

- Use the same param shape in screen props and linking paths.
- Keep `Link` destinations on known route names.
- Make sure `NavigationContainerRef<RootStackParamList>` uses the same root param list if you create an explicit ref.

### 7. Verify the typing shape

Check that the following compile cleanly:

- `useRoute('ScreenName')`
- `useNavigation('ScreenName')`
- `useNavigationState('ScreenName', ...)`
- `Link` targets
- container refs
- explicit navigator-specific `useNavigation<...>()` annotations

## Common Mistakes

- Forgetting the `@react-navigation/core` module augmentation.
- Keeping a nested dynamic navigator untyped inside a static root tree.
- Mixing manual `useNavigation<...>()` annotations everywhere instead of using screen-name inference.
- Letting `linking.path` params drift away from the screen param type.
