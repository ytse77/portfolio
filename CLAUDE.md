# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Deployment

Pushing to `main` triggers GitHub Actions (`.github/workflows/static.yml`), which deploys the entire repo as a static site to GitHub Pages. There is no build step — what's in the repo is what's served.

## Architecture

The entire site is a single file: `index.html`. It contains all CSS (in `<style>`), all HTML structure, and all JavaScript (in `<script>`). There are no external dependencies beyond the Google Fonts CDN.

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
- `fit: "contain"` — for logos/graphics with white backgrounds: card and lightbox render the image letterboxed on a white surface instead of cover-cropped (used by all `design` items)
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

### Tab order and default tab

Tab buttons are defined in HTML around line 555. The button with `class="tab-btn active"` is the default. The initial `renderCards()` call inside the `fetch` callback must match — `threedProjects` or `designProjects`.

### Design tokens

All colors are CSS custom properties on `:root` (lines 12–19): `--bg`, `--surface`, `--border`, `--text`, `--muted`, `--accent`.

## Local admin tool (admin.html)

`admin.html` in the repo root is a self-contained browser GUI for content updates. It is **gitignored on purpose** — local only, never deployed, and must be copied manually to other machines (it won't arrive via `git clone`).

What it does, entirely client-side (no server, no GitHub API, no credentials):

- Loads `projects.json` (via fetch when served over http, or a file picker when opened as `file://`)
- Shows all projects as a visual card grid — **drag cards to reorder**, click a card to open an editor modal for title, category, description, and embed URL (plus move/delete)
- Adding a project: drop an image on the page and it generates the WebP derivatives in-browser via Canvas (800px thumb + 1920px lightbox version for 3D stills; thumb only for videos; design logos pass through untouched with `fit: "contain"`)
- Exports either `projects.json` alone (for reorder/text-only changes) or `portfolio-update.zip` containing the JSON plus all new images already in repo folder structure (`images/thumbs/`, `images/web/`, `images/3d/` …)

Publishing is deliberately manual: extract the ZIP over the repo root (or replace `projects.json`), then commit and push. Do not add token-based auto-publishing — Roberto explicitly wants the download-and-push-manually workflow.

Caveats: requires a WebP-capable browser (Chrome/Edge/Firefox — not Safari; the page detects and warns). Canvas encoding is marginally simpler than the ImageMagick pipeline above; use `magick` when maximum quality matters.
