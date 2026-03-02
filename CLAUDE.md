# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Chrome Extension (Manifest V3) that automates AWS Builder ID registration. It uses Gmail alias generation to create unique email variants, AWS OIDC Device Flow for authentication, and content scripts to automate form filling.

## Loading the Extension (No Build Step)

This extension has no build system. Load it directly into Chrome:
1. Open `chrome://extensions/`
2. Enable "Developer mode"
3. Click "Load unpacked" → select the project root directory
4. **Required**: On the extension card → "Details" → enable "Allow in Incognito"

After any code change, click the refresh icon on the extension card in `chrome://extensions/`.

## Architecture

### Communication Model
- **popup.js** → **service-worker.js**: Via `chrome.runtime.sendMessage()` (request/response)
- **service-worker.js** → **popup.js**: Via `chrome.runtime.sendMessage({ type: 'STATE_UPDATE' })` (broadcast)
- **service-worker.js** → **content.js**: Via `chrome.tabs.sendMessage()` (targeted)
- **content.js** → **service-worker.js**: Via `chrome.runtime.sendMessage()` (request/response)

### Session Identification
Sessions are tracked by `windowId`, not `tabId`. AWS registration involves cross-domain navigation which changes `tabId`, but `windowId` remains stable throughout. The service worker updates `session.tabId` when `chrome.tabs.onUpdated` fires.

### Concurrency Control
Two Promise-based locks in service-worker.js:
- `windowCreationLock`: Prevents simultaneous incognito window creation
- `apiCallLock`: Serializes AWS OIDC API calls with a 500ms delay between calls

### Key Files

| File | Responsibility |
|------|----------------|
| `background/service-worker.js` | Session lifecycle, batch registration queue, token polling, message handling |
| `content/content.js` | Page type detection, form auto-fill, Toast UI overlay |
| `lib/oidc-api.js` | `AWSDeviceAuth` class (Device Flow: register client → device authorize → poll token), token validation/refresh |
| `lib/mail-api.js` | `GmailAliasClient` - generates Gmail alias variants (+ alias, dot insertion, case variation) |
| `lib/utils.js` | `generatePassword()`, `generateName()`, `generateUUID()`, name/email prefix generators |
| `popup/popup.js` | UI state rendering, history display, export (JSON/CSV), Kiro IDE sync commands |

### Registration Flow
1. `popup.js` sends `START_BATCH_REGISTRATION` with `loopCount`, `concurrency`, `gmailAddress`
2. Service worker creates sessions and spawns worker coroutines
3. Each session: generate Gmail alias → call OIDC `quickAuth()` (register client + device authorize) → open incognito window with `verificationUriComplete` URL
4. Content script detects page type every 500ms and auto-fills forms
5. Verification code page: waits for manual user input from Gmail inbox
6. Service worker polls `getToken()` until authorization completes (10 min timeout)
7. Token saved to `registrationHistory` in `chrome.storage.local`

### Content Script Page Detection
The content script (`content.js`) uses `detectPageType()` which checks URL patterns and DOM elements to identify: `LOGIN`, `NAME`, `VERIFY`, `PASSWORD`, `DEVICE_CONFIRM`, `ALLOW_ACCESS`, `COMPLETE`. Each page type has a dedicated handler function.

## Module System

- `background/service-worker.js` uses ES modules (`import`/`export`), declared as `"type": "module"` in manifest
- `content/content.js` is wrapped in an IIFE and **cannot use ES module syntax** (content scripts don't support modules in Manifest V3)
- `lib/*.js` files use ES module exports, importable only from the service worker

## Debugging

- Service worker logs: Chrome DevTools → Extensions → "service worker" link on the extension card
- Content script logs: DevTools on the incognito window (F12)
- Popup logs: Right-click extension icon → "Inspect popup"

Key log prefixes: `[Service Worker]`, `[Session <id>]`, `[Worker <id>]`, `[Content Script]`, `[Popup]`
