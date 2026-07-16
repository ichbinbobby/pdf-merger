# PDF & Image Merger

[![Nuxt UI](https://img.shields.io/badge/Made%20with-Nuxt%20UI-00DC82?logo=nuxt&labelColor=020420)](https://ui.nuxt.com)

A privacy-first tool to merge PDFs and images into a single PDF file — built with [Nuxt](https://nuxt.com) and [Nuxt UI](https://ui.nuxt.com), hosted at [pdf.ichbinbobby.de](https://pdf.ichbinbobby.de).

## Usage

Everything runs in the browser. No files are ever uploaded to a server.

### Simple merge

1. **Add files** — drag and drop or click to browse. Supports PDF, JPG, PNG, and WebP.
2. **Reorder** — drag items in the list to set the order.
3. **Name** — optionally set the output filename (defaults to `MyMergedFiles`).
4. **Merge** — click the button to combine everything into one PDF and download it instantly.

### Page-level rearranging & splitting

For finer control, click **Rearrange pages** after adding files. This expands all PDFs into individual page thumbnails (rendered on demand using [PDF.js](https://mozilla.github.io/pdf.js/)) and treats each image as a single page. You can then:

- **Drag** pages into any order across files
- **Remove** individual pages by hovering and clicking ×
- **Merge** only the pages you kept

Click **Back to files** to return to the simple view.

### How files are processed

[pdf-lib](https://pdf-lib.js.org) handles all PDF operations client-side:

- **PDFs** — pages are copied directly into the output document.
- **JPEG images** — embedded as-is.
- **PNG / WebP images** — converted to PNG via a canvas element, then embedded.

Each image becomes its own page, sized to the image's natural dimensions.

## Tech stack

- [Nuxt 4](https://nuxt.com) — Vue framework with SSG, deployed to Vercel
- [Nuxt UI 4](https://ui.nuxt.com) — component library built on Tailwind CSS v4
- [pdf-lib](https://pdf-lib.js.org) — client-side PDF creation and manipulation
- [PDF.js](https://mozilla.github.io/pdf.js/) — client-side PDF page rendering for thumbnails

## Setup

```bash
pnpm install
```

## Development

```bash
pnpm dev
```

## Build & Preview

```bash
pnpm build
pnpm preview
```

## Author

[ichbinbobby](https://ichbinbobby.de)
