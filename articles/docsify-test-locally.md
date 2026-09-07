# Test a Docsify Site Locally

How to preview your Docsify documentation site on your local machine before pushing to GitHub Pages. Since Docsify renders everything in the browser from a single `index.html`, any static file server will serve the site — but the official CLI adds live reload, which makes editing much smoother.

> Docsify has no build step. There's nothing to compile; the site is generated client-side at runtime. That's why any static server works, and why "just open index.html" almost works (but fails on browser file:// restrictions for fetching markdown).

## Prerequisites

You need one of the following installed:

- **Node.js** — for `npx` / `docsify-cli` (recommended, includes live reload)
- **Python 3** — built-in on macOS, good for a quick static server

## Option 1: Docsify CLI (Recommended)

The official Docsify CLI includes live reload out of the box, so the page refreshes automatically when you save a file.

```bash
npx docsify-cli serve .
```

This starts a server at `http://localhost:3000` with auto-refresh on file changes.

To install it globally (optional, avoids the `npx` download each time):

```bash
npm install -g docsify-cli
docsify serve .
```

## Option 2: Python HTTP Server

Any static file server works since Docsify is purely client-side. Python 3 ships with one:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000` in your browser. No live reload — refresh manually after edits.

## Option 3: Node http-server

If you prefer a Node-based static server without installing anything globally:

```bash
npx http-server . -p 8000
```

Open `http://localhost:8000`.

## Notes

- Docsify loads everything client-side from `index.html`, so **any static server works**.
- The **Docsify CLI is the only option that provides live reload** on file save.
- If you see a **blank page**, make sure you're serving from the directory that contains `index.html` (the repository root here).
- Opening `index.html` directly via `file://` usually fails — the browser blocks fetching the markdown files. Always serve over HTTP.
- Search indexing and sidebar rendering work the same locally as on GitHub Pages, so what you see locally is what you'll get after deploy.
