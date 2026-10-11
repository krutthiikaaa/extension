# Gemma 4 Browser Extension — Production & Deployment Guide

This document provides complete instructions for building, locally verifying, packaging, and submitting the **Transformers.js Gemma 4 Browser Assistant** to the Google Chrome Web Store.

---

## 1. Overview & Architecture

The Gemma 4 Browser Assistant is an on-device Chrome extension powered by **Transformers.js (v4.2.0)** and **WebGPU**. It operates without cloud LLM API dependencies:
- **Background Service Worker (`background.js`)**: Runs the central AI inference engine, loads ONNX models, and executes browser automation tools.
- **Side Panel (`sidebar.html` / React 19)**: Persistent user chat interface that displays streaming responses, token generation speed metrics, and tool execution status.
- **Content Scripts (`content.js`)**: Injected into active webpages to parse structured DOM content (headings, paragraphs, lists) and highlight elements for RAG interactions.
- **Vector Storage (`idb` IndexedDB)**: Maintains vector embeddings on-device for browsing history and page retrieval.

---

## 2. Prerequisites & System Requirements

Before running or deploying the extension, ensure the target environment meets these minimum specifications:

| Requirement | Specification | Notes |
| :--- | :--- | :--- |
| **Browser** | Google Chrome 113+ | Required for native WebGPU support. (Tested on Chrome 154) |
| **GPU / Acceleration** | WebGPU-compatible GPU | Apple Silicon (M1/M2/M3/M4/M5), NVIDIA RTX, or modern AMD/Intel GPU |
| **GPU / Unified VRAM**| 4 GB+ available VRAM | Required to load the quantized Gemma 4 weights into GPU memory |
| **System Memory** | 8 GB RAM (16 GB+ recommended) | Prevents browser tab eviction during initial model compilation |
| **Disk Space** | ~4 GB free disk space | Required for browser cache storage of downloaded ONNX models |
| **Build Tools** | Node.js 20+ and pnpm 10+ | Used to compile TypeScript and bundle assets with Vite |

---

## 3. Production Build Instructions

To build the extension from source:

1. **Install dependencies**:
   ```bash
   pnpm install --frozen-lockfile
   ```

2. **Run TypeScript check and production build**:
   ```bash
   pnpm run build
   ```
   *This executes `tsc && vite build`, creating the optimized distribution bundle in `dist/`.*

3. **Verify the build output**:
   Confirm that the `dist/` directory contains:
   - `manifest.json`: Valid Manifest V3 configuration
   - `background.js`: Background service worker entrypoint
   - `content.js`: Self-contained inlined content script
   - `sidebar.html`: Root side panel HTML document
   - `assets/`: Contains React bundles, CSS stylesheets, and `ort-wasm-simd-threaded.asyncify-*.wasm`
   - `icons/`: Extension icons in 16x16, 32x32, 48x48, and 128x128 resolutions

---

## 4. Local Installation & Verification in Chrome

To test the production build locally before distribution:

1. Open **Google Chrome**.
2. Navigate to:
   ```text
   chrome://extensions/
   ```
3. In the top-right corner, switch the **Developer mode** toggle to **ON**.
4. In the top-left corner, click **Load unpacked**.
5. In the file picker, select the generated `dist` directory:
   ```text
   /Users/krutthiikaaa/Desktop/gemma 4/gemma4-browser-extension/dist
   ```
6. The extension **Transformers.js Gemma 4 Browser Assistant** (version 0.2.1) will appear in your extension list with zero errors.
7. Click the **Extensions menu** (puzzle piece icon) on your Chrome toolbar and **Pin** the Gemma 4 assistant.

---

## 5. First-Run Setup & Model Download

When you launch the extension for the first time:

1. Click the pinned extension icon to open the Chrome **Side Panel**.
2. The extension automatically inspects the local browser cache for the two required models:
   - **Text Generation**: `onnx-community/gemma-4-E2B-it-ONNX` (quantized `q4f16`, ~2.99 GB)
   - **Feature Extraction**: `onnx-community/all-MiniLM-L6-v2-ONNX` (`fp32`, ~86.65 MB)
3. If not cached, the panel presents a welcome screen calculating the total download size (**~3.07 GB**).
4. Click **Download Models**.
5. Real-time progress bars will display download status for both models.
6. Once complete, weights are cached in browser storage (Cache API / Origin Private File System). Subsequent sessions launch immediately without re-downloading.

---

## 6. Testing Core Functionality

Once initialized, test the following built-in capabilities in the chat interface:

### A. Tab Management
- **List Tabs**: Prompt `"What tabs do I have open?"` → Agent executes `get_open_tabs` and summarizes active titles and URLs.
- **Switch Tab**: Prompt `"Switch to my GitHub tab"` → Agent executes `go_to_tab`.
- **Open URL**: Prompt `"Open wikipedia.org in a new tab"` → Agent executes `open_url`.
- **Close Tab**: Prompt `"Close tab 123"` → Agent executes `close_tab`.

### B. In-Page Retrieval Augmented Generation (RAG)
- Navigate to an article (e.g. any Wikipedia or documentation page).
- Prompt `"What is this page about?"` or `"Summarize the main points on this webpage"`.
- Agent calls `ask_website`, which retrieves sentences from `content.js`, generates embeddings via `all-MiniLM`, and scores them with cosine similarity.
- Prompt `"Highlight where it talks about [topic]"` → Agent calls `highlight_website_element` to scroll to and highlight the element in yellow.

### C. Browsing History Semantic Search
- Prompt `"Find pages I visited earlier about machine learning"`.
- Agent calls `find_history` to run vector similarity searches over your local `VectorHistoryDB` IndexedDB store.

---

## 7. Troubleshooting

| Issue | Cause | Resolution |
| :--- | :--- | :--- |
| **`WebGPU is not supported`** | Hardware acceleration disabled or outdated browser | Open `chrome://settings/system` and enable **Use graphics acceleration when available**. Ensure Chrome is version 113+. Check `chrome://gpu` to confirm WebGPU status is "Hardware accelerated". |
| **Model download stalls** | Network interruption during multi-gigabyte download | Click extension icon to refresh the side panel and click "Download Models" again. Transformers.js resumes cached chunks. |
| **Out of Memory / Tab crash** | Insufficient GPU memory for KV cache | Close unnecessary browser tabs and GPU-intensive applications. Ensure at least 4 GB free VRAM/unified memory. |
| **Content script fails on page** | User is on a restricted Chrome URL (`chrome://*`, `chrome-extension://*`, or Chrome Web Store) | Chrome prevents content script injection on internal pages by security design. Test on standard `http://` or `https://` websites. |

---

## 8. Distribution Package (`gemma4-browser-extension.zip`)

For Chrome Web Store distribution, a clean ZIP package has been created:
- **Location**: `/Users/krutthiikaaa/Desktop/gemma 4/gemma4-browser-extension/gemma4-browser-extension.zip`
- **Compressed Size**: **5.8 MB**
- **SHA-256 Checksum**: `a00a24c3756ed892733e9be76c30e7cfcca50d66af2bfd9f5a9fbcfe9b145d3f`
- **Package Structure**:
  ```text
  gemma4-browser-extension.zip
  ├── manifest.json
  ├── background.js
  ├── content.js
  ├── sidebar.html
  ├── icons/
  │   ├── icon-16.png
  │   ├── icon-32.png
  │   ├── icon-48.png
  │   └── icon-128.png
  └── assets/
      ├── ort-wasm-simd-threaded.asyncify-*.wasm
      ├── sidebar-*.js
      ├── sidebar-*.css
      ├── tools-*.js
      └── types-*.js
  ```
  *(Note: Model weights are downloaded on demand by Transformers.js and are intentionally not bundled into the ZIP to comply with Chrome Web Store file size limits.)*

---

## 9. Chrome Web Store Submission Guide

Follow these steps to submit the package to the Chrome Web Store:

### Step 1: Access the Chrome Web Store Developer Dashboard
1. Go to the [Chrome Web Store Developer Dashboard](https://chrome.google.com/webstore/devconsole/).
2. Sign in with your Google account.
3. If this is your first submission, register as a Chrome Web Store Developer (requires a one-time $5 USD registration fee).

### Step 2: Upload Package
1. Click **+ New Item**.
2. Drag and drop `gemma4-browser-extension.zip` or select it from your file system.
3. The dashboard will validate `manifest.json`.

### Step 3: Store Listing Details
Prepare and enter the required store listing information:
- **Title**: Transformers.js Gemma 4 Browser Assistant
- **Summary** (max 132 chars): On-device AI browser assistant powered by Transformers.js, Gemma 4, and WebGPU.
- **Detailed Description**: Explain how the extension works, that models run locally on-device via WebGPU, that initial setup downloads ~3 GB of open-source ONNX weights, and list the available tab management, webpage Q&A, and semantic history search tools.
- **Category**: Productivity / Search Tools
- **Graphic Assets**:
  - Small icon: 128x128 PNG (provided in `public/icons/icon-128.png`)
  - Screenshots: At least one 1280x800 or 640x400 PNG showing the side panel in action
  - Promotional Tile: 440x280 PNG (optional but recommended)

### Step 4: Privacy Practices & Permissions Justification
Chrome Web Store reviewers require specific justifications for each requested permission:

| Permission | Justification for Google Reviewers |
| :--- | :--- |
| `sidePanel` | Displays the persistent AI chat assistant interface alongside web pages. |
| `tabs` | Allows the AI assistant to inspect open tabs, switch tabs, open requested URLs, and close tabs upon user instruction. |
| `scripting` | Extracts page title and `meta[name="description"]` from active tabs to index them into the local semantic vector history. |
| `storage` | Stores user configuration (enabled tools and theme settings) locally in `chrome.storage.local`. |
| `host_permissions` (`http://*/*`, `https://*/*`) | Required for the content script (`content.js`) to extract structured text elements from active webpages when the user asks questions about the current page, and to highlight relevant sections. |

### Step 5: Data Usage Disclosures
- **Single Purpose**: On-device AI browser assistance and local webpage content interaction.
- **Data Collection**: Select **No, I do not sell user data**, and **No, I do not transfer user data to third parties**.
- **Network Disclosure**: Disclose that the extension contacts `huggingface.co` strictly on first launch to download public open-source model weights (`onnx-community/gemma-4-E2B-it-ONNX` and `onnx-community/all-MiniLM-L6-v2-ONNX`). No user prompts, browsing history, or webpage data leave the device.

### Step 6: Submit for Review
1. Review all inputs and click **Submit for Review**.
2. Google reviews Manifest V3 extensions typically within 1 to 3 business days.
3. Once approved, the extension status updates to **Published** on the Chrome Web Store.
