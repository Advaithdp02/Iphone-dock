# Mac Control — Product Spec (v1.0, condensed)

Local-first system: iPhone remote-triggers custom buttons that run actions on a Mac. No cloud, no internet dependency, LAN only.

## Core Principle
**The Mac is the brain; the iPhone is the remote.** The iPhone never holds shell commands — it sends an `actionId`, and the Mac resolves it to a predefined script/command. Benefits: smaller iPhone app, no shell-injection surface, no need to redeploy the iPhone app when scripts change.

```
iPhone sends: { "actionId": "start-work" }
Mac resolves: ~/Scripts/start-work.sh → executes → returns result
```

## Goals (v1)
- Create/edit/organize custom buttons on the Mac, grouped into categories
- Each button runs one or more actions (app launch, shell command, script, macOS Shortcut)
- Trigger from iPhone; see live status (running/success/fail) and logs
- Auto-discover Mac via Bonjour; pair once, authenticate every request after
- Fully offline-capable (LAN only), no cloud backend
- Architecture should scale to multiple Macs later

## Non-Goals (v1)
Screen/mouse/keyboard sharing, cloud sync, WAN access, plugin marketplace, multi-user accounts, visual workflow builder — defer all to later phases.

## Components
| Component | Role |
|---|---|
| **Mac Control Studio** (SwiftUI, macOS) | Create/edit controls, manage devices, view logs, configure agent |
| **Mac Control Agent** (Swift, background service) | Runs independently of Studio; handles discovery, auth, execution, logging |
| **iPhone Remote** (SwiftUI, iOS) | Discover + pair with Mac, display/trigger controls, show results — minimal config |

Agent must keep running even if Studio is closed.

## Data Model
```json
{
  "id": "start-work",
  "name": "Start Work",
  "icon": "laptopcomputer",
  "color": "#6366F1",
  "categoryId": "development",
  "requiresConfirmation": false,
  "actions": [
    { "type": "openApplication", "application": "Visual Studio Code" },
    { "type": "shellCommand", "command": "docker compose up -d" }
  ]
}
```
**Action types:** `openApplication`, `shellCommand`, `runScript`, `macOSShortcut` — (`wait`, `conditional` are advanced/hidden in v1).
**Execution:** sequential by default in v1; parallel deferred (dependency risk).
**Confirmation flag** required for destructive actions (shutdown, restart, prod deploy, file deletion).

## Networking & Discovery
- Bonjour/mDNS service: `_maccontrol._tcp` — iPhone auto-finds Mac, no manual IP entry
- REST + WebSocket API on LAN only, no public internet exposure by default

**API surface:**
```
GET  /api/v1/info
GET  /api/v1/config
POST /api/v1/pair
POST /api/v1/execute
GET  /api/v1/status
GET  /api/v1/executions
```
WebSocket carries execution progress (`executionStarted` → `executionCompleted`), Mac status, and live logs — needed for long-running processes (e.g. `npm run dev`) that shouldn't block the HTTP executor; design this in from v1, not Phase 5.

## Pairing & Security
1. First connection: Mac shows a short-lived, rate-limited pairing code (e.g. 6 digits, 60s expiry, lockout after 5 failed attempts); iPhone enters it.
2. On success, Mac issues a persistent device credential, stored in iPhone Keychain.
3. **Every subsequent request must be authenticated** — recommend a signed request (HMAC with nonce/timestamp) rather than a static bearer token, to prevent replay attacks.
4. **Transport encryption**: self-signed TLS cert generated at pairing, pinned by the iPhone (trust-on-first-use) — LAN ≠ inherently safe (shared Wi-Fi, hostile guests).
5. iPhone can only ever request a predefined `actionId` — never raw shell commands — closing off remote-shell abuse.
6. Devices are listed/revocable from Mac Studio; revoked devices must re-pair.
7. Scripts run under the user's own account, never auto-`sudo`.

## Storage
Local-only, Mac is source of truth:
```
~/Library/Application Support/MacControl/
├── config.json
├── devices.json
├── logs/
└── scripts/
```
SQLite optional if data model grows. iPhone caches config for fast display but re-syncs from Mac on connect. Config export/import supported for migration.

## Logging
Every execution logged with device, timestamp, per-step detail, duration, and final status.

## Roadmap
| Phase | Delivers |
|---|---|
| 1 | Mac HTTP server + iPhone client, manual IP, one hardcoded action (open Calculator) |
| 2 | Bonjour discovery + pairing |
| 3 | Full control model, categories, execute-by-ID |
| 4 | Mac Studio UI: editor, icon/color picker, test button |
| 5 | Multi-action sequencing, scripts, Shortcuts, progress, logs |
| 6 | Security hardening: credentials, Keychain, revocation, confirmation flows |
| 7 | Polish: animations, haptics, dark mode, error/empty states |
| 8 | Advanced: widgets, Watch, Siri, multiple Macs, workflow builder, variables |

Build order: **execution → local server → iPhone connection → single button → discovery → pairing/security → config model → Studio UI → multi-action → logs → polish.** Don't start on UI before the core loop works.

## MVP Definition
- **Mac:** Studio + agent, button creation (icon/name/category), shell/app/script actions, test button, logs
- **iPhone:** discovery, pairing, categorized buttons, execute, success/error state, connection status
- **Networking:** Bonjour + local HTTP/WebSocket + authenticated requests

## Definition of Done (v1.0)
User can install on Mac + iPhone, create a control end-to-end (name/icon/action/test/save), auto-discover and pair the iPhone, tap a button to trigger real execution on the Mac, see success/failure and logs, revoke and re-pair the device — all without leaving the local network.
