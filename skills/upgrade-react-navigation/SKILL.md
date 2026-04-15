---
name: upgrade-react-navigation
description: Use when upgrading React Navigation across major versions (6.x to 7.x or 7.x to 8.x). Covers breaking changes in TypeScript setup, navigator options, bottom tabs, linking config, custom navigators with useNavigationBuilder, deprecated API removal, and peer dependency updates.
compatibility: Requires React Navigation 6.x or 7.x as starting version.
---

# Upgrade React Navigation

## Goal

Upgrade React Navigation to the next major version and handle all required breaking changes while preserving existing navigation behavior.

## When to use

You are upgrading React Navigation from one major version to the next (6.x -> 7.x or 7.x -> 8.x).

## Safety principle

**Do not break working code.** If you are uncertain whether a change preserves behavior, do NOT apply it. Instead:

1. Leave the code unchanged.
2. Add a comment or report to the user describing the suggested change, the uncertainty, and the risk.
3. Let the user decide whether to proceed.

This applies to: removing APIs whose replacements are unclear, updating custom navigator internals where the new type signature is ambiguous, changing `inactiveBehavior` on screens with custom persistence logic, and any transformation where the before/after equivalence is not obvious. When in doubt, leave it and explain.

## Adaptation policy

Treat the migration steps in this skill as canonical starting points, not an exhaustive list.

When applying this skill to a codebase:

- Follow the migration order specified in the reference file.
- If the codebase uses custom navigators, custom routers, or direct rendering of internal components (e.g., `StackView`), the reference may not cover every affected API. Use llms.txt to fetch the relevant official documentation pages and verify what changed.
- Do not require an exact matching example in the skill before proceeding. Infer the closest equivalent pattern and adapt it.
- Keep changes minimal -- upgrade only what the new version requires. Do not refactor unrelated code.

## Scope rule

Do not treat the absence of an explicit example as a blocker. If the reference covers the general pattern (e.g., "custom navigators need updated overloads"), use the guidance plus official docs to derive the specific changes needed for the local code.

## When to ask for clarification

Inspect the local code first.

If, after reading the relevant navigator and the upgrade reference, you cannot determine with high confidence whether an API changed or how to update it, pause and ask the user before editing code.

Ask for clarification when:

- A custom navigator uses internal APIs not covered by the reference or official docs.
- The upgrade would require assumptions about which behavior changes are acceptable.
- It is unclear whether related helpers, tests, or configuration should be updated as part of the same change.

## Environment conflicts

Project-level instructions (CLAUDE.md, .cursorrules, etc.) may restrict operations this skill requires. Handle conflicts as follows:

- **Package updates blocked** -- If project instructions prohibit modifying package.json, document the required version changes and inform the user. Do not proceed with code changes until dependencies are updated, as the new APIs may not be available.
- **Code modification restricted** -- If instructions restrict editing navigation files, produce the upgrade as a diff or structured plan instead of applying changes directly.
- **Tool execution restricted** -- If instructions prohibit running npm/yarn commands, skip automated checks. Document which checks the user should run manually.
- **Conflicting conventions** -- If project conventions conflict with the upgrade requirements, flag the conflict to the user and ask how to proceed.

When in doubt, inform the user of the conflict and let them decide. Do not silently skip steps or silently override project instructions.

## Common Mistakes

These are patterns that commonly cause upgrade failures. Check proactively.

### 1. Not upgrading all `@react-navigation/*` packages together

Mixing major versions (e.g., `@react-navigation/core` 8.x with `@react-navigation/native-stack` 7.x) causes type errors and runtime crashes.

**Check:** After upgrading, run:
```bash
npm ls 2>/dev/null | grep '@react-navigation' | grep -v 'deduped'
```
All packages must be on the same major version.

### 2. Stale `NavigatorID` generic in custom navigator types

Custom navigators using `useNavigationBuilder` typically have a `NavigatorID extends string | undefined` generic. This generic is removed in 8.x but easy to miss.

**Check:** Run:
```bash
grep -rn 'NavigatorID' src/ --include='*.ts' --include='*.tsx'
```
Every match needs updating — remove the generic parameter and its usage in `NavigationProp`, `DefaultNavigatorOptions`, and factory overloads.

### 3. Using removed `freezeOnBlur`/`detachInactiveScreens` instead of `inactiveBehavior`

These options are removed silently (no runtime error), so the app appears to work but inactive screens behave differently.

**Check:** Run:
```bash
grep -rn 'freezeOnBlur\|detachInactiveScreens\|detachPreviousScreen' src/ --include='*.ts' --include='*.tsx'
```
Replace every match with the appropriate `inactiveBehavior` value.

### 4. Missing `moduleResolution: 'bundler'` in tsconfig

8.x requires `moduleResolution: 'bundler'`. Without it, TypeScript cannot resolve the new package exports.

**Check:** Run:
```bash
grep -n 'moduleResolution' tsconfig.json
```
Must show `"bundler"`.

### 5. Replacing `setParams` with `pushParams` globally

Only call sites inside tab/drawer navigators with `backBehavior="fullHistory"` that NEED history push should use `pushParams`. All other `setParams` calls remain unchanged.

**Check:** Before replacing, identify which navigators use `backBehavior="fullHistory"`:
```bash
grep -rn 'backBehavior.*fullHistory\|fullHistory.*backBehavior' src/
```
Only `setParams` calls on screens INSIDE those navigators are candidates.

### 6. Bottom tabs missing `headerShown: true`

8.x changes the default to `headerShown: false`. Existing tab navigators lose their headers silently.

**Check:** Run:
```bash
grep -rn 'createBottomTabNavigator' src/ --include='*.ts' --include='*.tsx'
```
For each match, verify `headerShown: true` is in `screenOptions` if headers were previously visible.

## Definition of Done

Upgrade is complete when ALL of the following pass:

1. **TypeScript compiles cleanly:**
   ```bash
   npx tsc --noEmit
   ```
   Zero errors related to navigation types.

2. **No removed APIs remain:**
   ```bash
   grep -rn 'navigateDeprecated\|navigationInChildEnabled\|freezeOnBlur\|detachInactiveScreens\|detachPreviousScreen\|headerBackImage\b\|headerBackImageSource\|\.getId\b\|UNSTABLE_router\|UNSTABLE_routeNamesChangeBehavior' src/ --include='*.ts' --include='*.tsx'
   ```
   Zero matches.

3. **No stale `NavigatorID` generics remain:**
   ```bash
   grep -rn 'NavigatorID' src/ --include='*.ts' --include='*.tsx'
   ```
   Zero matches (unless the codebase defines its own unrelated `NavigatorID`).

4. **No navigator `id` props remain:**
   ```bash
   grep -rn '<.*Navigator.*\bid=' src/ --include='*.tsx'
   ```
   Zero matches. All `navigation.getParent(id)` calls rewritten to use screen names.

5. **Peer dependencies correct:**
   ```bash
   npm ls react-native-screens react-native-safe-area-context react-native-reanimated react-native-pager-view @callstack/liquid-glass 2>/dev/null
   ```
   All installed at required minimum versions.

6. **Root type registration updated:**
   ```bash
   grep -rn 'namespace ReactNavigation' src/
   ```
   Should show `RootNavigator` module augmentation (8.x), not `RootParamList` (7.x).

7. **Hook generics removed:**
   ```bash
   grep -rn 'useNavigation<\|useRoute<\|useNavigationState<' src/ --include='*.ts' --include='*.tsx'
   ```
   Zero matches. Hooks no longer accept generics in 8.x.

8. **App runs and navigates correctly:** Manual smoke test — navigate to every major screen, test deep links, test back button, verify tab headers visible.

## References

Check `@react-navigation/native` in `package.json` first.

- If `6.x`, read [`references/upgrade-6-to-7.md`](./references/upgrade-6-to-7.md)
- If `7.x`, read [`references/upgrade-7-to-8.md`](./references/upgrade-7-to-8.md)

Load exactly one reference file unless explicitly comparing versions.
