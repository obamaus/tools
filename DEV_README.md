# ObamaTools / Premium Session Injector - Developer Documentation

## 1. Project Overview
This project is a Chrome Extension and Web Dashboard designed to inject premium session cookies (Netflix, Prime Video, Coursera, etc.) into the user's browser, allowing free access to these services.

**Core Workflow:**
1.  User visits the **Dashboard** (`https://obamaus.github.io/tools/` or local `index.html`).
2.  User clicks "Watch Now" on a tool (e.g., Netflix).
3.  Dashboard communicates with the **Chrome Extension** via `window.postMessage`.
4.  Extension fetches session data (cookies) from a remote server (`session.khanlegacyagency.com`).
5.  Extension sets these cookies in the browser.
6.  Extension navigates the current tab to the service's homepage (e.g., `netflix.com`).

## 2. The Core Challenge: LiteSpeed Security
The session server (`session.khanlegacyagency.com`) is protected by **LiteSpeed Anti-DDoS / Browser Verification**.

**Symptoms:**
- Visiting the URL shows a "One moment, please..." spinner.
- The browser reloads 1-3 times (taking 5-10 seconds) to verify the client is a real browser (checking JS engine, window dimensions, user agent).
- Only *after* verification does the server serve the HTML containing the JSON payload with cookies.

**Previous Failed Strategies:**
- **FETCH API**: Blocked (returns 403 or challenge HTML) because it can't execute the challenge JavaScript.
- **Hidden/Inactive Tabs**: Browser throttles CPU for background tabs, causing the challenge to time out or fail.
- **Minimized Windows**: Same throttling issue; detection scripts identify the window state.
- **Offscreen API**: Google Chrome's `offscreen` API has limited DOM capabilities and is often flagged by anti-bot scripts.

**Current Strategy: "Micro-Window" (Ghost Window)**
- Open a **Normal** `chrome.windows.create({ type: 'normal' })` window.
- Size it to **1x1 pixel**.
- Position it off-screen or far bottom-right (`left: 9999, top: 9999`).
- **Theory**: The OS sees a "visible", active window and gives it full priority. The challenge runs, cookies are extracted, and the window closes.

## 3. Project Structure

### `manifest.json`
- Permissions: `cookies`, `tabs`, `storage`, `scripting`, `declarativeNetRequest`.
- Host Permissions: `*://*.khanlegacyagency.com/*`, `*://*.netflix.com/*`, etc.

### `background.js` ( The Orchestrator)
- **Listeners**: Listens for messages from Dashboard and Content Scripts.
- **`handleSyncRequest(sessionId, url)`**: The main entry point.
    - Creates the "Ghost Window" to load the session URL.
- **`processSessionData(html)`**: Receives raw HTML from the ghost window, parses cookies, and injects them using `chrome.cookies.set`.
- **`cleanupSync(tabId)`**: Closes the ghost window after success.

### `content.js` (The Sniffer)
- Runs on `khanlegacyagency.com` pages.
- **`isChallenge()`**: Detects if the page is currently showing the "One moment" spinner.
- **`performSniff()`**:
    - Waits for the challenge to pass.
    - Scans the DOM for specific elements (`#extv`, `#ext01JSON`) containing the cookie JSON.
    - Sends found data to `background.js`.
- **Note**: This script *must not* interrupt the LiteSpeed reload cycle. It should only act when the data is finally present.

### `index.html` (The Dashboard)
- **UI**: Lists available tools.
- **`startSyncTimer()`**: Visual countdown.
- **Communication**: Sends `INJECT_COOKIES` event to the extension.

## 4. Debugging & Maintenance
If the sync fails (Connection Refused), it usually means:
1.  The Ghost Window was throttled or blocked.
2.  The Content Script started sniffing too early or crashed.
3.  The LiteSpeed challenge changed its detection logic.

**To Debug:**
- Check `background.js` console (Extensions -> Service Worker).
- Check `content.js` logs (inspect the Ghost Window if possible).
