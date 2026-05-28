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
  "title":    "Project Name",
  "category": "Category Label",
  "type":     "image",
  "src":      "images/my-project.jpg",
  "embedUrl": "",
  "gradient": "135deg, #1a1e2a, #1a2640"
}
```

- `type: "image"` — set `src` to a file path (e.g. `images/foo.jpg`) or URL; `null` shows a gradient placeholder
- `type: "video"` — set `embedUrl` to a framerate.tv embed URL; leave `src` empty
- `gradient` — used as card background and lightbox fallback when no media is set

The demo reel embed URL is set in `REEL_URL` near the top of the `<script>` block.

### Tab order and default tab

Tab buttons are defined in HTML around line 555. The button with `class="tab-btn active"` is the default. The initial `renderCards()` call inside the `fetch` callback must match — `threedProjects` or `designProjects`.

### Design tokens

All colors are CSS custom properties on `:root` (lines 12–19): `--bg`, `--surface`, `--border`, `--text`, `--muted`, `--accent`.
