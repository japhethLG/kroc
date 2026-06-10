# KROC — Zesty WebEngine

Native HTML / CSS / JS site served by a Zesty WebEngine instance. The `webengine/`
folder is synced to the cloud instance (`8-80c0bba6b8-lhkcc7`). Styling is
**Tailwind CSS v4**, compiled locally into a single stylesheet that Zesty serves.

## Styling / Tailwind

Tailwind is used as a **build-time tool only** — no Tailwind JS or CDN ships to the
browser. The source of truth is [`src/tailwind.css`](src/tailwind.css); the compiled,
purged output is [`webengine/styles/tailwind.css`](webengine/styles/tailwind.css). The
build then **inlines** that CSS into the [`components/styles.html`](webengine/views/components/styles.html)
snippet, which the loader includes.

> **Why inlined, not a linked stylesheet?** Zesty combines + compiles + minifies
> everything in `styles/` into a single `/site.css`. Running Tailwind v4's compiled
> output (cascade `@layer`, `@property`, nested at-rules) through that pipeline
> **mangles it** — it drops `@layer components` and most utilities. A snippet is HTML,
> so Zesty delivers it verbatim. The compiled CSS contains no `{{`, so Parsley leaves
> it alone. (`webengine/styles/tailwind.css` is just the build artifact the inliner
> reads — it does not need to be the served stylesheet.)

```bash
npm install        # one-time
npm run watch:css  # rebuild on change while developing
npm run build:css  # one-off minified build (run before syncing to Zesty)
```

> **Workflow note:** Tailwind purges unused classes, so a class only exists in the
> output CSS if it appears in a scanned file. Author markup **locally** and rebuild
> before syncing. Classes added directly in the Zesty cloud Manager won't have CSS
> until the next local build picks them up (`@source` globs scan `webengine/views`
> and `webengine/scripts`).

`node_modules/`, `package.json`, and `src/` live outside `webengine/` and are **not**
synced to the instance.

## Design tokens (3-tier)

Values come from the KROC Web Design System (`design-system.md`). Never hardcode hex
values in markup — use the token utilities.

| Tier | What | Examples |
|---|---|---|
| **Primitives** | Raw brand palette (fixed) | `bg-brand-red-200`, `text-brand-navy-150` |
| **Semantic** | Author with these | `bg-primary`, `bg-secondary`, `bg-surface`, `text-content`, `text-content-muted`, `rounded-card`, `rounded-button` |
| **Profiles** | Runtime theme swap | `[data-theme="dark"]`, `[data-theme="sharp"]` |

Semantic utilities resolve to runtime CSS variables (`--ui-*`), so **theming is a
runtime attribute swap, not a rebuild**:

```js
// switch the whole site to the dark profile
document.documentElement.setAttribute("data-theme", "dark");
localStorage.setItem("kroc-theme", "dark"); // persisted; applied by the loader
```

To add a profile or change a mapping, edit the `:root` / `[data-theme="…"]` blocks
in `src/tailwind.css` and rebuild. To add a brand-new token value, add it to the
design system first, then to `src/tailwind.css`.

Other token utilities: type scale (`text-heading-xl` … `text-caption`), elevation
(`shadow-header`, `shadow-card`, `shadow-active`, `shadow-search`), fonts
(`font-sans` = Creato Display, `font-serif` = Adobe Jenson Pro*). Spacing uses
Tailwind's default 4px scale (`p-5`=20px, `p-6`=24px, `p-8`=32px, `p-16`=64px).

\* Adobe Jenson Pro `.ttf` is not yet in the repo — `font-serif` falls back to
Georgia until the font file is added and the `@font-face` stub in
`src/tailwind.css` is wired up.

## Loading model

The instance `<head>` is managed by Zesty and not editable here, so everything we
control loads through the **loader** ([`webengine/views/loader`](webengine/views/loader)),
a snippet injected into `<body>`. The loader composes the page:

```
{{ include /components/styles.html }}   <!-- inlined Tailwind CSS -->
{{ include /components/icons.html }}    <!-- SVG sprite -->
{{ include /components/header.html }}
<main>{{ current_view }}</main>
{{ include /components/footer.html }}
```

Web fonts are `@import`ed at the top of the compiled CSS, so they load with the inlined
styles. (We do **not** use Zesty's auto-linked `/site.css` for Tailwind — see the
styling note above for why.)
