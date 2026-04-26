# Hero Dirt fork — local modifications

This fork of [sdegenaar/liquid_glass_widgets](https://github.com/sdegenaar/liquid_glass_widgets)
carries Hero Dirt-specific changes on top of upstream `v0.8.3`. Used by the
Hero Dirt app via a `dependency_overrides: git:` entry in `hero_dirt/pubspec.yaml`
that pins to this `hero-dirt-local` branch.

## Branches

- **`indicator-behind-icons`** (1 commit) — the focused PR change submitted
  upstream as [sdegenaar/liquid_glass_widgets#29](https://github.com/sdegenaar/liquid_glass_widgets/pull/29).
  Adds an opt-in `indicatorBehindIcons: bool = false` flag. Backward-compat,
  ready to merge as-is.
- **`hero-dirt-local`** (default for the Hero Dirt app) — built on top of
  `indicator-behind-icons` with three additional local-only tweaks (#2-#4
  below) that are stylistic preferences not slated for upstream.
- **`hero-dirt-modifications`** (deprecated) — earlier version of the same
  branch before the PR was extracted out. Kept for history.

If/when upstream releases a new version, **read this doc first**, then sync
with: `git fetch upstream && git rebase upstream/main` from the
`hero-dirt-local` branch. Conflicts should only land on the lines listed
below.

---

## 1. Indicator drawn BEHIND icons (the PR)

**Problem upstream:** the moving glass indicator is painted *on top of* the
icons in a Stack. Because the indicator's lens has a translucent fill, the
active icon's color gets washed out — a saturated blue/red/green icon under
the indicator looks gray.

**Fix:** added an opt-in `indicatorBehindIcons: bool = false` flag on
`GlassBottomBar` and `GlassSearchableBottomBar`. When `true`, the indicator
paints first and icons paint over it.

**Affected lines:**
- `lib/widgets/surfaces/glass_bottom_bar.dart` — added field + plumbed to internal
- `lib/widgets/surfaces/glass_searchable_bottom_bar.dart` — same
- `lib/widgets/surfaces/shared/bottom_bar_internal.dart` — `TabIndicator` ctor
  param; conditional Stack ordering in `_buildSimpleMode` and
  `_buildHighQualityMode`
- `lib/widgets/surfaces/shared/searchable_bottom_bar_internal.dart` — same
  pattern in `_buildSimple` and `_buildHighQuality`

**Status:** PR open at https://github.com/sdegenaar/liquid_glass_widgets/pull/29.
If/when it merges and ships in a new version, we can drop our fork override
and just set `indicatorBehindIcons: true` on the published package.

---

## 2. Indicator / jelly expansion: 14 → 5

**Why:** the package's default morph "puff" is roughly 2× more dramatic than
iOS's native tab bar feel. We dial it down.

**Affected lines (8 spots, all `expansion: 14,` literals — ours are `5`):**
- `bottom_bar_internal.dart` — 4 places (in both quality builders, on both
  the `JellyClipper` and `AnimatedGlassIndicator` calls)
- `searchable_bottom_bar_internal.dart` — 4 places, same pattern

**Find with:** `grep -rn 'expansion: 1[0-9]' lib/widgets/surfaces/shared/`

If upstream adds new call sites, replace `14 → 5` consistently.

---

## 3. Search pill icon: `Icons.search_rounded` size 31

**Why:** the package defaults to `CupertinoIcons.search` at default size 24,
which reads thin/small relative to iOS App Store's chunkier magnifier.

**Affected line (1 spot):**
- `searchable_bottom_bar_internal.dart` — inside the `GlassButton` that
  renders the collapsed search pill (around line 645).

**Pattern to look for:**
```dart
icon: Icon(CupertinoIcons.search, color: iconColor),
```
**Replaced with:**
```dart
icon: Icon(Icons.search_rounded, color: iconColor, size: 31),
```

---

## 4. Tab label weights: w700 / w600 (was w600 / w500)

**Why:** Inter at the package's default weights reads slightly anemic in our
dark theme. Bumped both states up one notch, preserving the
selected-vs-unselected differentiation.

**Affected line (1 spot):**
- `bottom_bar_internal.dart` — the conditional inside the `Text(tab.label!,
  style: textStyle ?? TextStyle(...))` block (around line 204).

**Pattern to look for:**
```dart
fontWeight: selected ? FontWeight.w600 : FontWeight.w500,
```
**Replaced with:**
```dart
fontWeight: selected ? FontWeight.w700 : FontWeight.w600,
```

The same conditional only exists in `bottom_bar_internal.dart` — the
searchable variant reuses this code path.

---

## Maintenance protocol

When you want to upgrade to a newer upstream version:

1. `cd /Users/jhair/Developer/liquid_glass_widgets-fork`
2. `git checkout hero-dirt-local`
3. `git fetch upstream`
4. `git rebase upstream/main` (or `upstream/<some-tag>`)
5. Conflicts land on one or more of the file:line areas above. For each:
   - Look at upstream's diff (it tells you what they changed)
   - Re-apply our intent (e.g. swap Stack children → use the `indicatorBehindIcons` plumbing; change `expansion: 14 → 5`)
   - Mark resolved
6. `git push --force-with-lease origin hero-dirt-local`
7. In the Hero Dirt project: `flutter pub get` to refetch the new commit, hot restart, smoke test.

Realistic conflict cost per upstream release: ~5 minutes for someone who has
this doc open. Two of the four changes (#3 search icon, #4 weights) are
single-line edits — trivial to redo even if upstream rewrites everything
around them.

If the PR (#1) merges upstream, change #1's portion of this doc becomes
unnecessary — drop the fork override entirely on that one and use the
published package's `indicatorBehindIcons: true` flag.
