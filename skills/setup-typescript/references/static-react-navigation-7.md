# React Navigation 7.x Static TypeScript

Use this file only when `@react-navigation/native` is on `7.x` and the navigator uses static config.

## Goal

Make static navigators type-safe end to end: screen props, nested navigators, container refs, and `Link` targets should all stay aligned with the route map.

## Check First

1. Confirm the project is on React Navigation `7.x`.
2. Confirm the navigator is static, for example `createXNavigator({ screens: ... })` or `createStaticNavigation`.
3. Check `tsconfig.json` has `strict: true` or `strictNullChecks: true` and `moduleResolution: "bundler"`.
4. Identify any dynamic nested navigator inside the static tree.

## Workflow

### 1. Type screen props first

Use `StaticScreenProps` for each screen component. Keep the param type close to the screen so the component stays reusable.

```tsx
import type { StaticScreenProps } from '@react-navigation/native';

type ProfileProps = StaticScreenProps<{
  userId: string;
}>;

function ProfileScreen({ route }: ProfileProps) {
  return <Text>{route.params.userId}</Text>;
}
```

Use `undefined` for routes with no params, or a union with `undefined` when params are optional.

### 2. Derive the root param list

Generate the root param list from the root navigator and expose it globally so default navigation APIs pick it up.

```tsx
import type { StaticParamList } from '@react-navigation/native';

const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
    Profile: ProfileScreen,
  },
});

type RootStackParamList = StaticParamList<typeof RootStack>;

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

Use a `type` alias for param lists. Do not use an interface for the route map itself.

### 3. Type refs and navigation helpers

Once `ReactNavigation.RootParamList` is extended, `useNavigation`, `Link`, and container refs use the static root types by default.

Use explicit ref generics when you create a navigation container ref:

```tsx
import type { NavigationContainerRef } from '@react-navigation/native';

const navigationRef =
  React.createRef<NavigationContainerRef<RootStackParamList>>();
```

If you use a helper such as `createNavigationContainerRef` or `useNavigationContainerRef`, pass the same root param list type.

Only annotate `useNavigation` manually when you need navigator-specific APIs such as `push`, `pop`, `openDrawer`, or `closeDrawer`. Prefer the default root types when they are enough.

### 4. Handle mixed static and dynamic trees

If a nested navigator is still dynamic, type it explicitly. `StaticParamList<typeof RootStack>` cannot infer screens declared only inside JSX.

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

function SettingsStack(_: SettingsProps) {
  return (
    <Tab.Navigator>
      <Tab.Screen name="General" component={GeneralScreen} />
      <Tab.Screen name="Account" component={AccountScreen} />
    </Tab.Navigator>
  );
}
```

If the nested navigator is also static, keep it static and let `StaticParamList` infer it instead of adding manual nesting types.

### 5. Keep linking and params in sync

The root param list also drives `Link` and `linking` nesting. Keep screen params, link targets, and path patterns aligned.

- If a route path carries `userId`, make the screen params include `userId`.
- If a screen has no params, keep the route type `undefined`.
- If you use `path`, `parse`, or `stringify`, keep the value types aligned with the route params you want to expose.

### 6. Verify the typing shape

Check that all of the following compile:

- screen `route.params`
- `navigate`, `push`, `replace`, and `reset` calls
- `NavigationContainerRef` usage
- `Link` targets for root and nested routes
- any explicit `useNavigation<...>()` annotations

## Common Mistakes

- Forgetting to extend `ReactNavigation.RootParamList`.
- Annotating every hook manually instead of letting the global root type do the work.
- Using a static root navigator but leaving a nested navigator completely untyped because it is still JSX-based.
- Letting a linking path and route params drift apart.
