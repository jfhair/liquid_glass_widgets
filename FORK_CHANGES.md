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

## Change #2 — `searchIcon` override on `GlassSearchBarConfig`
(PR-shaped, additive — see below)

**Why:** the upstream collapsed search pill hardcodes
`Icon(CupertinoIcons.search, color: iconColor)`. `cupertino_icons`
is an aging static font whose `search` glyph doesn't match SF Pro's
`magnifyingglass`: thinner stroke, handle/circle disconnect, smaller
circle proportionally. For a package that explicitly mimics iOS 26
Liquid Glass design, the search-pill icon should match what real
iOS renders.

Because this is a stylistic-default question more than a clear bug,
the cleanest path is to **expose the icon as caller-overridable**
rather than picking one "best" default and arguing for it. Apps that
care about exact iOS fidelity can pass `Icon(SFSymbol.magnifyingglass,
...)` from `flutter_sficon`; apps wanting just a heavier weight can
pass `Icon(Symbols.search, weight: 500, ...)` from
`material_symbols_icons`; apps fine with the current default just
don't pass anything.

Hero Dirt currently passes a Material Symbols search icon via this
override (see `ios_shell.dart`).

**Approach (the upstream-PR shape):** added `searchIcon: Widget?`
field on `GlassSearchBarConfig` (default `null`). When `null`, the
existing `Icon(CupertinoIcons.search, color: iconColor)` literal is
used — zero behavior change for any existing caller. When non-null,
the supplied widget is used in place of the default.

**Affected files:**
- `lib/widgets/surfaces/shared/glass_search_bar_config.dart` —
  added `final Widget? searchIcon;` + ctor param + dartdoc
- `lib/widgets/surfaces/shared/searchable_bottom_bar_internal.dart`
  — call site now reads
  `widget.config.searchIcon ?? Icon(CupertinoIcons.search, color: iconColor)`

**Discussion status:** open a discussion upstream proposing either
(a) replacing the hardcoded default with a real SF Symbol via
`flutter_sficon` for closer iOS fidelity, OR (b) accepting this PR
to expose `searchIcon` as a caller override. Either solves the
problem; (b) is more flexible and respects different apps' icon-set
preferences.

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
