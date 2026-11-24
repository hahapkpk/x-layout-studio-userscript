# X Layout Studio

Tampermonkey userscript that takes back control of x.com's layout: drag the
timeline edge to resize it, reclaim the empty right rail, and declutter the shell.

## Features

- **Drag to resize** — grab the right edge of the timeline and drag. Width is
  remembered. Double-click the edge to reset.
- **Hide the right rail** — X renders an empty 424px `<aside>` on the timeline
  (the trends/suggestions column is often blank). This reclaims it.
- **Compact nav** — collapses the left navigation from 256px to 72px (icon only).
- **Hide the tabs** — removes the 为你推荐 / 正在关注 header row.
- **Hide view counts** — removes the view counter from the action bar.
- **Remove borders** — drops the column separators.
- **Hide promoted posts** — detects X's ad label ("来自 example.com 的广告" /
  "Promoted by …") and hides the whole post.
- **全宽模式** — one click: rail hidden + compact nav + width maxed out.

Settings persist per browser via `GM_setValue`. A small floating button on the
right edge opens the panel; the Tampermonkey menu has "打开面板" / "重置布局设置".

## Install

https://raw.githubusercontent.com/hahapkpk/x-layout-studio-userscript/main/x-layout-studio.user.js

## Changelog

### 0.2.1

- **Compact nav no longer eats the icons.** The old rule hid `nav a > span > span:last-child`,
  which on the XDS markup is the element wrapping the icon as well as the label, so the
  whole sidebar went blank. XDS now leans on X's own collapse switch
  (`--x-sidebar-expanded: 0`), and labels are flagged at runtime by finding the outermost
  element inside each nav item that holds text and contains no `<svg>` — an icon can never
  be caught by that test. Verified: icons 4/4 visible, labels hidden, nav 272 → 88px.
- **Settings can no longer be clobbered.** `pagehide`/`visibilitychange` used to write the
  in-memory settings unconditionally, so a stale tab could overwrite a change just made in
  another tab (or on the page you navigated away from). Writes are now gated on a dirty
  flag and only flush when something actually changed.

### 0.2.0

- **Works on both X frontends.** X flips accounts between the new XDS/Tailwind shell
  (CSS variables + grid) and the older React-Native-Web shell (`primaryColumn`, atomic
  classes, fixed 600px). 0.1.x only understood XDS, so on a legacy-shell account the
  script booted but changed nothing. The RNW shell is now handled by
  `primaryColumn{max-width,width}` + clearing the `r-1ye8kvj` max-width wrapper.
- **Width persistence hardened.** During a drag the handle is only 5px wide, so the
  pointer leaves it immediately; the old code listened on the handle itself and could
  miss `pointerup` entirely, never saving. Listeners now live on `window` for the
  duration of the drag, the width is written on a 120ms debounce while dragging, and
  again on `pointerup` / `pagehide` / `visibilitychange`.
- The stored width is applied on every load and survives refresh. A narrower window
  clamps what is *rendered* without overwriting the stored preference.
- Orphan UI from a previous copy is removed at boot, so a stale panel can no longer
  show numbers the live instance does not control.

### 0.1.1

- **Promoted posts**: detection was missing ads whose label contains a nested link
  and ads rendered outside an `<article>`. It now matches any element whose entire
  text is the ad label, then walks up to the owning post (or the virtualised row
  cell as a fallback, with a size guard so a page-level container can never be
  hidden). Verified: 3/3 ads hidden, 0 false positives, 0 visible ad labels left.
- **Single instance**: a second copy (Tampermonkey install + manual injection)
  would each rewrite the same `<style>` from its own settings and make toggles
  flap. Only the first instance now boots.

### 0.1.0

Initial release.

## How it works (and why it survives redesigns)

X drives its shell from CSS custom properties, so this script moves no DOM
nodes — it rewrites variables and a few `max-width`/`display` rules:

| Variable | Effect |
|---|---|
| `--x-main-column-width` | the column shell: grid track + `main` max-width |
| `--x-timeline-max-width` | the content wrapper *inside* main — widening only the first leaves the timeline stuck at 600px, centred in a wide empty column |
| `--x-sidebar-expanded` | `0` = 72px icon-only nav, `1` = 256px labelled nav |

Shell geometry observed at 1491×882:

```
div.max-w-[1024px]                       (outer, mx-auto)
  div.flex.w-full.grow.justify-center
    div.grid.lg:grid-cols-[var(--x-main-column-width)_…]   -> "600px 424px"
      main   600px  max-w-(--x-main-column-width)
      aside  424px  "hidden flex-col lg:flex"   <- empty on the timeline
div.sticky  nav shell  w-[calc(72px+184px*var(--x-sidebar-expanded))]
```

Notes for future maintenance:

- Selectors use `[class*="…"]` fragments, not exact class lists, because X
  reorders Tailwind classes between builds.
- State is **not** stored in `<html>` classes: X rewrites `className` on
  re-render and wipes them. The stylesheet is regenerated from settings instead.
- The shell is re-created on client-side navigation, so the resize handle is
  re-attached whenever `main` changes identity.

## License

MIT.
