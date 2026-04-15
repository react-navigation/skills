---
name: migrate-to-static-config
description: "Use when converting React Navigation navigators from JSX-based dynamic config to the static object-based config API. Covers createXNavigator({screens}), .with() wrappers, custom navigators using useNavigationBuilder, lazy loading migration, auth flows, linking, and type updates."
compatibility: "React Navigation 7.x+"
---

# Migrating to Static Config

## Goal

Convert React Navigation navigators from JSX-based dynamic setup to static configuration while preserving behavior, typing, and deep links.

## When to use

You are migrating screens from Dynamic API to the Static API in React Navigation.

Triggering symptoms:

- Navigator files use `<Stack.Navigator>`, `<Tab.Navigator>`, or similar JSX patterns with `<Stack.Screen>` children
- The codebase has a `NavigationContainer` wrapping a component-based root navigator
- Hand-written `ParamList` types are maintained alongside navigator definitions
- A centralized `linking.config.screens` object duplicates the navigator tree structure
- `getComponent` or render callbacks are used on `Screen` elements

## Safety principle

**Do not break working code.** If you are uncertain whether a change preserves behavior, do NOT apply it. Instead:

1. Leave the code unchanged.
2. Add a comment or report to the user describing the suggested change, the uncertainty, and the risk.
3. Let the user decide whether to proceed.

This applies to: type changes that may break call sites, wrapper removal that may lose behavior, linking changes that may break deep links, and any transformation where the before/after equivalence is not obvious. When in doubt, leave it and explain.

## Adaptation policy

Treat the patterns in this skill as canonical starting points, not an exhaustive list. The examples are meant to illustrate the core patterns.

When applying this skill to a codebase:

- Prefer the simplest migration pattern that preserves behavior.
- First try to map the local code to an equivalent of the patterns in this skill.
- Do not require an exact matching example in the skill before proceeding.
- If the local code differs in structure, infer the closest equivalent pattern and adapt it.
- Avoid inventing bespoke abstractions unless the simpler patterns clearly cannot preserve existing behavior.
- Keep changes minimal and local to the migration.
- If multiple approaches are possible, choose the one closest to the existing code style and React Navigation's intended static API.

## Decision rule

Use this order of preference:

1. Direct static config conversion.
2. Static config plus `.with()` for navigator-level wrappers or dynamic navigator props.
3. Static config plus context for extra screen data that was previously passed through render callbacks.
4. Keep the navigator dynamic only if static config cannot express the screen structure without changing behavior materially.

## Scope rule

Do not treat the absence of an explicit example in this skill as a blocker. Use the guidance here to derive the appropriate migration for the local code.

## When to ask for clarification

Inspect the local code first.

If, after reading the relevant navigator and its immediate callers, you cannot explain how the final screen structure, linking behavior, and preserved behavior map to the static API with high confidence, pause and ask the user before editing code.

Ask for clarification when:

- Part of the behavior is hidden behind local abstractions.
- Migrating would require assumptions about which behavior is intentional.
- It is unclear whether related helpers should be updated as part of the same change.

## Environment conflicts

Project-level instructions (CLAUDE.md, .cursorrules, etc.) may restrict operations this skill requires. Handle conflicts as follows:

- **Package updates blocked** -- If project instructions prohibit modifying package.json, skip the prerequisite version check. Note which packages may need updating and inform the user, but proceed with migration guidance using current versions.
- **Code modification restricted** -- If instructions restrict editing navigation files, produce the migration as a diff or structured plan instead of applying changes directly. Explain what would change and why.
- **Tool execution restricted** -- If instructions prohibit running npm/yarn commands, skip automated checks. Document which checks the user should run manually.
- **Conflicting conventions** -- If project conventions conflict with this skill's patterns (e.g., "always use dynamic config"), flag the conflict to the user and ask how to proceed. Do not silently follow either instruction.

When in doubt, inform the user of the conflict and let them decide. Do not silently skip steps or silently override project instructions.

## Common Mistakes

These are patterns that commonly cause migration failures. Check for them proactively.

### 1. Bottom-up migration breaks parent types

`.with()` changes the return type from `React.ComponentType` to `TypedNavigatorStaticDecorated`. If the parent navigator is dynamic and uses `component={MyNavigator}`, it will produce a type error.

**Check:** Before migrating any navigator, run:
```bash
grep -rn 'component={YourNavigatorName}\|component: YourNavigatorName' src/
```
If any match is in a dynamic parent, migrate the parent first or use `.getComponent()` on the static child.

### 2. Using `React.lazy` for `getComponent` migration

`React.lazy` requires async `import()` and adds Suspense fallback flashes for screens that previously loaded instantly via synchronous `require()`. Use the synchronous `lazyScreen` utility from the reference file instead.

**Check:** After migration, run:
```bash
grep -rn 'React\.lazy' src/ --include='*.tsx' --include='*.ts'
```
Any match in a navigator file is likely wrong.

### 3. Hooks called directly in static config callbacks

Static navigator config is created at module load time, not during render. Hooks in `options`, `listeners`, or `screenOptions` callbacks will crash.

**Check:** Review every `options:` and `screenOptions:` in the static config. If any calls a hook (`use*`), move it to `.with()` or a `screenOptions` callback passed via `.with()`.

### 4. Missing `linking` on screens with custom paths

Auto-generated linking uses kebab-case of the screen name. Removing an explicit `linking` entry that had a custom path (e.g., `linking: 'contacts'`) silently changes the URL to the auto-generated path (e.g., `tab-contacts`), breaking existing deep links.

**Check:** Compare old linking config entries against new screen-level `linking`. Every screen that had a custom path in the old `linking.config.screens` must have an explicit `linking` in static config.

### 5. Outdated `@react-navigation/*` packages

`.with()` was added in later 7.x versions. If packages are outdated, `.with()` does not exist and the migration fails.

**Check:** Before starting, run:
```bash
npm ls @react-navigation/core @react-navigation/native @react-navigation/native-stack 2>/dev/null | grep -E '@react-navigation'
```
Then compare against latest published versions:
```bash
npm view @react-navigation/core@latest version
npm view @react-navigation/native@latest version
```

### 6. Test files rendering navigator as JSX

Test files that render `<MyNavigator />` break when the export changes from a component to a static config object.

**Check:** After migration, run:
```bash
grep -rn '<YourNavigatorName' tests/ __tests__/ src/**/*.test.* src/**/*.spec.*
```
Update test files to use `.getComponent()` or render via `createStaticNavigation()`.

## Definition of Done

Migration is complete when ALL of the following pass:

1. **TypeScript compiles cleanly:**
   ```bash
   npx tsc --noEmit
   ```
   Zero errors related to navigation types.

2. **No dynamic navigator patterns remain in migrated files:**
   ```bash
   grep -rn '<.*\.Navigator\|<.*\.Screen\|<.*\.Group' <migrated-files>
   ```
   Should return zero matches for files that were migrated to static config.

3. **No hand-written ParamList types remain (unless derived):**
   ```bash
   grep -rn 'ParamList\s*=' src/
   ```
   Every match should use `StaticParamList<typeof ...>`, not hand-written types.

4. **No old screen prop types remain in migrated screens:**
   ```bash
   grep -rn 'NativeStackScreenProps\|BottomTabScreenProps\|DrawerScreenProps' src/
   ```
   Migrated screens should use `StaticScreenProps` instead.

5. **Root type augmentation exists:**
   ```bash
   grep -rn 'RootParamList extends' src/
   ```
   Exactly one `ReactNavigation.RootParamList` augmentation next to the root static navigator.

6. **No `NavigationContainer` remains (if root was migrated):**
   ```bash
   grep -rn 'NavigationContainer' src/
   ```
   Should be replaced with `createStaticNavigation()`.

7. **Linking preserved:** Compare old `linking.config.screens` paths against new screen-level `linking` entries. Every custom path must be explicitly present.

8. **Wrapper behavior preserved:** For each migrated navigator that had wrapper JSX (providers, View wrappers, accessibility attributes, event handlers), verify the `.with()` callback reproduces the same DOM/component structure. Compare the old render output with the new one — every wrapper, style, and accessibility prop (`accessibilityViewIsModal`, `aria-modal`, `role`, etc.) must be present.

9. **App runs and navigates correctly:** Manual smoke test — navigate to every migrated screen, test deep links, test back button behavior.

## References

Check `@react-navigation/native` in `package.json` first.

- If `7.x`, read [`references/react-navigation-7.md`](./references/react-navigation-7.md)
- If `8.x`, read [`references/react-navigation-8.md`](./references/react-navigation-8.md)

Load exactly one reference file unless explicitly comparing versions.
