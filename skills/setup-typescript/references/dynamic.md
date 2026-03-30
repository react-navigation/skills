# React Navigation Dynamic TypeScript

Use this file when the navigator tree is JSX-based dynamic config or when the app is not using static config.

## Goal

Type dynamic navigators explicitly so each screen, nested navigator, ref, and hook gets the correct route and param shape.

## Check First

1. Confirm the tree is dynamic, for example `<Stack.Navigator>` / `<Stack.Screen>` or `createXNavigator()` with JSX children.
2. Check `tsconfig.json` has `strict: true` or `strictNullChecks: true` and `moduleResolution: "bundler"`.
3. Identify nested navigators and routes that take params.

## Workflow

### 1. Define a param list for every navigator

Use `type` aliases, one per navigator. Use `undefined` for routes without params.

```tsx
type HomeTabParamList = {
  Feed: undefined;
  Profile: { userId: string };
};

type RootStackParamList = {
  Home: NavigatorScreenParams<HomeTabParamList>;
  Details: { itemId: string };
};
```

Use `NavigatorScreenParams<ChildParamList>` when a route points to another navigator.

### 2. Pass the param list to the navigator generic

Give each navigator its matching param list so route names and params are checked.

```tsx
const HomeTabs = createBottomTabNavigator<HomeTabParamList>();
const RootStack = createNativeStackNavigator<RootStackParamList>();
```

If the tree is deeply nested, keep each param list local to the navigator it describes.

### 3. Type each screen component with the navigator-specific prop type

Use the prop type that matches the navigator you are inside.

```tsx
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

type DetailsProps = NativeStackScreenProps<RootStackParamList, 'Details'>;

function DetailsScreen({ navigation, route }: DetailsProps) {
  return <Text>{route.params.itemId}</Text>;
}
```

For tab and drawer screens, use the matching prop type from that navigator package. For a screen nested inside another navigator, combine props with `CompositeScreenProps`.

```tsx
import type { CompositeScreenProps } from '@react-navigation/native';
import type { BottomTabScreenProps } from '@react-navigation/bottom-tabs';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

type FeedProps = CompositeScreenProps<
  BottomTabScreenProps<HomeTabParamList, 'Feed'>,
  NativeStackScreenProps<RootStackParamList>
>;
```

### 4. Expose the root param list globally when you want defaults

Extend `ReactNavigation.RootParamList` so `useNavigation`, `Link`, and refs pick up the root types by default.

```tsx
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

This is the easiest way to keep default hooks and navigation refs in sync with the route map.

### 5. Type refs and navigation helpers

Use the root param list on refs so `navigate`, `reset`, and similar actions stay checked.

```tsx
import type { NavigationContainerRef } from '@react-navigation/native';

const navigationRef =
  React.createRef<NavigationContainerRef<RootStackParamList>>();
```

You can also use `createNavigationContainerRef<RootStackParamList>()` or `useNavigationContainerRef<RootStackParamList>()` when that fits the codebase better.

Only annotate `useNavigation` manually when you need navigator-specific APIs such as `push`, `pop`, `openDrawer`, or `closeDrawer`. Manual annotations are not type-safe, so keep them narrow.

### 6. Keep linking and params in sync

The root param list also helps `Link` and the `linking` prop stay aligned with nested route names. Keep the params, route names, and linking path structure in sync.

- Use `NavigatorScreenParams` for nested routes.
- Keep `Link` targets on real route names.
- Make sure path params and screen params describe the same shape.

### 7. Verify the typing shape

Check that these compile cleanly:

- each screen's `route.params`
- each `navigate` or `push` call
- nested screen props using `CompositeScreenProps`
- navigation container refs
- `Link` destinations

## Common Mistakes

- Using one flat param list for a nested navigator tree.
- Forgetting `NavigatorScreenParams` for child navigators.
- Annotating `useNavigation` everywhere instead of typing the screen props first.
- Extending `RootParamList` but leaving the navigator's own param lists incomplete.
