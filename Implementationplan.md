# Wazuh Log Analyzer — Browser Extension

A Manifest V3 Chrome/Edge extension that injects an AI-assist side panel directly into the analyst's Wazuh Discover tab.

## Design Decisions (from Q&A)

| Decision | Choice | Rationale |
|---|---|---|
| Query capture | URL parsing → DOM fallback | Wazuh doesn't always flush state to URL in real-time |
| UI delivery | Shadow DOM side panel injected into Wazuh page | Keeps log view visible while reading summary |
| Popup role | Settings only (backend URL + API key) + "Launch Panel" trigger | Panel lives in the page, not the tiny popup |
| Auth | API key header (`X-API-Key`), swappable module design for mTLS later | Gets running without cert management overhead |
| Wazuh URL | Analyst-configured in settings | Avoids brittle hardcoded hostname |

## File Structure

```
extension/
├── manifest.json       — MV3 declaration, permissions
├── popup.html          — Settings UI (backend URL, API key, status, Launch Panel)
├── popup.js            — Settings logic + panel injection trigger
├── background.js       — Service worker: relays fetch to backend, streams chunks to content script
├── content.js          — Injected into Wazuh tab: query capture + side panel render + stream display
├── panel.css           — Shadow DOM styles (isolated, no Wazuh CSS conflicts)
└── icons/
    ├── icon16.png
    ├── icon48.png
    └── icon128.png
```

## Architecture Flow

```
Analyst on Wazuh → clicks extension icon → popup.html opens
  └─ popup.js: "Launch Panel" → sends {action:'injectPanel'} to background.js
        └─ background.js: scripting.executeScript(content.js) + insertCSS(panel.css)
              └─ content.js: creates Shadow DOM panel, captures query context
                    └─ shows query preview (time, KQL, index, filters)
                          └─ analyst clicks "Analyze" in panel
                                └─ content.js → {action:'analyze', queryContext} → background.js
                                      └─ background.js: fetch(backendUrl/analyze, {X-API-Key})
                                            └─ streams SSE chunks → chrome.tabs.sendMessage
                                                  └─ content.js: appends chunks to panel
```

## Proposed Changes

### Extension Root

#### [NEW] manifest.json
- `manifest_version: 3`
- Permissions: `activeTab`, `storage`, `scripting`
- No host_permissions (uses `activeTab` + dynamic scripting)
- Background service worker: `background.js`
- Action popup: `popup.html`

#### [NEW] background.js
- Message handlers: `injectPanel`, `analyze`, `checkConnection`
- `injectPanel`: uses `chrome.scripting.executeScript` + `insertCSS` on active tab
- `analyze`: `fetch(backendUrl/analyze)` with `X-API-Key` header, reads SSE stream chunk-by-chunk, relays via `chrome.tabs.sendMessage`
- `checkConnection`: `GET /health` with 5s timeout for popup status indicator
- Auth module: currently API key header — designed as a single `buildHeaders()` function so mTLS client cert can replace it without changing the rest of the code

#### [NEW] popup.html / popup.js
- Dark SOC-themed settings panel (~340px wide)
- Fields: Backend URL, API Key (password input)
- Live status dot: green (connected) / red (unreachable) — probes `/health` on load and on save
- "Launch Analyzer Panel" button → injects content.js into current tab
- Settings persisted to `chrome.storage.sync`

#### [NEW] content.js
**Query Capture (two-pass):**
1. URL parse — extracts `_g` and `_a` RISON params from hash, regex-extracts:
   - `time:(from:...,to:...)` → `timeFrom`, `timeTo`
   - `query:(language:kuery,query:'...')` → `queryString`
   - `index:'...'` → `indexPattern`
2. DOM fallback — reads OpenSearch Dashboards selectors:
   - `[data-test-subj="superDatePickerShowDatesButton"]` → time range text
   - `[data-test-subj="queryInput"]` → KQL query value
   - `[data-test-subj*="globalFilterItem"]` → active filter pills
   - `[data-test-subj="discover-dataView-switch-link"]` → index pattern

**Side Panel (Shadow DOM):**
- Isolated from Wazuh styles via `host.attachShadow({mode:'open'})`
- Sections: Query Context card → Analyze button → AI Summary box → disclaimer footer
- Query preview shows detected time range, KQL query, index, active filters
- "Refresh Context" button re-runs capture and updates preview
- Streaming: appends token chunks to summary box as they arrive
- Status states: idle → loading (spinner) → streaming → done / error

#### [NEW] panel.css
- Injected into Shadow DOM — fully isolated
- Dark theme: `#0a0e1a` bg, `#00c8ff` cyan accent, `#1e2a3a` card bg
- Monospace font for AI output, sans-serif for UI chrome
- Smooth slide-in animation from right edge
- Responsive to panel being opened/closed

## Verification Plan

### Manual Verification
1. Load unpacked extension in Chrome (`chrome://extensions` → Developer mode → Load unpacked → select `extension/`)
2. Open any HTTPS page, click extension icon — popup should open with settings fields
3. Configure a test backend URL, click "Launch Panel" — side panel should slide in
4. Navigate to a Wazuh Discover tab with active filters — "Refresh Context" should populate the query preview card
5. With a mock backend running (`/health` → 200, `/analyze` → SSE stream), click "Analyze" — summary should stream in
6. Close and reopen panel — settings should persist

### Backend Mock (for testing without the full pipeline)
A simple Python one-liner SSE server can be used to validate the streaming path:
```python
# python -m http.server 8080 (then curl simulate SSE)
```
