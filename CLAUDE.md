# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm start` — Vite dev server on port 8000, opens `/demo/`, with watch-mode build running in parallel.
- `npm run build` — Production library build via Vite/Rollup. Outputs `dist/widget-toast.js` (ES module).
- `npm run watch` — `vite build --watch` only (no dev server).
- `npm run types` — Regenerate `src/definition-schema.d.ts` from `src/definition-schema.json` using `json2ts`. Run this after editing the JSON schema.
- `npm run release` — `npm version patch`: preflight guards (on `main`, clean tree, not behind `origin/main`, generated files current, build passes), then the bump and a `dist/` rebuild (the `version` hook, so the local build registers the new tag), then commit, bare-semver tag, `git push --follow-tags`, then waits on the CI run and fails if the npm publish fails. `npm run release:minor` / `release:major` for other bumps. `npm version` creates the **annotated** tag that `--follow-tags` needs (a plain `git tag` is lightweight and `--follow-tags` silently skips it). The bare `v`-less prefix comes from `tag-version-prefix=""` in `.npmrc`.
- `npm run link` / `npm run unlink` — Link this package into a sibling `../RESWARM/frontend` checkout for local integration testing.

No test runner or lint script is wired up despite README mentioning `lint`/`format` — those scripts do not exist in `package.json`. ESLint/Prettier configs are present (`.prettierrc`, eslint deps) but invocation is manual.

Node `>=24.9.0`, npm `>=10.0.2` (per `engines`).

## Architecture

This repo publishes `@ironflock/widget-toast`, a single Lit web component consumed by the IronFlock/RESWARM platform frontend as a dashboard widget. It shows transient toast notifications (success / error / info / warning).

### Entry point and integration

- `src/widget-toast.ts` defines the Lit element. The custom element tag is `widget-toast-versionplaceholder` — the literal string `versionplaceholder` is replaced at build time by `@rollup/plugin-replace` (see `vite.config.ts`) with `pkg.version`. This versioned tag name lets multiple widget versions coexist on the same page (the host app reads the version from `package.json` and constructs the tag dynamically — see `demo/index.html`).
- The host platform passes data via two reactive properties: `inputData: ToastNotificationConfiguration` (the message + config) and `theme: { theme_name, theme_object }`. Theme can also be supplied via CSS custom properties `--re-text-color` and `--re-tile-background-color`. The text colour is not snapshotted: `registerTheme()` stores a `var(--re-text-color, <theme value>)` chain, so a change to the host property repaints live without the widget being told. The card background is read at render time instead (every new toast renders): `surfaceColor()` composites the tile colour (`--re-tile-background-color`, which a per-tile style can set for this widget) over the dashboard's `--re-background-color` over white into an opaque `#rrggbb`, so the cards look like a tile and `--re-text-color` reads on them. The browser parses each colour through a canvas `fillStyle` and the blend is plain source-over in JS; never read canvas pixels for it, since Firefox's `resistFingerprinting` randomises them.
- **This widget is fully self-contained — it has NO `echarts` and NO peer dependencies.** `lit` and `tslib` are bundled into `dist/widget-toast.js` (unlike `widget-doughnut`, the Rollup config does not externalize anything).

### How "listening to a topic" works

Widgets in this ecosystem are purely presentational and never open their own MQTT/WebSocket connections. The host (`IronFlock-UI/src/components/widget-item.ts`) subscribes to the backend over WAMP and pushes the latest row into the widget's `inputData`. So a user "configures a topic the toast listens to" by binding the toast's `message` (and `type`) field to a backend table column in the dashboard editor; when the backend writes a row, the host replaces `inputData` with a **new object reference** and the widget reacts. There is intentionally no literal `topic` field in the widget.

### Data schema

- `src/definition-schema.json` is the source of truth for the input shape. It is consumed both as runtime documentation by the platform's widget configuration UI (note the `order`, `dataDrivenDisabled`, and rich `description` fields, which are platform-specific extensions read by the IronFlock dashboard editor) and compiled to `src/definition-schema.d.ts` via `bun run types`. The component imports `ToastNotificationConfiguration` from the generated `.d.ts`.
- `message` and `type` (severity) are **data-driven** (no `dataDrivenDisabled`) — they reflect the latest published row. `displayTime`, `position`, `maxVisible`, `showIcon`, `showCloseButton`, and the per-severity color overrides are **static config** (`dataDrivenDisabled: true`).
- Note: `json2ts` maps `"type": "color"` fields to *empty interfaces*, not `string`. The component therefore reads the `*Color` overrides defensively (indexed `Record<string, unknown>` cast + `typeof === 'string'` check) in `accentColor()`.
- `src/default-data.json` is sample data used by the demo.

### Toast pipeline (inside `widget-toast.ts`)

1. **Trigger** — the host replaces `inputData` on each new backend row, so `update()` sees `changedProperties.has('inputData')`. The first such update is suppressed (it carries the stale last-persisted value on dashboard load). Every subsequent push with a non-empty `message` enqueues exactly one toast. Identical consecutive messages are intentionally NOT deduped — each push is a real event. (Tradeoff: editing a static config field in the dashboard editor also replaces `inputData` and can fire one spurious toast; accepted — the clean fix would be host-side.)
2. **`enqueueToast()`** — assigns a unique id (`Date.now() + counter`), enforces `maxVisible` by hard-dropping the oldest active toasts (clearing their timers first), and schedules a `setTimeout(displayTime)` dismissal unless `displayTime === 0` (persistent).
3. **`dismissToast()`** — sets the toast's `leaving` flag to play the CSS exit transition, then removes it after `LEAVE_MS`. **`LEAVE_MS` must equal the `transition` duration in the `.toast` style rule** — they are a coupling point.
4. **Timers** — all `setTimeout` handles live in a `Map` keyed by toast id and are cleared in `disconnectedCallback()`.
5. **Rendering** — an absolutely-positioned `.stack` inside a `position: relative; overflow: hidden` host. The `position` enum maps to CSS anchors (bottom positions use `flex-direction: column-reverse`). Each toast is a card with a colored left border, optional severity icon, the message (`white-space: pre-wrap`), and an optional close button. The card is `color-mix(in srgb, <opaque surface> 85%, transparent)` with a 3px backdrop blur: always 85% opaque, because the base is opaque. Mixing the tile colour itself used to multiply its alpha, which left cards see-through (or without any background) on translucent and transparent tiles. Studio paints no tile background or border behind this widget (`widget-item`'s `is-overlay`), since its tile floats above other widgets; the cards are the only visible surface. Sizing uses container-query units (`cqw`/`cqh`, `clamp()`), so no `ResizeObserver` is needed.

Default severity colors: success `#2e7d32` ✓, error `#c62828` ✕, info `#1565c0` ℹ, warning `#ed6c02` ⚠.

Not implemented (possible future enhancements): pause-auto-dismiss-on-hover and a `prefers-reduced-motion` opt-out.

### Build pipeline

`vite.config.ts` configures a library build:

- Single ES-module output (`dist/widget-toast.js`) with sourcemaps and a license banner.
- Nothing is externalized (self-contained widget; `lit`/`tslib` are bundled).
- `@rollup/plugin-replace` substitutes `versionplaceholder` → current `package.json` version (affects the custom element tag name and the `version` class field).
- `process.env.NODE_ENV` is hardcoded to `'production'`.

### Release flow

Tags pushed to GitHub trigger `.github/workflows/build-publish.yml`: `npm ci`, build, then `npm publish --access public` via npm trusted publishing (OIDC — no `NPM_TOKEN`), and a GitHub Release is created. `npm run release` is the canonical local command — it produces bare semver tags (e.g. `1.0.0`, not `v1.0.0`) to match the rest of the widget repos.

## `aiSelection` in `src/definition-schema.json`

The schema root carries an `aiSelection` block next to `title` and `description`. It is **not** JSON Schema and describes no config field — it exists so the IronFlock AI's Widget Builder can pick the right widget for a given shape of data, using knowledge only the widget author has:

```jsonc
"aiSelection": {
  "dataShape": "…what columns this widget consumes and what each one means…",
  "useWhen":   ["…a situation, naming the properties that express it…"],
  "notFor":    ["…a situation this widget is wrong for, naming the widget to use instead…"]
}
```

It is inert everywhere else, and must stay that way: `json2ts` ignores it (the generated `.d.ts` is byte-identical with and without it), the dashboard config editor renders only `schema.properties`, and the AI service's `validate_widget` validates *configs* against the schema, skipping unknown Draft-7 keywords.

When maintaining it:

- `notFor` is the high-value half and the part plain descriptions always omit. Every entry must name the widget that *should* be used, or it rejects without routing.
- Write for an LLM with no other documentation: describe the visible result and the user's intent, not the implementation.
- Prefer entries that discriminate against a *neighbouring* widget. Generic rejections are cheap; the ones that pay are those an author could plausibly get wrong.
- The `notFor` lists are a set across all `widget-*` repos and are meant to be reciprocal — if this widget routes to another for some case, that widget should usually route back for the converse. Changing one side is a cue to check the other.
- Update it whenever a property changes what this widget can *do*, not just how it looks.
