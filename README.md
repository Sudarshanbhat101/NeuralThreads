<div align="center">

<img src="N/icons/icon128.png" alt="NeuralThreads Logo" width="80" />

# NeuralThreads

**Your memory layer above all AI platforms.**

Transfer conversations — including photos and documents — seamlessly across ChatGPT, Claude, and Gemini using RAG compression. Never lose context when switching AI tools again.

[![Version](https://img.shields.io/badge/version-1.2.0-blueviolet?style=flat-square)](https://github.com/Sudarshanbhat101/NeuralThreads)
[![Manifest V3](https://img.shields.io/badge/Manifest-V3-blue?style=flat-square)](https://developer.chrome.com/docs/extensions/mv3/intro/)
[![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)](#license)
[![Platforms](https://img.shields.io/badge/platforms-ChatGPT%20%7C%20Claude%20%7C%20Gemini-orange?style=flat-square)](#supported-platforms)

</div>

---

## What is NeuralThreads?

You're deep in a complex conversation with Claude — it understands your codebase, your constraints, your preferences. But now you need GPT-4o for its code interpreter, or Gemini for its long-context window. You'd have to start from scratch and re-explain everything.

**NeuralThreads fixes this.**

It's a Chrome extension that scrapes your current conversation, compresses it into a dense, structured context-handoff document using a Gemini-powered RAG pipeline, and injects that document directly into the composer of any other AI platform — complete with your original attachments. The new AI picks up exactly where you left off.

---

## Features

### Cross-Platform Conversation Transfer
Export any conversation from ChatGPT, Claude, or Gemini and inject it into any of the others in seconds. The context handoff document preserves:
- Every concrete detail: numbers, names, decisions, constraints, URLs
- Code blocks, commands, config, and error messages **verbatim**
- What was tried and rejected, not just the final answer
- A "Where this left off" section so the new AI knows what to do first

### RAG Compression Pipeline
Long conversations are intelligently compressed using a full Retrieval-Augmented Generation pipeline — powered by your own Gemini API key:

1. **Chunk** — splits the transcript into overlapping windows (~800 tokens each)
2. **Embed** — batch-embeds all chunks in a single API call (`gemini-embedding-001`)
3. **Retrieve** — ranks chunks by cosine similarity to a semantic query built from the conversation context
4. **Summarize** — generates a thorough handoff document via Gemini Flash

Short conversations (< ~60,000 chars / ~15k tokens) skip chunking entirely and go straight to summary — one API request, no rate limit waste.

### Attachment Capture and Re-injection
NeuralThreads captures the actual bytes of photos and documents from your conversations and re-attaches them when injecting into the target platform. The pipeline:
- Tries a direct worker fetch first (bypasses page CORS via host permissions)
- Falls back to an in-page fetch via the content script for `blob:` URLs and same-origin files
- Uses platform APIs (Claude's conversation API, ChatGPT's backend API) to discover files the DOM doesn't expose
- Downscales oversized images to 2048px max before storing (saves quota and storage)
- Handles up to 40 files, 30 MB per file, 150 MB per conversation

### Privacy First
- Your **Gemini API key** is stored only in `chrome.storage.local`. It is never sent to any backend, never forwarded anywhere — it stays in your browser.
- **Raw conversation messages** never leave the extension to a backend. Only AI-generated summaries sync to the cloud (and only if you opt in to Cloud Sync mode).
- Local save always happens first; cloud sync fires afterward, non-blocking.

### Two Storage Modes

| Mode | Description |
|------|-------------|
| **Local Only** (default) | Everything stays on-device. No account required. |
| **Cloud Sync** | Summaries sync to your backend for access across devices. Requires login. |

### Graceful Degradation
Every failure path produces a usable result:
- **No API key set** — exports a raw transcript excerpt so Export and Inject still work end-to-end
- **Gemini rate-limited or down** — falls back to a raw excerpt with a soft warning
- **Embedding unavailable** — summarizes the first/last chunks instead of failing the export
- **Attachment capture fails** — injects the summary alone and reports which files failed and why

---

## Supported Platforms

| Platform | Export | Inject | Attachment Capture |
|----------|--------|--------|--------------------|
| **ChatGPT** (`chatgpt.com`, `chat.openai.com`) | Yes | Yes | Yes (API-assisted) |
| **Claude** (`claude.ai`) | Yes | Yes | Yes (API-assisted) |
| **Gemini** (`gemini.google.com`) | Yes | Yes | Yes (DOM-based) |

---

## Installation

### From Source (Developer Mode)

1. **Clone the repository**
   ```bash
   git clone https://github.com/Sudarshanbhat101/NeuralThreads.git
   cd NeuralThreads/N
   ```

2. **Open Chrome Extensions**
   Navigate to `chrome://extensions` in your browser.

3. **Enable Developer Mode**
   Toggle the **Developer mode** switch in the top-right corner.

4. **Load the extension**
   Click **Load unpacked** and select the `NeuralThreads/N` folder (the one containing `manifest.json`).

5. **Pin NeuralThreads**
   Click the puzzle icon in the Chrome toolbar and pin NeuralThreads for easy access.

---

## Setup

### 1. Get a Gemini API Key (Free)

NeuralThreads uses the Gemini API for RAG compression. The free tier is sufficient for everyday use.

1. Go to [aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey)
2. Create a new API key
3. Copy the key (it starts with `AIza...`)

### 2. Add Your Key to NeuralThreads

1. Click the NeuralThreads icon in your Chrome toolbar
2. Go to **Settings** → **Gemini API Key**
3. Paste your key and click **Save key**

The extension validates the key immediately. If it's valid, you'll see a confirmation. The key is stored locally — it never leaves your browser.

> **No key?** NeuralThreads still works. Export will produce a raw transcript excerpt instead of an AI-compressed summary. You can always add a key later.

---

## How to Use

### Exporting a Conversation

1. Open a conversation on **ChatGPT**, **Claude**, or **Gemini**
2. Click the **NeuralThreads** extension icon
3. Click **Export This Conversation**
4. NeuralThreads scrapes the page, captures any attachments, compresses the conversation with RAG, and saves it to your local session library

### Injecting into Another Platform

1. After exporting, a **Summary Preview** screen appears
2. Review (and optionally edit) the AI-generated context handoff document
3. Select the **target platform** (ChatGPT, Claude, or Gemini) from the dropdown
4. Open a new chat on the target platform in any browser tab
5. Click **Inject**
6. NeuralThreads attaches the summary (and any captured files) directly to the composer — press Send when ready

### Managing Sessions

- The **Library** tab shows all saved sessions, filterable by platform
- Each session card shows the title, platform, compression stats (e.g. `12,400 tokens → 890 tokens, 93% saved`), and an **Inject** button
- Use **Clear all local sessions** in Settings → Danger Zone to wipe everything

---

## Architecture

```
NeuralThreads/
├── manifest.json              # MV3 manifest — permissions, content scripts, service worker
│
├── background/
│   ├── background.js          # Service worker: message broker, RAG pipeline, cloud sync, auth
│   └── attachments.js         # Attachment download, validation, storage, and API discovery
│
├── content/
│   ├── content.js             # Orchestrator: platform detection, export/inject flows, message router
│   ├── attachment-refs.js     # DOM scanner: collects attachment URLs/filenames from page elements
│   ├── chatgpt.js             # ChatGPT scraper adapter (window.NeuralThreadsScraper)
│   ├── claude.js              # Claude scraper adapter
│   └── gemini.js              # Gemini scraper adapter
│
├── popup/
│   ├── popup.html             # 380x580 popup shell with Library, Settings, Preview panels
│   ├── popup.js               # All popup UI logic: tabs, export trigger, inject flow, auth, sessions
│   └── popup.css              # Dark-mode design system with Inter + JetBrains Mono
│
├── config/
│   └── selectors.json         # Bundled DOM selectors for all 3 platforms (remote-config fallback)
│
├── styles/
│   └── inject.css             # Injected CSS for visual hints on supported pages
│
└── icons/
    └── icon{16,32,48,128}.png # Extension icons
```

### Key Design Decisions

**Scraper isolation** — Each platform adapter (`chatgpt.js`, `claude.js`, `gemini.js`) exposes a single `window.NeuralThreadsScraper` interface. `content.js` calls `.scrape()`, `.getTitle()`, `.getInputSelector()` without knowing which platform it's on. Adding a 4th platform (e.g. Perplexity) is a matter of writing one adapter file and updating the manifest.

**API key never leaves the browser** — The Gemini API key is read directly by the service worker for embedding and generation calls. It is not forwarded to the backend on any code path.

**Local-first** — `saveSession()` writes to `chrome.storage.local` before triggering any cloud sync. If sync fails, the session is never lost.

**Remote config** — `config/selectors.json` can be hot-patched via a hosted URL without publishing an extension update. DOM selectors are the most brittle part of a browser extension; this lets them be patched in minutes.

**Inject via file attachment** — Rather than pasting the summary as raw text (which platforms truncate or mangle), NeuralThreads wraps it in a `.txt` file and attaches it through the platform's own upload path. Three injection routes are tried in order: file `<input>`, synthetic `paste` event, synthetic `drop` event.

**Model fallback chain** — Summarization tries `gemini-flash-latest` → `gemini-2.5-flash` → `gemini-3.1-flash-lite` in order. A model that's retired or overloaded degrades silently to the next instead of failing the export.

---

## Permissions Explained

| Permission | Why It's Needed |
|------------|-----------------|
| `storage`, `unlimitedStorage` | Store sessions, API key, and attachment bytes locally |
| `activeTab` | Read the URL of the currently active tab to detect platform |
| `scripting` | Inject content scripts on demand |
| `alarms` | Periodic remote config refresh (every 6 hours) and JWT token renewal |
| `downloads` | Save the exported conversation as a `.txt` file |
| Host permissions (AI platform domains) | Allow the service worker to fetch attachment files (bypassing page CORS) and make Gemini API calls |

---

## Development

### Prerequisites
- Google Chrome or any Chromium-based browser
- A Gemini API key (free tier works)
- Basic familiarity with Chrome Extension development

### Local Development Workflow

```bash
# Clone the repo
git clone https://github.com/Sudarshanbhat101/NeuralThreads.git
cd NeuralThreads/N

# Load unpacked in Chrome (see Installation above)
# Make changes to any source file
# Go to chrome://extensions → click the reload icon on NeuralThreads
# For content script changes, also refresh the AI platform tab
```

### Adding a New Platform Scraper

1. Create `content/<platform>.js` that sets `window.NeuralThreadsScraper` with these methods:
   - `scrape()` → `Promise<Array<{role, content}>>`
   - `getTitle()` → `string`
   - `getInputSelector()` → `string` (CSS selector for the text composer)
   - `getFileInputSelector()` → `string`
   - `getAttachButtonSelector()` → `string`
   - `isLoggedIn()` → `boolean`
   - `applyRemoteSelectors(sel)` → `void`

2. Add the platform's URL patterns to `manifest.json` under `content_scripts` and `host_permissions`

3. Add selectors to `config/selectors.json`

4. Add the platform URL to `PLATFORM_URLS` in `background/background.js`

### Enabling Remote Config

To hot-patch selectors without an extension update, set these two constants in `background/background.js`:

```js
const REMOTE_CONFIG_ENABLED = true;
const REMOTE_CONFIG_URL = 'https://your.host/selectors.json';
```

Host a valid `selectors.json` (same schema as the bundled one) and the extension will refresh it every 6 hours automatically.

---

## Roadmap

- **v2 — Web App** — Full dashboard at a dedicated URL for managing sessions, full-text search across all conversations, and detailed compression stats
- **v3 — Knowledge Graph** — Tag sessions, build a semantic memory graph, retrieve relevant past context automatically
- **v4 — Agent Routing** — Intelligently suggest which AI platform to route a task to based on conversation history and capabilities
- **Firefox support** — Port to WebExtensions API (MV3 compatible)
- **Perplexity / Grok support** — Additional platform adapters
- **Local embedding model** — Run embeddings in-browser via WebAssembly for zero API calls

---

## Contributing

Contributions are welcome! Before opening a PR:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Test your changes against all three platforms (ChatGPT, Claude, Gemini)
4. Ensure no raw messages are sent to any backend (please keep the privacy model intact)
5. Open a Pull Request with a clear description of what changed and why

For bug reports: include the platform, Chrome version, and the type of conversation that caused the issue.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

<div align="center">

Built by [Sudarshan Bhat](https://github.com/Sudarshanbhat101)

*Stop re-explaining yourself. Start threading your thoughts.*

</div>
