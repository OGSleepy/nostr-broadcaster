# BROADCAST ⚡

A Nostr note rebroadcaster. Paste any npub, pick a time range or note count, and blast their notes across 23 relays — including blast relays that propagate events further into the network.

No install. No backend. No build step. One HTML file.

---

## What it does

Fetches a user's kind:1 notes from multiple read relays in parallel, then rebroadcasts each event to every relay you have selected. Blast relays in the default list automatically forward events onward to other relays they're connected to, maximising reach.

Useful for:
- Recovering notes lost due to relay outages
- Ensuring a profile's history is visible across the network
- Pushing your own notes to relays you've recently joined
- Helping a friend whose notes aren't propagating

---

## Usage

1. Open `nostr-broadcaster.html` in any browser
2. Paste an `npub1...` or 64-char hex pubkey
3. Choose **By Time** or **By Count**
4. Select which relays to broadcast to (all selected by default)
5. Hit **⚡ Broadcast Notes**

The activity log shows fetch and publish results in real time. A summary of sent/failed counts appears when the run completes.

---

## Relay list

The default list ships with 23 relays across two categories.

**⚡ Blast relays** rebroadcast events to other relays they're peered with — one publish becomes many. These are selected by default and cover the widest network surface.

**Standard relays** are high-traffic relays with large subscriber bases.

You can:
- Check or uncheck individual relays
- Use **Select Blasters Only** to target blast relays exclusively
- Add any custom relay via the input at the bottom of the list
- Remove any relay from the list

Relay selections are not saved between page loads.

---

## Scope options

| Mode | Options |
|------|---------|
| By Time | Today, This Week, This Month, 3 Months, 6 Months, 1 Year, All Time |
| By Count | Any number (default 50) |

In **By Time** mode the fetch uses a `since` timestamp filter and caps at 2000 notes. In **By Count** mode the most recent N notes are fetched and broadcast.

---

## Deployment

Drop `nostr-broadcaster.html` anywhere that serves HTML:

**Cloudflare Pages**
Upload the file to your repo and point Pages at it. No build command needed.

**GitHub Pages**
Rename to `index.html`, push to a `gh-pages` branch or `/docs` folder.

**Local**
Open directly in a browser. WebSocket connections work from `file://` in most browsers.

---

## Technical notes

- Built on [nostr-tools v2](https://github.com/nbd-wtf/nostr-tools) (official browser bundle via unpkg)
- Fetch uses `SimplePool.querySync()` across 7 read relays in parallel with a 12-second timeout
- Publish uses `SimplePool.publish()` firing all selected relays simultaneously, with an 8-second per-relay timeout
- Events are deduplicated by `id` before broadcasting
- No private key required — only rebroadcasts already-signed public events
- No tracking, no analytics, no external requests beyond Google Fonts, unpkg CDN, and Nostr relays

See `SPEC.md` for the full technical specification.

---

## Stack

```
nostr-tools v2    WebSocket relay connections, npub decoding
Space Grotesk     UI typography
Space Mono        Monospace labels and log
Vanilla JS        No framework
Single HTML file  No bundler, no build, no deps to install
```
