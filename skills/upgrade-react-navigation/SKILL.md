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

## References

Check `@react-navigation/native` in `package.json` first.

- If `6.x`, read [`references/upgrade-6-to-7.md`](./references/upgrade-6-to-7.md)
- If `7.x`, read [`references/upgrade-7-to-8.md`](./references/upgrade-7-to-8.md)

Load exactly one reference file unless explicitly comparing versions.
