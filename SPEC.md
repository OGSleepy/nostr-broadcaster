# BROADCAST — Technical Specification

## Overview

A single-file browser application that fetches a Nostr user's kind:1 notes and rebroadcasts them to a configurable set of relays, with heavy emphasis on blast/rebroadcast relays that propagate events further across the network.

No build step. No backend. No server. One HTML file.

---

## Architecture

### Runtime

- Pure browser JavaScript (ES2020+)
- Single `.html` file — CSS, JS, and markup all inline
- Zero dependencies beyond the nostr-tools browser bundle loaded from unpkg CDN

### Library

```
nostr-tools v2 (browser bundle)
Source: https://unpkg.com/nostr-tools/lib/nostr.bundle.js
Exposed as: window.NostrTools
Key APIs used: SimplePool, nip19
```

---

## Relay Configuration

### Relay Types

| Type | Badge | Behavior |
|------|-------|----------|
| `blast` | ⚡ blast | Rebroadcast-capable relays that forward events to other relays in the network |
| `standard` | relay | High-coverage relays with large subscriber bases |

### Default Relay List

#### Blast Relays (9)
```
wss://nostr.mutinywallet.com
wss://purplepag.es
wss://nostr.wine
wss://relay.nos.social
wss://nostr21.com
wss://blastr.f7z.xyz
wss://nostr-relay.nokotaro.com
wss://nostr.coinfundit.com
wss://relay.nostr.com.au
```

#### Standard Relays (14)
```
wss://relay.damus.io
wss://nostr.fmt.wiz.biz
wss://relay.snort.social
wss://nos.lol
wss://relay.current.fyi
wss://offchain.pub
wss://nostr.zbd.gg
wss://relay.primal.net
wss://nostr.oxtr.dev
wss://nostr-pub.wellorder.net
wss://relay.nostr.wirednet.jp
wss://soloco.nl
wss://nostr.onsats.org
```

### Read Relays (fetch only, never write)
```
wss://relay.damus.io
wss://relay.primal.net
wss://nos.lol
wss://nostr.fmt.wiz.biz
wss://relay.snort.social
wss://offchain.pub
wss://nostr.oxtr.dev
```

### Permanently Banned Relays
```
wss://relay.nostr.band  — offline, must never appear anywhere in this project
```

---

## Data Flow

### Step 1 — Input Validation

Accepts either:
- `npub1...` — decoded via `nip19.decode()`, type must be `npub`
- 64-character lowercase hex pubkey — validated via regex `/^[0-9a-f]{64}$/`

Fails fast with a log error if input is invalid before making any network requests.

### Step 2 — Fetch

```
SimplePool.querySync(READ_RELAYS, filter)
```

- Connects to all 7 read relays in parallel
- Collects events until EOSE from each relay
- Hard timeout: **12 seconds** — resolves with whatever was collected
- Results are deduplicated by event `id` after collection
- Events sorted by `created_at` descending (newest first)

#### Filters

Time mode:
```js
{ kinds: [1], authors: [pubkey], since: <unix_timestamp>, limit: 2000 }
```

Count mode:
```js
{ kinds: [1], authors: [pubkey], limit: <user_input> }
```

Time scope → unix timestamp mapping:
| Scope | Since |
|-------|-------|
| Today | now − 86400 |
| This Week | now − 604800 |
| This Month | now − 2592000 |
| 3 Months | now − 7776000 |
| 6 Months | now − 15552000 |
| 1 Year | now − 31536000 |
| All Time | no `since` field |

### Step 3 — Broadcast

```
SimplePool.publish(selectedRelays, event)
```

- A new `SimplePool` instance is created for publishing (separate from the fetch pool)
- For each event, `pool.publish()` fires connections to all selected relays simultaneously
- Returns one promise per relay
- Each relay promise is individually wrapped with a **8-second timeout**
- `Promise.all()` waits for all relay results before moving to the next event
- A `setTimeout(0)` yield between events keeps the UI thread responsive
- Both pools are explicitly closed after use via `pool.close(relays)`

---

## UI Components

### Header
- Animated conic-gradient spinner (6s loop)
- App title in Space Mono
- Live status indicator dot (grey = idle, green + glow = active)

### Card 01 — Target Profile
- Single text input accepting npub or hex pubkey

### Card 02 — Scope
- Mode toggle: **By Time** / **By Count**
- Time picker: 7 buttons in a responsive grid (4-col desktop, 2-col mobile)
- Count picker: number input (min 1, max 5000, default 50)

### Card 03 — Relay List
- Scrollable list (max-height 240px) with per-relay checkbox, URL, badge, remove button
- Relay controls: Select All, Deselect All, Select Blasters Only
- Add relay input with auto `wss://` prefix if omitted
- Duplicate detection on add
- Live count display: `N / Total selected`

### Progress Bar
- Hidden until broadcast starts
- Tracks event-level progress (per event, not per relay-send)
- Fills to 100% on completion

### Summary
- Shown after broadcast completes
- Three stat boxes: Notes Found, Sent OK, Failed

### Activity Log
- Timestamped entries (HH:MM:SS)
- Four message types: `info` (blue), `ok` (green), `warn` (amber), `err` (red)
- Auto-scrolls to latest entry
- Clear button

---

## Fonts

```
Space Grotesk — UI text (weights 300–700)
Space Mono    — monospace labels, inputs, log, header
Source: Google Fonts (CDN)
```

---

## Color Tokens

| Token | Hex | Usage |
|-------|-----|-------|
| `--bg` | `#080b10` | Page background |
| `--surface` | `#0d1117` | Card backgrounds, header |
| `--panel` | `#111820` | Input backgrounds, relay items |
| `--border` | `#1e2d3d` | All borders |
| `--accent` | `#00c2ff` | Primary accent, active states |
| `--accent2` | `#7b2fff` | Secondary accent, count mode |
| `--hot` | `#ff3c6f` | Broadcast button, blast badge, errors |
| `--text` | `#d4e4f0` | Body text |
| `--muted` | `#4a6477` | Secondary text, placeholders |
| `--success` | `#00e5a0` | OK states, checked relays, live dot |
| `--warn` | `#ffb800` | Warning log entries |

---

## Error Handling

| Scenario | Behaviour |
|----------|-----------|
| Invalid npub / hex | Log error, abort before any network call |
| No relays selected | Log error, abort |
| Fetch timeout (12s) | Resolve with whatever events were collected |
| Zero events found | Log error, re-enable button, abort broadcast |
| Relay publish timeout (8s) | Counted as failed, does not block other relays |
| Relay publish rejection | Counted as failed, does not throw |
| nostr-tools not loaded | Log error, abort |

---

## Constraints & Non-Goals

- Read-only access to Nostr — no signing, no private key input
- Does not create new events — only rebroadcasts existing signed events
- No persistent storage — relay list resets on page reload
- No authentication — works with any public npub
- No rate limiting logic — relies on relay-side enforcement
- No pagination UI — limit handled server-side via filter
