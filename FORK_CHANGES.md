# Hero Dirt fork — local modifications

This fork carries two local changes on top of upstream
[`sdegenaar/liquid_glass_widgets`](https://github.com/sdegenaar/liquid_glass_widgets)
`v0.9.6`. Used by the Hero Dirt app via a `dependency_overrides: git:` entry
in `hero_dirt/pubspec.yaml` that pins to the `hero-dirt-expansion-only`
branch.

1. **`indicatorExpansion` parameter** (PR-shaped, additive — see below)
2. **Search-pill icon swap** (local-only — `CupertinoIcons.search` →
   `Icons.search_rounded` size 31). A respectful upstream discussion
   has been opened to ask about the icon choice and possibly expose
   it as a caller-overridable param. If/when that lands, this local
   patch goes away.

## Branches

- **`hero-dirt-expansion-only`** (default for the Hero Dirt app) —
  exposes a new `indicatorExpansion: double = 14` parameter on
  `GlassBottomBar` and `GlassSearchableBottomBar`. Same default as
  before, so behavior is unchanged for callers that don't set the
  param. Hero Dirt sets it to `5`.
- **`indicator-behind-icons`** — historical, the focused PR change
  submitted upstream as
  [#29](https://github.com/sdegenaar/liquid_glass_widgets/pull/29) and
  merged in `v0.9.3`. Now part of the published package — no longer
  needed as a fork carrier.
- **`hero-dirt-local`** — historical, prior fork branch that carried
  three local tweaks (jelly expansion, search-pill icon, tab label
  weights). Replaced by `hero-dirt-expansion-only` after we accepted
  upstream behavior on the icon and label-weight tweaks; only the
  jelly-expansion change remained worth carrying.

If/when upstream releases a new version, **read this doc first**, then
sync with: `git fetch upstream && git rebase upstream/main` from the
`hero-dirt-expansion-only` branch. Conflicts should only land on the
file:line areas listed below.

---

## The change — `indicatorExpansion` parameter

**Why:** the upstream default morph "puff" of `expansion: 14` is
roughly 2× more dramatic than iOS's native tab bar feel. We dial it
down to `5`.

**Approach (the upstream-PR shape):** instead of editing the literal
in place, we expose a new public parameter `indicatorExpansion: double`
on `GlassBottomBar` and `GlassSearchableBottomBar`, default `14`
(matching the previous behavior — zero breaking changes), and plumb
it through `TabIndicator` / `SearchableTabIndicator` to the existing
`expansion:` callsites on `JellyClipper` and `AnimatedGlassIndicator`.

This is the structure of the proposed upstream PR. Once it merges,
this fork becomes redundant — the published package will accept the
same parameter, and Hero Dirt can drop the `dependency_overrides`
entry.

**Affected files (10 callsites + 4 ctor/field plumbing edits):**

`lib/widgets/surfaces/shared/bottom_bar_internal.dart`:
- `TabIndicator` constructor: added `indicatorExpansion = 14`
- `TabIndicator` fields: added `final double indicatorExpansion;`
- 5 occurrences of `expansion: 14` → `expansion: widget.indicatorExpansion`

`lib/widgets/surfaces/shared/searchable_bottom_bar_internal.dart`:
- `SearchableTabIndicator` constructor: added `indicatorExpansion = 14`
- `SearchableTabIndicator` fields: added `final double indicatorExpansion;`
- 5 occurrences of `expansion: 14` → `expansion: widget.indicatorExpansion`

`lib/widgets/surfaces/glass_bottom_bar.dart`:
- `GlassBottomBar` constructor: added `indicatorExpansion = 14`
- `GlassBottomBar` fields: added `final double indicatorExpansion;`
- `TabIndicator(...)` callsite: pass `indicatorExpansion: widget.indicatorExpansion`

`lib/widgets/surfaces/glass_searchable_bottom_bar.dart`:
- `GlassSearchableBottomBar` constructor: added `indicatorExpansion = 14`
- `GlassSearchableBottomBar` fields: added `final double indicatorExpansion;`
- `SearchableTabIndicator(...)` callsite: pass `indicatorExpansion: widget.indicatorExpansion`

**Find with:**

```sh
grep -rn "expansion: widget.indicatorExpansion" lib/widgets/surfaces/shared/
grep -rn "indicatorExpansion" lib/widgets/surfaces/
```

---

---

## Local change — search-pill icon size

**Why:** the upstream collapsed search pill renders
`Icon(CupertinoIcons.search, color: iconColor)` at default size ~24,
which reads as small relative to surrounding tab icons sized 30 in
Hero Dirt's config. Bumped to size 30 — proportional with the rest
of the bar without changing the glyph itself.

**Affected line (1 spot):**
- `lib/widgets/surfaces/shared/searchable_bottom_bar_internal.dart`
  — inside the `GlassButton` for the collapsed pill (around line 709).
  Just adds `size: 30` to the existing `Icon(CupertinoIcons.search,
  ...)` literal.

**Why not also bump the weight:** `cupertino_icons` is a static
(non-variable) font — `Icon(...)`'s `weight` parameter is silently
ignored on it, and the glyph it ships is closer to SF Pro
magnifyingglass at the Light/Regular weight rather than the heavier
Medium/Semibold that App Store and Apple Music actually use.

To match the real iOS magnifyingglass weight we'd need either
(a) `flutter_sficon` for actual SF Symbols (variable weight, exact
shape match) or (b) `material_symbols_icons` for Material 3's
variable-font search icon. Both are heavier deps than just bumping
the size — deferring that decision until after the upstream
discussion concludes.

**Discussion status:** opening a discussion upstream asking about
the design intent and proposing SF Symbols (via something like
`flutter_sficon`) as the "actual right" default for an iOS
Liquid Glass design package. Result will inform whether this local
override stays as-is or gets replaced with a heavier-glyph approach.

---

## Changes that were rolled back

Two local tweaks from the previous `hero-dirt-local` branch were
considered but not carried forward:

1. **Tab label weights** (`w700/w600` selected/unselected, was
   `w600/w500`) — minor enough that the upstream defaults are
   acceptable.
2. **Indicator-behind-icons** — landed upstream in `v0.9.3` as our
   PR #29. Now the default behavior, no opt-in needed.

---

## Maintenance protocol

When upgrading to a newer upstream version:

1. `cd /Users/jhair/Developer/liquid_glass_widgets-fork`
2. `git checkout hero-dirt-expansion-only`
3. `git fetch upstream`
4. `git rebase upstream/main` (or `upstream/<some-tag>`)
5. Conflicts will land on the file:line areas above. Re-apply the
   parameter plumbing.
6. `git push --force-with-lease origin hero-dirt-expansion-only`
7. In Hero Dirt: `flutter pub get` to refetch the new commit, hot
   restart, smoke test the bottom bar.

If the upstream PR lands, drop the `dependency_overrides` block in
Hero Dirt's `pubspec.yaml` entirely and use the published package.
The Hero Dirt callsite already passes `indicatorExpansion: 5` — no
app-side change needed.
