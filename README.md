# MAGE-AI-Exporter
Your threads belong to you
# MAGE AI Exporter

> One-click export of AI conversations from Claude, ChatGPT, Gemini, and Grok.
> HTML · JSON · TXT · MD — fully local, no server.

HTML export approach for ChatGPT inspired by [ChatGPT Export](https://github.com/AliYmn/chatgpt-export) by Enes Saltik (phlmox).

## Supported Platforms

| Platform | URLs | Adapter |
|----------|------|---------|
| Claude | claude.ai | `claude.js` |
| ChatGPT | chatgpt.com, chat.openai.com | `chatgpt.js` |
| Gemini | gemini.google.com | `gemini.js` |
| Grok | grok.com, x.com/i/grok | `grok.js` |

## How It Works

```
User clicks toolbar icon
        │
        ▼
   ┌─────────┐        ┌──────────────────┐
   │ popup.js │───────▶│  Content Script   │
   │          │ scrape │  (per-platform    │
   │ Detects  │◀───────│   adapter)        │
   │ platform,│ data   │                   │
   │ format   │        │  base.js provides │
   │          │───────▶│  shared utilities  │
   │          │cloneDOM│                   │
   │          │◀───────│                   │
   └────┬─────┘ html   └────────┬──────────┘
        │                       │
        │                       ▼
        │               ┌──────────────┐
        │               │ background.js│
        │               │ (service     │
        │               │  worker)     │
        │               │              │
        │               │ CORS-free    │
        │               │ image fetch  │
        │               └──────────────┘
        │
        ▼
  ┌───────────┐
  │ Download  │
  │ (saveAs)  │
  └───────────┘
```

1. **Popup opens** — `popup.js` detects the platform from the active tab URL
2. **User selects format** (HTML, JSON, or TXT) and clicks Export
3. **For JSON/TXT**: popup sends `scrape` message → adapter walks the DOM, extracts structured content blocks → returns normalized conversation object → popup converts to selected format
4. **For HTML**: popup sends `cloneDOM` message → adapter clones the DOM, strips UI chrome, embeds images as base64 → returns self-contained HTML string
5. **Download** triggers via `chrome.downloads` with `saveAs: true` (native OS save dialog)

All processing is local. Nothing leaves the browser.

## Architecture

```
mage-ai-exporter/
├── manifest.json                 # MV3 Chrome extension manifest
├── src/
│   ├── background.js             # Service worker — CORS-free image fetching
│   ├── adapters/
│   │   ├── base.js               # Shared: image embedding, LaTeX extraction,
│   │   │                         #   content block classification, model detection
│   │   ├── claude.js             # Claude.ai adapter (705 lines)
│   │   ├── chatgpt.js            # ChatGPT adapter (877 lines)
│   │   ├── gemini.js             # Gemini adapter (585 lines)
│   │   └── grok.js               # Grok adapter (1080 lines) — handles both
│   │                             #   grok.com and x.com/i/grok
│   ├── lib/
│   │   ├── purify.min.js         # DOMPurify (ChatGPT adapter only)
│   │   └── highlight.min.js      # highlight.js (ChatGPT adapter only)
│   ├── popup/
│   │   ├── popup.html            # Extension popup UI
│   │   └── popup.js              # Platform detection, format conversion, download
│   └── icons/
│       ├── icon16.png
│       ├── icon48.png
│       └── icon128.png
├── tests/
│   ├── debug.html                # Browser-based DOM inspection tool
│   ├── test_adapters.js          # jsdom-based adapter validation
│   └── test_pipeline.js          # End-to-end pipeline test
└── docs/
    ├── STORE_LISTING.md          # Chrome Web Store listing copy + privacy policy
    ├── promo-small.png
    └── promo-marquee.png
```

## Adapter Contract

Every adapter implements three message handlers:

### `scrape` → Structured data (for JSON/TXT)

Returns a normalized conversation object:

```javascript
{
  title: "Conversation Title",
  source: {
    platform: "claude",        // claude | chatgpt | gemini | grok
    platform_url: "https://...",
    model: "Claude 3.5 Sonnet", // best-effort, may be null
    model_detected: true,
    exported_at: "2026-04-12T...",
    exporter: "mage-ai-exporter",
    exporter_version: "1.0.0"
  },
  turns: [
    {
      role: "user",           // user | assistant
      content: [              // array of typed content blocks
        { type: "text", value: "...", html: "..." },
        { type: "code", language: "python", value: "..." },
        { type: "latex", raw: "E=mc^2", display: "block", html: "..." },
        { type: "image", alt: "...", source_url: "...", data_url: null },
        { type: "table", headers: [...], rows: [[...], ...] },
        { type: "embed", subtype: "artifact", title: "...", value: "..." }
      ]
    }
  ]
}
```

### `cloneDOM` → HTML string (for HTML export)

Returns a complete, self-contained HTML document with:
- MAGE dark theme (IBM Plex Sans/Mono, CSS custom properties)
- Images embedded as base64 data URLs
- KaTeX CSS loaded via CDN `@import`
- Code blocks with syntax highlighting (ChatGPT uses highlight.js; others preserve platform highlighting)
- Print stylesheet (`@media print`) with dark-on-light colors
- Visualization placeholders (Claude) with styled cards

The HTML path uses a two-phase approach: sync `exportDOM()` builds the document, async `exportDOMAsync()` adds image embedding. If the async path fails, it falls back to sync.

### Content Block Types

| Type | Fields | Notes |
|------|--------|-------|
| `text` | `value`, `html?` | Plain or rich text. `html` preserves original formatting. |
| `code` | `language`, `value` | Raw source code. Language detected from CSS classes or header labels. |
| `latex` | `raw`, `display`, `html?` | Raw LaTeX source. `display`: `"block"` or `"inline"`. |
| `image` | `alt?`, `source_url?`, `data_url?`, `filename?` | URL captured; base64 embedded in HTML export. |
| `table` | `headers`, `rows` | Structured tabular data. KaTeX in cells cleaned to raw notation. |
| `embed` | `subtype`, `title?`, `value`, `html?`, `children?` | Artifacts, canvas panels, visualizations. |
| `attachment` | `label`, `subtype?`, `preview?` | Claude paste.txt, file cards (partial — see limitations). |

## Image Embedding Pipeline

HTML exports embed images as base64 data URLs so the exported file is fully self-contained. The pipeline uses a three-tier strategy with 15-second timeout per image:

**Tier 1 — Canvas**: Find the original `<img>` on the live page (cloned images lose `.complete` status), draw it to a `<canvas>`, call `toDataURL()`. Works for same-origin and CORS-enabled images.

**Tier 2 — Content script fetch**: `fetch()` with `credentials: 'include'` from the content script. Works for authenticated CDN URLs where the page has session cookies (e.g., Claude's image hosting).

**Tier 3 — Background script fetch**: Message `background.js` to fetch the URL. The service worker has broader network access than content scripts, bypassing some CORS restrictions.

Each tier falls through to the next on failure. Small images (<50px) are skipped by default.

### Platform-specific image handling

- **ChatGPT**: DALL-E images use `position: absolute` inside complex wrappers. The adapter strips positioning, deduplicates (ChatGPT renders 3 copies at different sizes), and walks up 4 parent levels to fix wrapper CSS.
- **Gemini**: Generated images are wrapped in `<button>` elements. The adapter unwraps images from buttons before button removal. Canvas strategy draws from the original page element, not the clone.
- **Grok (grok.com)**: Images deduplicated in both DOM clone and scrape paths (Grok renders same image in bubble + overlay).
- **Grok (x.com)**: Extract-and-rebuild strategy. Twitter's CSS positioning is unfixable in export, so the adapter extracts the real URL (resolving `blob:` → `background-image` when needed), removes the wrapper chain, inserts a clean `<img>` at the original position via placeholder markers. Emoji images get inline sizing (`height: 1.2em`).

## Platform-Specific Notes

### Claude (`claude.js`)

- **Turn detection**: `[data-testid="user-message"]` for user; `[class*="font-claude-response"]` div containers for assistant.
- **Visualizations**: Detected via `iframe[src*="claudemcpcontent"]` only. A text-matching heuristic ("Connecting to visualize...") was removed due to false positives with file download cards. The async export path attempts to fetch SVG source from visualization iframe URLs.
- **SVG filtering**: Attribute `width`/`height` takes priority over `viewBox` for size classification. SVGs inside `.katex` or `pre` are always preserved.
- **Artifacts**: `[class*="artifact-block-cell"]` extracted as `embed` blocks with `subtype: "artifact"`.

### ChatGPT (`chatgpt.js`)

- **HTML export**: Uses DOMPurify with a strict tag allowlist (the Saltik approach). LaTeX is extracted before purification and re-injected after, since DOMPurify strips KaTeX's nested spans. Code blocks are extracted, DOMPurify runs, then code is re-highlighted with highlight.js.
- **Turn detection**: `[data-testid^="conversation-turn-"]` on live DOM; `.message.user-message` / `.message.assistant-message` on exported HTML.
- **Canvas panels**: ProseMirror editors detected and parsed as `embed` blocks with `subtype: "canvas"` and parsed `children`.
- **Download links**: `<button class="behavior-btn">` inside `<span data-state="closed">` — the span is preserved and the button converted to `<u>text</u>`.

### Gemini (`gemini.js`)

- **Turn detection**: Angular custom elements `<user-query>` and `<model-response>`. Fallback to `.conversation-container` divs.
- **Angular cleanup**: TTS containers (`.response-tts-container`), avatar gutters, visually-hidden headers, response container headers all removed. Inline `height` styles stripped from non-KaTeX elements.
- **KaTeX**: Gemini renders KaTeX delimiter SVGs as `<img>` elements with black strokes. The export CSS applies `filter: invert(1) brightness(0.85)` for dark background visibility, with `filter: none` in print.
- **Math**: Supports both standard KaTeX annotations and Gemini's `data-math` attribute on `.math-block` / `.math-inline` wrappers.

### Grok (`grok.js`)

The most complex adapter — handles two completely different DOMs.

**grok.com**: Standard DOM with `.message-bubble` containers. Role detection uses a multi-signal approach: (1) parent/ancestor class signals (`justify-end`, `ml-auto` = user), (2) Grok avatar/logo SVG presence = assistant, (3) content heuristic scoring (code blocks, tables, length → assistant), (4) alternation as fallback.

**x.com/i/grok**: Twitter's React app with atomic CSS (`r-*` classes). No semantic HTML — headings are styled `<span>` elements, paragraphs are `<div>` with `r-1g7jtus` classes. Turn detection uses `r-1kt6imw` (definitive user text marker) and `r-16lk18l` (assistant wrapper), with orphan detection for the last assistant response which sometimes lacks `r-16lk18l`. Text extraction falls back through semantic HTML → x.com paragraph spans → full container text.

**x.com CSS class mapping** (current as of April 2026 — these rotate with X deployments):

| Class | Meaning |
|-------|---------|
| `r-obd0qt` | Message frame |
| `r-1kt6imw` | User text container (definitive) |
| `r-16lk18l` | Assistant response wrapper |
| `r-3pj75a` | Content wrapper (orphan detection) |
| `r-rjixqe` | Assistant text styling |
| `r-1blvdjr` | h2-like heading |
| `r-adyw6z` | h3-like heading |
| `r-b88u0q` | Bold text |
| `r-36ujnk` | Italic text |
| `r-xoduu5` | KaTeX wrapper |
| `r-13awgt0` | Display math marker |
| `r-1g7jtus` | Paragraph-like text span |
| `r-p1pxzi` | Closing paragraph span |

**x.com KaTeX**: Uses MathML-only KaTeX (no `katex-html`). Hidden `code.raw_katex` / `code.raw_katex_block` elements contain duplicate LaTeX source and are stripped. The `r-xoduu5` wrapper with `r-13awgt0` = display math, without = inline.

## Export Formats

### HTML (.html)

Self-contained dark-themed document. IBM Plex Sans body, IBM Plex Mono code. User messages right-aligned with subtle background; assistant messages left-aligned. Code blocks with `#111` background. KaTeX renders via CDN stylesheet. Images embedded as base64. Print-friendly with `@media print` light theme.

### JSON (.json)

```json
{
  "mage_version": "1.0",
  "source": { ... },
  "conversation": {
    "title": "...",
    "turns": [ ... ]
  }
}
```

Machine-readable structured data. `html` fields stripped from blocks (raw data only). Image `data_url` set to `null` (base64 is HTML-only). Suitable for programmatic consumption, pipeline integration, and future MAGE import.

### TXT (.txt)

Flat text with `--- You ---` / `--- Assistant ---` role labels. Code wrapped in `[code:lang]...[/code]` markers. LaTeX as `$...$` / `$$...$$` notation. Tables as ASCII art. Images as `[image: alt | url]` placeholders. Human-readable archive format.

### MD (.md)

Markdown with `### **You:**` / `### **Assistant:**` role headers. Fenced code blocks with language tags. LaTeX as `$...$` / `$$...$$` notation. Tables as pipe tables. Images as `![alt](url)`. Embeds and attachments as blockquotes. Compatible with Obsidian, Karpathy-style LLM wikis, static site generators, and any MD-native workflow. Generated from the JSON intermediate representation, not the DOM.

## Known Limitations

### By design
- **Four formats only**: HTML, JSON, TXT, MD. PDF and `.binder` were evaluated and removed as derivative — HTML can be printed to PDF; `.binder` is a future MAGE ecosystem format.
- **No image base64 in JSON/TXT**: Image URLs are captured but base64 embedding only runs for HTML exports.
- **Grok model detection**: Not exposed in DOM on either grok.com or x.com. `model` field will be `null`.
- **Grok conversation title**: x.com does not expose the conversation title in the DOM. Falls back to first 80 characters of the first user message.

### ChatGPT
- **Canvas content**: ChatGPT canvas panels render in a floating React portal outside the conversation DOM. Canvas turns are captured as placeholder cards (`[Canvas content — open in ChatGPT to view]`). Full canvas extraction requires click-through automation and is planned for v1.1.
- **DALL-E images on long virtualized threads**: Chrome DOM virtualization prevents image harvest on long threads. Images show as `🖼️ Generated Image` placeholder cards. Short threads embed DALL-E images directly.
- **Web search images on long virtualized threads**: Show as placeholder cards. Auto-scroll approach was tested and failed (ChatGPT re-virtualizes during scroll).
- **filecite tokens**: `□filecite□turn8file0□...□` tokens appear as raw text when citations reference uploaded documents. Planned for v1.2.

### Claude
- **Extended thinking body when collapsed**: Only the summary line is available in the DOM when thinking is collapsed. Expand thinking blocks before exporting to capture full body text.
- **paste.txt attachment cards**: Only the 300-character preview is shown in the DOM. Full content requires Claude API access (v1.1).
- **SVG visualizations**: Rendered via `claudemcpcontent.com` iframes (cross-origin). No placeholder is shown — visual gap where the visualization appeared. Planned for v1.1.

### Gemini
- **Image CORS**: Canvas strategy works for generated images; `fetch` strategies may fail for `lh3.googleusercontent.com` URLs. If images appear missing, the background script fetch may have been blocked.

### Grok (x.com/i/grok)
- **`r-*` CSS class rotation**: Twitter rotates atomic class names with deployments. Exports may degrade when this happens. Selector update procedure: run the debug.html snippet on x.com, identify new class names, update grok.js selectors.
- **Code blocks in light mode**: x.com exports retain the dark code block background (`#111`) in light mode. This preserves visual fidelity of the original platform rendering; consistent light-mode code blocks across all adapters are planned for v1.1.

### All platforms
- **User-uploaded file cards (non-Claude)**: Not captured on ChatGPT, Gemini, or Grok. Each platform renders these differently. Minimum viable signpost capture is planned for v1.2.
- **Edit/regeneration branches**: Exporter captures the visible (final) response only. Regenerated alternatives are not exported.
- **No Firefox support**: Extension is Chrome/Chromium only (MV3). Firefox port planned post-submission.

## Development

### Load as unpacked extension

1. Go to `chrome://extensions/` → Enable Developer mode
2. Click "Load unpacked" → select the `mage-ai-exporter/` directory
3. Navigate to any supported platform → click the MAGE hexagon in toolbar

### Testing

`tests/debug.html` provides a browser-based DOM inspection tool with bookmarklet and console snippet. Run on live platform pages to verify adapter selectors match current DOM.

`tests/test_adapters.js` and `tests/test_pipeline.js` are jsdom-based validation scripts for offline testing against exported HTML files.

### Adding a new platform

1. Create `src/adapters/newplatform.js` following the existing adapter pattern (message listener for `scrape` + `cloneDOM`, turn discovery, content parsing)
2. Add URL match pattern to `manifest.json` `content_scripts`
3. Add hostname → platform mapping in `popup.js` `PLATFORMS` object
4. Add `host_permissions` in `manifest.json` for any CDN domains the platform uses for images

### Key design decisions

- **Per-adapter HTML shells**: Each adapter has its own `buildHTMLShell()` with platform-specific CSS overrides (x.com atomic class styling, Gemini KaTeX SVG inversion, etc.). A shared template was considered but rejected — the per-platform CSS differences are significant enough that a shared base would need so many overrides it wouldn't actually reduce maintenance.
- **DOMPurify only for ChatGPT**: ChatGPT's DOM is the most deeply nested with the most wrapper elements. The other platforms have cleaner DOM structures where targeted `querySelectorAll` cleanup is simpler and more predictable than allowlist-based sanitization.
- **WeakSet for capture tracking**: Prevents double-extraction when multiple selectors match nested elements (e.g., a `<pre>` inside a `<div>` that matches both code block and text selectors). Reset per `scrape()` call; Grok additionally resets per assistant turn due to its alternation-based parsing.
- **Document-order sorting**: All adapters collect blocks with their source DOM elements, then sort by `compareDocumentPosition` before output. This ensures content order matches the visual page regardless of which selectors fired in which order.

## License

See LICENSE file.

---

*Built as part of the MAGE ecosystem. BINDER app coming soon.*
