# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Deployment

Pushing to `main` triggers GitHub Actions (`.github/workflows/static.yml`), which deploys the entire repo as a static site to GitHub Pages. There is no build step — what's in the repo is what's served.

## Architecture

The entire site is a single file: `index.html`. It contains all CSS (in `<style>`), all HTML structure, and all JavaScript (in `<script>`). There are no external dependencies beyond the Google Fonts CDN (Familjen Grotesk + IBM Plex Mono).

### Updating content

All project data lives in `projects.json` at the repo root. It has two arrays: `threed` and `design`. The site fetches this file at runtime.

Each project object supports these fields:
```json
{
  "title":       "Project Name",
  "category":    "Category Label",
  "type":        "image",
  "src":         "images/web/my-project.webp",
  "thumb":       "images/thumbs/my-project.webp",
  "embedUrl":    "",
  "fit":         "contain",
  "description": "Optional 1–2 sentence context shown in the lightbox",
  "gradient":    "135deg, #1a1e2a, #1a2640"
}
```

- `type: "image"` — `src` is shown in the lightbox, `thumb` in the grid card; omit both to show a gradient placeholder
- `type: "video"` — set `embedUrl` to a framerate.tv embed URL and `thumb` for the card; leave `src` empty
- `fit: "contain"` — legacy flag carried by the `design` items. The site no longer letterboxes: **grid cards are always cover-cropped**, and the **lightbox shows each image at its own aspect ratio**. `fit` is now read only by `admin.html`'s edit-modal preview.
- `description` — optional; rendered under the title in the lightbox when present
- `gradient` — used as card background and lightbox fallback when no media is set

The demo reel embed URL is set via `reelUrl` in `projects.json`.

### Image pipeline

Originals live in `images/3d/` (and `images/thumbnails/` for video stills). Derived web assets are generated with ImageMagick (`magick` is on PATH via Maya):

```sh
# 800px card thumbnail (~30-60 KB)
magick images/3d/foo.jpg -auto-orient -resize 800x -quality 80 images/thumbs/foo.webp
# 1920px lightbox version (~150-250 KB)
magick images/3d/foo.jpg -auto-orient -resize "1920x1920>" -quality 85 images/web/foo.webp
```

Never point grid cards at the multi-MB originals — always generate a thumb. The `images/og-image.jpg` (1200x630) is the social share preview referenced from the `og:image` meta tag.

### Tabs and default tab

The `3D` / `Design` tab buttons (`class="tab-btn"`) and the `All` / `Animated` / `Still` sub-tabs live in `index.html`. The button carrying `class="tab-btn active"` is the default, and the initial `renderCards()` call inside the `fetch` callback must match it (`threedProjects`). Design items are all `type: "image"`.

### Design tokens

The palette lives in CSS custom properties on `:root` and under `html[data-theme="light"]` in `index.html`: `--bg`, `--surface`, `--border`, `--text`, `--muted`, `--accent`, plus `--nav-bg`, `--overlay`, `--chip-bg`, `--chip-border`. Dark is the default ("darkroom"); light is the "paper proof sheet".

The initial theme is set by a small inline `<script>` in `<head>` (before first paint, so light-mode visitors get no dark flash): a saved choice wins, otherwise it follows the OS `prefers-color-scheme`. Only an explicit click on the nav toggle writes `localStorage`, so visitors who never click keep following their OS. **Every `localStorage` access must be wrapped in `try/catch`** — it throws when site data is blocked, and an uncaught throw would stop the whole page script (no projects, no lightbox, no CV menu).

### Stacking and overlays

- `.masthead` is `z-index: 20`, above `.section` (`10`), so the CV dropdown isn't covered by the reel section. Keep the masthead above the sections if you touch these values; nav is `100`, lightbox `300`, film grain `400`.
- The CV dropdown button carries `aria-expanded`; Escape closes the menu and returns focus to the button.
- The lightbox uses `inert`: it is `inert` while closed (its buttons can't be tabbed to), and while open the rest of the page's top-level elements are `inert` so focus stays in the viewer. Open it via `showLightbox()` and close via `closeLightbox()` (which restores focus to the card that opened it) rather than toggling the `active` class directly.

## Contact email

The contact email in the Contact section is deliberately NOT in the HTML source — it is assembled at runtime by JS (reversed string parts) to defeat spam scrapers. Never put the plain address back into the markup.

## Analytics

GoatCounter (cookieless, no consent banner needed) — script tag at the bottom of `index.html`, site code `ytse77`, dashboard at https://ytse77.goatcounter.com. Google Analytics was removed in July 2026; do not re-add it without a consent banner.

## Local tooling (gitignored)

`admin.html`, `serve.sh`, `serve.bat`, `publish.sh`, `index.old.html`, and the `trash/` folder are **gitignored on purpose** — local only, never deployed, and must be copied manually to other machines (they won't arrive via `git clone`). Because they don't sync, copies can drift between machines — check that a machine's `admin.html` matches the description below (e.g. it has **Save to repo folder**) before relying on it.

### admin.html — content editor

A self-contained browser GUI for content updates. Its styling mirrors the live site (same design tokens, fonts, amber accent and film grain).

- Loads `projects.json`: auto-fetch when served over http; otherwise a file picker. When opened as `file://`, `fetch` is blocked, so the page disables auto-load and points you at **Choose file…**.
- Visual card grid — **drag cards to reorder**, click a card to open an editor modal (title, category, description, embed URL; plus move/delete).
- Adding a project: drop an image; WebP derivatives are generated in-browser via Canvas (800px thumb + 1920px lightbox version for 3D stills; thumb only for videos; design logos pass through untouched).
- **Save to repo folder** — writes `projects.json` and all new images straight into the repo, creating subfolders as needed (File System Access API; **Chrome/Edge only**, and only in a secure context). The folder is remembered between visits. In other browsers the button is disabled and the `projects.json only` / `Download ZIP` exports remain.
- **Removing a project → `trash/`** — Delete in the edit modal only queues the project's files; nothing moves until the next **Save to repo folder**. The save then *copies* each file to `trash/<YYYY-MM-DD_HHMM>/<same repo path>` and removes it from `images/` (copy before remove, so a failure never loses a file; restoring is a straight copy back). Files moved: thumb + lightbox WebP, plus the archival original — exact for projects added in the current session, otherwise matched by name (`images/3d/<base>.*` for stills, `images/thumbnails/<base>.*` for videos, where `<base>` is the WebP's name). Files still used by another project are never moved; a project added and removed before saving just has its unsaved files discarded. The confirm dialog lists exactly what will move. `trash/` is gitignored (never committed or deployed); the next `publish.sh` commits the removals from `images/`. The ZIP / `projects.json only` exports can't delete files and leave them in place. Empty `trash/` by hand when you no longer need it.
- Caveat: requires a WebP-capable browser for image generation (Chrome/Edge/Firefox — not Safari; the page detects and warns).

### serve.sh / serve.bat — local server

Serve the repo over `http://localhost` so `admin.html` runs in a secure context (enables auto-load and reliable direct-save), then open it in the browser. `serve.sh` (Linux/macOS) opens a terminal window when double-clicked; `serve.bat` (Windows) opens a console window. Closing the window stops the server. Both bind to `127.0.0.1` only, and skip to the next free port if the default (8000) is taken.

### publish.sh — commit + push

Stages everything (`git add -A`), shows the changes, prompts for a commit message, commits, and pushes (setting upstream on the first push). Double-clickable (opens a terminal window that stays open so you can read the result). Run it after **Save to repo folder**.

Publishing stays local and manual — there is no GitHub API or stored-token publishing. Do not add token-based auto-publishing.
