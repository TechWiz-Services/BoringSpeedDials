# Functional Specification: Grouped Speed-Dial New-Tab Extension (no install-time permission warnings)

Working title: **TBD**. Choose your own name, logo, and visual design. Do not reuse any existing product's name, icons, or wording (including its menu labels).

This document describes behavior only. It specifies no visual design, copy text, or icons. Make your own original choices for those.

## 0. Design goals (in priority order)

1. **No install-time permissions that carry a warning.** The only install-time permission is `storage`, which shows no warning or prompt and exists solely so administrators can apply policy (10.1). `host_permissions` is empty. Every other permission is optional, requested from a user click at the moment a feature needs it, and every feature degrades gracefully if refused. Section 10 is normative.
2. **Never request** `<all_urls>` or any host permission, `history`, `bookmarks`, `cookies`, `scripting`, `webRequest`, `downloads`, `geolocation`, `topSites`, `sessions`, or `management`.
3. **Local first.** Phase 1 has no server at all. Everything works without network in every phase; server features (sync, weather, lookup) are additive.
4. **Zero-knowledge sync.** The server can never read a user's dials, titles, URLs, or images, whoever runs it.
5. **One server.** The extension talks to exactly one server at a time: the developer's server by default, or a self-hosted one the user or an administrator configures (section 8). It never contacts any other service.

## 1. Purpose

A browser extension that replaces the new-tab page with a visual start page: a grid of tiles ("dials"), each opening a website with one click, organized into named **groups** shown as tabs. A visual replacement for bookmarks.

## 2. Platform

- **Chromium only:** Chrome and Microsoft Edge (primary test targets), plus Brave, Vivaldi, and Opera on a best-effort basis. Manifest V3.
- Distributed through the Chrome Web Store and Microsoft Edge Add-ons. The two stores assign **different extension IDs**; servers must accept both origins (8.3).
- No Firefox build for now. Don't add abstractions for it.

## 3. Core concepts

| Concept | Meaning |
|---|---|
| Dial | A tile. Has a type (section 5), a title, and type-specific content. |
| Group | A named collection of dials, shown as a tab/selector. One group is active at a time. |
| Folder (Phase 2) | A container that holds multiple groups. |
| Layout | How dials in a group are arranged (section 6). |
| Logo | The small site icon shown on a link dial (4.3). No page screenshots are used anywhere. |
| Server (Phase 3) | The single backend the extension talks to: the developer's by default, or self-hosted (section 8). |

## 4. Core features

### 4.1 Start page
- Overrides the new-tab page (`chrome_url_overrides.newtab`; not a permission). Shows the active group; remembers the last active group.
- Group switching: visible selector plus keyboard (arrows / number keys).
- Renders entirely from local data; no network call needed to display it.
- **First run:** a short setup screen explaining each optional capability with an "enable" button per feature (section 10). Everything is off until the user enables it.

### 4.2 Link dials
- **Fields:** URL (required), title (defaults to the tab title when added from a tab, otherwise the server lookup result if enabled (4.10), otherwise the hostname; always editable), logo.
- **Edit / delete / duplicate / move to another group** via a per-dial context menu on the start page. This is the page's own HTML menu and needs no permission.
- **Reorder:** turn on a drag-and-drop mode to move dials by dragging; drop onto a group-selector entry to move across groups.
- **Open behavior:** left click = same tab. Middle-click and Ctrl/Cmd-click always open a background tab.
- Dials are opened using `chrome.tabs.update` / `chrome.tabs.create`, which need no permission.

### 4.2.1 Selecting and moving multiple dials (Phase 1)
- **Selection mode:** a "select" button on the start page (and an entry in the per-dial context menu) turns it on. Each dial shows a checkbox, and clicking a dial toggles it instead of opening it. Esc or a "done" button leaves selection mode and clears the selection.
- **Selecting:** click to toggle; Shift-click selects the range from the last clicked dial; Ctrl/Cmd-click toggles without leaving selection mode. "Select all" and "select none" apply to the current group. Keyboard: arrow keys move focus, Space toggles.
- Works for every dial type (link, multi-link, clock, and so on). The selection covers only the active group; switching groups clears it.
- **Action bar** while anything is selected: shows the count and a **"move to…"** action that lists the other groups plus "new group…". Moved dials are appended to the end of the target group, keeping their relative order. The target group's layout applies; for a fixed grid that runs out of cells, add rows (or tell the user the group is full, if it's already at the 9 × 9 maximum, and move nothing).
- **Drag:** in selection mode, dragging any selected dial onto a group-selector entry moves all selected dials there.
- A move is one undoable step: show a brief "moved N dials · undo" message on the page, and include it in local snapshots like any other edit (4.9).
- Must stay responsive when moving all 500 dials of a group (4.6). Write the move as a single IndexedDB transaction, so it either fully happens or not at all.

### 4.3 Logos (no screenshots, no host permissions)

A link dial shows the site's icon centered on a colored tile, with the title below. Tile background: a color derived from the hostname (user can override). Logo sources, in order of preference:

1. **Browser icon cache (optional `favicon` permission).** Chrome serves site icons from its own icon store to the extension at `chrome.runtime.getURL("/_favicon/") + "?pageUrl=<dial URL>&size=<px>"`. This is a URL on the extension's own origin, so it works with the strict CSP (`img-src 'self'`) and causes no request to the site or any server. Request the largest useful size (e.g., 32 or 64; Chrome scales whatever it has). Sites never visited in this browser profile return a generic placeholder; detect that (compare against the placeholder for a known-nonexistent URL) and fall back to 3.
   - Copy the icon into IndexedDB as a data URL when the dial is created or first rendered, so logos keep working (and can sync, encrypted) even if the permission is later revoked.
2. **Tab icon at add time (no extra permission).** When adding from a tab under `activeTab` (4.4), the tab's `favIconUrl` is readable. If it is a `data:` URL, store it directly. Otherwise, try to fetch it once, at add time. This is the only request the extension ever makes to anything other than the configured server (8.3), and it only goes to a site the user is visiting right now. It only succeeds if the icon host sends CORS headers (see 15.4). On failure, fall back.
3. **Generated logo:** the first letter of the title on the colored tile.
4. **User upload:** any image, any time; overrides everything else.

If the user has enabled **server lookup** (4.10, Phase 3), it is tried before sources 1–3, because it can return larger icons. A setting "prefer local logos" reverses this so the server is only asked when sources 1 and 2 find nothing. An uploaded image always wins.

Provide "refresh logo" per dial and "refresh all logos" per group (re-reads the cache from source 1, or the server if enabled). Higher-resolution sources such as touch icons or a page's `<link rel="icon">` list would require reading the page (`scripting` or host access) and are out of scope.

### 4.4 Adding dials from the browser (key feature)

Requires the optional `contextMenus` permission (for the menus) and `activeTab` (for reading the tab). Without `contextMenus`, only the toolbar button and shortcut work.

- **Tab-strip right-click** on any tab → "add to speed dial" (context `"tab"`). Documented in Chrome's API since Chrome 148 and confirmed working in Edge. Feature-detect: if creating a `"tab"` context item throws, omit it.
  - Save the clicked tab's URL, title, and logo (4.3). For a background tab, see 15.1.
- **Page right-click** → same action for the current page.
- **Link right-click** → same action for the link target (URL and link text; logo from source 1 or 3).
- **Toolbar-button right-click** → same action, plus "save all tabs in this window" (4.5).
- **Toolbar button click** → small popup: choose group (or new group), edit title, confirm. Clicking the toolbar button grants `activeTab`.
- **Keyboard shortcut** (rebindable through `chrome://extensions/shortcuts`) → quick-add to the last-used group.
- Each menu entry offers the user's groups as a submenu (keep it current when groups change), or a "choose…" entry that opens the popup.
- Confirm saves with a brief toolbar badge, not a notification.

### 4.5 Save all tabs of this window as a multi-dial

The only feature that uses `tabs`, which is requested only when the user starts this action.

**Entry points:** a button in the toolbar popup and an entry in the toolbar-button right-click menu. Both are hidden entirely when an administrator disables the feature (10.1).

**Flow:**
1. User triggers the action. Record the **source window ID** (from the clicked tab's `windowId`, or `chrome.windows.getCurrent()` in the popup; neither needs a permission).
2. If `tabs` is already held (the user previously chose to keep it), skip to step 5.
3. Otherwise open an **explanation window**: `chrome.windows.create({ type: "popup", url: "save-tabs.html?source=<windowId>" })`, focused, about 480 × 600, centered over the source window. No permission is needed for this.
   - This separate window is required, not cosmetic. `permissions.request` only works from a user click inside an extension page. A right-click handler runs in the service worker, which can't request permissions, and the toolbar popup closes as soon as Chrome's permission dialog takes focus.
   - Content (write your own wording):
     - What the feature does.
     - Exactly what the permission lets the extension see: the address and title of **every open tab in every window** for as long as it is granted. Not page contents, not history, not passwords.
     - That the extension will give the permission back immediately after saving.
     - That Chrome will show its own warning next ("Read your browsing history"), and why the wording sounds broader than what happens.
     - A checkbox "keep this permission for next time", unchecked by default.
     - **Continue** and **Cancel** buttons.
4. **Continue** calls `chrome.permissions.request({ permissions: ["tabs"] })` directly in its click handler.
   - Declined, or the request throws (for example, because policy blocks it): show a short explanation in the same window ("not granted; it may also be blocked by your organization") and a Close button. Nothing is saved. No retry prompts later.
5. In the same window, read `chrome.tabs.query({ windowId: <source> })` and show a **preview list** with a checkbox per tab, pre-unchecked for: pinned tabs (setting), `chrome://` / `edge://` / extension pages, `about:blank`. Fields: multi-dial name (default: date and time), target group, click behavior.
6. **Save** creates the multi-dial; **Cancel** discards.
7. Either way, unless "keep" was checked, call `chrome.permissions.remove({ permissions: ["tabs"] })`, then close the window.

**Safety nets** (release the permission even if the flow is interrupted):
- If the explanation window is closed early (`windows.onRemoved` in the service worker), release `tabs`.
- On service-worker startup, if `tabs` is held but the user never chose "keep", release it.
- The tab data read in step 5 lives only in that window's memory until saved; nothing is logged or persisted except the saved dial.

**Multi-dial behavior:**
- Multi-dial tile: a small mosaic of the first few entries' logos (4.3) plus a link count.
- Multi-dial click behavior (setting): open all in background tabs, open in a new window, or show a picker popover.

### 4.6 Groups
- Create, rename, delete, reorder. Deleting a non-empty group needs confirmation.
- Per-group layout and optional background.
- Handle at least 10 groups × 500 dials without noticeable lag.

### 4.7 Search
- Search box on the start page. Filters dial titles and URLs (and tags, from Phase 2) across all groups; Enter opens the top result.
- Fallback: send the query to the user's configured search engine as a normal navigation (no extension network call).

### 4.8 Settings
- Open behavior, theme (light/dark/system), background, dial size and spacing.
- Show/hide: search box, group selector, titles.
- Permissions screen (10), clocks (5.1).
- Phase 3: server address (8.2), sync (9), server lookup (4.10), weather units and place (5.2).

### 4.9 Storage, backup, import, export
- All user data in **IndexedDB** (accessible from both the page and the service worker). `chrome.storage` is used only to read administrator policy (`chrome.storage.managed`, 10.1), never for user data.
- **Export:** one JSON file through a `blob:` download link (no `downloads` permission).
- **Import:** a file input. Merge or replace. Also import a browser bookmarks HTML export (no `bookmarks` permission).
- Versioned schema, documented in the repo.
- **Local snapshots:** keep the last N (default 5), restorable; snapshot current state before restoring.

### 4.10 Server lookup for titles and logos (Phase 3, opt-in, off by default)

The configured server (section 8) can fetch a page's title and icons on the user's behalf. This works for dials added by typing a URL or by import, needs no browser permission, and usually yields better icons than the browser cache (touch icons, web-app manifest icons). **The cost is privacy: the server operator learns which pages the user adds.**

**Enabling:**
- A switch in settings ("Look up titles and logos via the server") and an offer on the first-run setup screen, both off by default. Hidden if the configured server doesn't offer lookup (8.2).
- Turning it on first shows an explanation dialog (write your own wording) that states plainly:
  - When this is on, the address of each page you add or refresh is sent to the configured server, along with your IP address, as with any web request. Name the server's host and who runs it: by default, the developer of this extension; for a self-hosted server, whoever operates it.
  - **The server operator can therefore see which pages you add.** This is not covered by the end-to-end encryption of sync, because the server must read the address to fetch the page.
  - For the default server: what it keeps (no association with a sync account, no long-term logs of addresses or IPs, only short-lived caching and rate limiting), the actual retention period, and a link to the privacy policy. For a self-hosted server, say that its retention depends on its operator.
  - The private alternatives: browser icon cache (`favicon` permission), the tab's own icon when adding from a tab, letter tiles, or typing a title yourself.
  - Buttons: **Enable** and **Keep off**.
- Also offered contextually: when a typed URL dial ends up with a letter tile, show a subtle "look up via server?" action on that dial. Choosing it opens the same explanation dialog.
- Changing the server address (8.2) turns lookup off again; the user re-enables it after seeing the dialog for the new server.

**What is sent:**
- Default: scheme, host, and path only. Query string and fragment are stripped, since they often contain session tokens or personal data. Per-dial option "send full address" for pages whose title depends on the query string.
- Setting "logos only (send domain only)": sends just the origin; the title stays the hostname or user-typed. This gives the operator less information.
- **Never sent, regardless of settings:** `localhost`, IP literals in private or reserved ranges, single-label hostnames (e.g., `intranet`), `.local` / `.internal` / `.lan` / `.home.arpa`, and non-`http(s)` schemes. These dials use local sources only, so internal hostnames never leak.
- Anonymous request: no sync credentials, no cookies, no account ID.

**When requests happen:**
- On add and on "refresh logo / title", only while the setting is on.
- Bulk "look up all dials" needs its own confirmation showing how many addresses will be sent.
- Never in the background, and never automatically for existing dials when the setting is turned on (unless the user runs the bulk action).

**Client handling of responses:**
- The title is treated as plain text (never inserted as HTML), trimmed, and length-capped.
- The icon arrives as image bytes from the server. It is stored in IndexedDB as a data URL, so it displays under `img-src 'self' data: blob:`.
- Failures are silent: fall back to local sources.

**Server requirements:**
- `GET /meta?url=…&mode=full|domain` returns `{ title, icon }`, where `icon` is a re-encoded PNG (for example, up to 128 × 128) or null. Pick from apple-touch-icon, web-app manifest icons, `<link rel="icon">`, then `/favicon.ico`.
- **SSRF protection** (required, since the server fetches user-supplied URLs):
  - Resolve DNS and reject private, loopback, link-local (including `169.254.169.254`), and reserved ranges, for both IPv4 and IPv6.
  - Re-check after every redirect (max 3); `http(s)` only.
  - No cookies or credentials. Timeouts (around 5 s), response-size caps (for example, 1 MB of HTML, 512 KB per icon), and parsing only the `<head>`.
  - Self-hosters may want to allow their own internal ranges; make that an explicit, documented server setting that is off by default.
- Re-encode icons server-side, never pass raw bytes through (strips SVG scripts and malformed files).
- Don't log URLs or IPs beyond what rate limiting needs; cache results by URL for a short documented period; never link to sync accounts.

## 5. Dial types

| Type | Behavior | Phase |
|---|---|---|
| Link | 4.2. | 1 |
| Multi-link | Several URLs in one tile (4.5). | 1 |
| World clock | 5.1. Local only, no network. | 1 |
| Search | Search-engine URL template (`%s`); several engines per tile. Plain navigation, no extension network call. | 2 |
| Weather | 5.2. Through the configured server only. | 3 |

### 5.1 World clock dial
- Shows one or more clocks in a tile, each with a user label, the time, and a day indicator (e.g., "+1 day") relative to local time. Options: 12/24-hour, seconds, date, digital or analog.
- **Standard zones:** searchable list from `Intl.supportedValuesOf('timeZone')` (IANA zones). The time zone database ships with the browser and updates with it; the extension never bundles its own copy.
- **Custom zones** (for when the browser's data is outdated or a region changes its rules):
  - Label, standard UTC offset (±HH:MM, supporting :30 and :45 offsets).
  - Optional daylight-saving rule: DST offset (default +1:00), start and end each as "Nth or last weekday of month at local time" (e.g., last Sunday of March at 02:00).
  - **Advanced:** paste a POSIX TZ string (e.g., `CET-1CEST,M3.5.0,M10.5.0/3`) instead of using the form. Support the `Mm.w.d/time` rule format; reject other formats with a clear error. Remember POSIX inverts the offset sign.
  - Live preview of the resulting current time and the next two transitions before saving.
- A custom zone can be created as a copy of an IANA zone's current offset, to make "my country changed its rules" fixes quick.
- Timekeeping uses the device clock; no network.

### 5.2 Weather dial
- User picks a place by name through the server's place-search endpoint. **No geolocation.**
- Current conditions plus a multi-day forecast; unit choice.
- The extension fetches only from the configured server, which fetches from an upstream provider (check its license terms) and caches by place and time bucket. A self-hosted server may have weather turned off (for example, if its operator has no upstream API key); then the weather dial type is hidden (8.2).
- **Privacy:** weather endpoints reveal the place to the server; disclose this. The server must not log IPs or place-to-client mappings beyond short-term rate limiting.
- Refreshed when the start page opens and at most every N minutes while it stays open. No background polling.

## 6. Layouts
- **Fixed grid:** rows × columns (up to 9 × 9).
- **Dynamic grid:** columns fit the window width; rows grow.
- **Free grid (Phase 2):** dials span multiple cells.

## 7. Organization aids (Phase 2)
Tags (unlimited, filterable, searchable); folders for groups; focus mode (schedule locking tagged dials).

## 8. Server and self-hosting (Phase 3)

Phase 1 and Phase 2 ship no server and no server code paths. Everything in this section starts in Phase 3.

### 8.1 One server program, two ways to run it
- A single server program provides everything server-side: place search and weather proxy (5.2), title/logo lookup (4.10), and encrypted sync (9).
- **Default:** the developer's own hosted instance. A fresh install uses it without any configuration.
- **Self-hosted:** the server source is published under an open-source license, with a container image and a setup guide. A self-hosted instance exposes the same API as the default one.
- The client treats both the same way. Zero-knowledge sync (9) holds whoever runs the server; lookup and weather privacy (4.10, 5.2) depend on the operator, and the disclosures say so.

### 8.2 Choosing the server
- Settings → "Server": shows the current server's address, default pre-filled. The user can enter another address; `https://` only.
- Before switching, the client calls `GET /info` on the new address, which returns the server's API version and which features it offers (`sync`, `weather`, `lookup`). Reject incompatible API versions with a clear error. Features the server doesn't offer are hidden in the extension.
- Switching shows a confirmation that explains:
  - A sync account belongs to one server. The user signs in again or creates an account on the new server; nothing moves between servers automatically. Local data stays on the device, and the user can upload it to the new server's account.
  - Server lookup (4.10) is turned off again and must be re-enabled for the new server.
  - Weather dials refetch from the new server.
- **No fallback:** if the configured server is unreachable, server features fail clearly. The extension never falls back to the default server on its own.
- "Reset to default server" is one click (with the same confirmation).
- An administrator can set and lock the server with the `serverUrl` policy (10.1). A locked address shows "managed by your organization" and cannot be edited.

### 8.3 Network enforcement
- Extension-page CSP, in every phase: `script-src 'self'; object-src 'self'; connect-src https:; img-src 'self' data: blob:`. No `'wasm-unsafe-eval'`, no `'unsafe-eval'`, no remote scripts.
- `connect-src` has to allow any `https:` origin because the server address is user-configurable and the CSP is fixed at build time. So the "one server" rule (goal 5) is enforced in code:
  - All network access goes through a single module. A lint rule bans `fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`, and `navigator.sendBeacon` everywhere else; CI fails if the rule is disabled anywhere.
  - That module only allows: (a) the configured server's origin, exactly (Phase 3), and (b) the one-time fetch of the `favIconUrl` of a tab the user is adding at that moment (4.3, source 2). In Phase 1 and 2 only (b) exists.
  - Automated tests drive every feature and assert that no request goes anywhere else.
- `img-src` stays `'self' data: blob:`, so images can never trigger third-party requests. Remote images are always fetched through the network module and stored as data URLs.
- **CORS:** the server responds with CORS headers allowing exactly the extension's origins (`chrome-extension://<Chrome Web Store ID>` and `chrome-extension://<Edge Add-ons ID>`), so no `host_permissions` are needed. The self-hosted server ships with both IDs in its default configuration. Use bearer-token headers, not cookies.

## 9. End-to-end encrypted sync (Phase 3)

### 9.1 Trust model
The server stores only ciphertext and minimal metadata. The extension code reaches users through the browser stores, not from the server, so the server (default or self-hosted) cannot ship altered client code. Publish the client source with reproducible builds so this is verifiable.

### 9.2 Keys

Key derivation uses only WebCrypto (PBKDF2, HKDF, AES-GCM). Nothing is bundled as WASM.

PBKDF2 isn't memory-hard, so GPUs can guess passphrases quickly against a stolen database. To make that useless, every account also has a **secret key**: a random value that lives only on the user's devices and in their emergency kit, never on the server.

**Secret key (created at setup):**
- The user must supply a random seed: setup asks them to type or mash random keys into a box until a progress indicator fills (for example, at least 64 characters). Collect the keystroke timings as well as the characters.
- Combine it with the browser's generator: `secretKey = HKDF-SHA256(ikm = 32 bytes from crypto.getRandomValues() ‖ SHA-256(seed text ‖ timings), info = "secret-key v1")`, truncated to 128 bits. The browser's randomness alone is enough, so a weak seed can never make the key weak; the seed only adds to it. Never derive the key from the user's seed alone.
- Encode it for humans: version prefix, Base32 in groups of four or five characters, and a checksum so typos are caught before any key derivation runs.

**Passphrase:**
- The user sets a **sync passphrase**. Enforce a minimum length (at least 12 characters) and show a strength estimate.

**Master key:**
- `K_pw = PBKDF2-HMAC-SHA256(passphrase, salt, iterations, 256 bits)`, through WebCrypto.
- Iterations: default **2,000,000**, never below 1,000,000. That's above OWASP's 600,000 baseline and takes around a second on a typical desktop. It only runs on sign-in on a new device and on passphrase change, never on ordinary use, so the cost is acceptable. Show a progress indicator while it runs.
- Salt: random 128-bit, per account.
- `masterKey = HKDF-SHA256(ikm = K_pw ‖ secretKey, salt = account salt, info = "master v1")`.
- Salt, iteration count, and KDF version are stored on the server as non-secret account parameters, so every device derives the same key. The iteration count can be raised on any passphrase change.

**Derived keys:**
- HKDF from the master key into:
  - **Auth key:** sent to the server as the login secret; the server stores only a slow hash of it (for example Argon2id; this is server-side and doesn't affect the extension). Because it depends on the secret key, a stolen server database can't be brute-forced even if the passphrase is weak.
  - **Wrapping key:** never leaves the device.
- A random 256-bit **data key** encrypts all content; the server holds it only wrapped. Changing the passphrase re-wraps the data key without re-encrypting data.

**Emergency kit and recovery key:**
- At setup, offer an **emergency kit** as a printable page and a `blob:` download: server address, account ID, and secret key, with a blank space for writing the passphrase by hand. Require the user to confirm they saved it by retyping one group of the secret key.
- Also generate a separate random 256-bit **recovery key**, shown once and kept apart from the kit. It wraps a second copy of the data key and derives its own recovery auth key. With the recovery key alone, the user can regain access and set a new passphrase; afterwards, generate a new secret key and a new recovery key.
- **No recovery** if the passphrase or the secret key is lost and the recovery key is lost too. State this at setup and require confirmation.

**On the device:**
- After sign-in, store the secret key and the unwrapped data key in IndexedDB, the data key as a non-extractable `CryptoKey`. The passphrase is never stored. Daily use needs no passphrase; it's needed on a new device, on passphrase change, and after signing out.
- Adding a device needs the server address, account ID, secret key (typed from the kit or copied from another device), and passphrase.
- Local data in IndexedDB is not encrypted at rest. It's protected like the rest of the browser profile. Document this.

### 9.3 Encryption
- AES-256-GCM through WebCrypto, a fresh random 96-bit nonce per encryption; record ID and revision as additional authenticated data (prevents record swapping and rollback).
- Encrypt group/dial records, settings, cached logos and uploaded images. Pad sizes to coarse buckets.
- Server-visible metadata: random record IDs, size bucket, revision counters, timestamps. Document this.

### 9.4 Accounts
- Individual and organization accounts, with admins for organizations.
- Organization accounts come after individual sync works. Organization admin features never get access to members' data keys.

### 9.5 Sync protocol
- One record per dial/group/settings; last-writer-wins per record by (revision, device ID); tombstones for deletions.
- Incremental pull since a server cursor; offline edits queue and replay; failures shown clearly; never silently drop local data.
- Sync runs when the start page opens, on edits (debounced), and periodically while it's open. There is no background sync when no start page is open: that would need `alarms`, which isn't in the permissions contract (10). Leave it out unless users ask.

### 9.6 Quotas
Per-account record and storage limits, configurable on the server. Not needed for the first server release.

### 9.7 No server-served web view
A web page from the server that decrypts dials would let the operator ship passphrase-stealing code, which would break the zero-knowledge claim. Mobile access is out of scope for now.

## 10. Permissions contract (normative)

### Manifest
```json
"permissions": ["storage"],
"host_permissions": [],
"storage": { "managed_schema": "policy-schema.json" },
"optional_permissions": ["activeTab", "contextMenus", "favicon", "tabs"]
```
Also declared (not permissions): `chrome_url_overrides.newtab`, `action`, `commands`, `background.service_worker`, and the CSP from 8.3. If `activeTab` turns out not to work as an optional permission (see 15.2), move it to `permissions`. It shows no install warning or prompt, but treat that as a fallback.

### Optional permissions
| Permission | Enables | Chrome prompt? | Granted how |
|---|---|---|---|
| `activeTab` | Reading the invoked tab's URL, title, and icon URL when the user invokes the extension (4.4). Never used for screenshots or script injection. | No warning: Chrome grants it without a dialog | "Enable quick add" in setup |
| `contextMenus` | Right-click entries on tabs, pages, links, toolbar icon (4.4) | No warning: granted without a dialog | Same "Enable quick add" click |
| `favicon` | Site logos from the browser's icon cache (4.3) | Yes ("Read the icons of the websites you visit") | "Show site logos" in setup or when adding the first dial; letter tiles if declined |
| `tabs` | Save all tabs of a window (4.5) | Yes ("Read your browsing history") | Only when the user starts "save all tabs", after the explanation window (4.5); released afterwards by default; can be blocked by policy (10.1) |

### Rules
- Any permission not in this table is a defect. CI must compare the built manifest to this contract and fail on any change.
- Each optional permission gets a plain-language explanation before the request and a working "no thanks" path.
- Permissions screen in settings: what's granted, what it's used for, one-click revoke. When a permission is revoked externally (`permissions.onRemoved`), disable the dependent features cleanly; for `contextMenus`, remove the menus. Features disabled by policy show "managed by your organization" instead of an enable button.

### 10.1 Administrator policy

Two layers, which administrators can use separately or together, plus a third once sync exists.

**Layer 1: browser-enforced (no extension code needed).** In Chrome or Edge, an administrator sets the `ExtensionSettings` policy for this extension's ID with `"blocked_permissions": ["tabs"]`. The browser then refuses the permission, and the extension can't obtain it whatever its code does. This is the hard guarantee. The extension must handle the refused request gracefully (4.5, step 4). Document this recipe, with both store IDs, in the admin guide.

**Layer 2: managed policy (`chrome.storage.managed`).** The extension ships `policy-schema.json` (JSON Schema, top-level `object`, no `additionalProperties`). Administrators set values through Group Policy, registry, plist, or the Google Admin / Edge management consoles; values are visible at `chrome://policy`. Initial keys:

| Key | Type | Effect when set |
|---|---|---|
| `disableSaveAllTabs` | boolean | Hides every entry point for 4.5, never requests `tabs`, and immediately releases `tabs` if currently held. |
| `disableSiteLogos` | boolean | Never requests `favicon`; releases it if held; letter tiles only. |
| `serverUrl` | string (`https://` URL) | Phase 3. Sets the server (8.2) and locks the setting so users can't change it. Lets organizations point every install at their self-hosted server. |
| `disableSync` | boolean | Phase 3. Hides and disables sync (9). |
| `disableWeather` | boolean | Phase 3. Hides the weather dial type; no weather requests. |
| `disableServerLookup` | boolean | Phase 3. Hides and disables 4.10; no page addresses ever go to the server. |
| `orgEnrollment` | string | Phase 3, reserved: a token that binds the extension to an organization in the sync backend. |

Rules:
- Read policy at service-worker startup and on every start-page load; react live to `chrome.storage.onChanged` (area `"managed"`).
- Precedence: managed policy > organization policy from the backend (Phase 3, only when enrolled) > user settings. A user can never re-enable something policy disables.
- Unknown or invalid values: ignore them, log one console warning, keep the default. An invalid `serverUrl` (not `https://`, or `/info` fails) disables server features rather than falling back to the default server.
- Policy only removes capabilities or pins the server address. It can never add permissions or change the CSP.

**Layer 3 (Phase 3, optional): backend organization policy.** Once `orgEnrollment` is set, the sync backend may deliver the same keys (except `serverUrl`) as org policy. This alone is a soft control, so it's only honored for enrolled installs, which are configured by managed policy.

## 11. Non-functional requirements
- Start page renders from cache in under 300 ms; no network needed.
- No analytics or telemetry of any kind.
- Keyboard accessible; screen-reader labels; respects reduced-motion.
- All strings in a message catalog (start with English).
- No remote code; every script ships in the package. Reproducible builds.

## 12. Acceptance criteria

**Phase 1**
1. Installing from a packed build shows no permission dialog and no warnings; the manifest matches section 10 exactly (CI check passes).
2. With the network inspector open, using every Phase 1 feature makes no network requests, except the one-time icon fetch when adding from a tab (4.3, source 2). The build contains no server address.
3. Before "Enable quick add", no right-click entries exist and the start page still works fully.
4. After enabling it (no browser dialog appears): right-clicking a tab in the tab strip → "add" saves URL, title, and logo, in both Chrome (148+) and Edge.
5. With `favicon` granted, a dial added by typing the URL of a previously visited site shows that site's logo immediately, and the network inspector shows no request for it.
6. With `favicon` declined, all dials show letter tiles (or the add-time tab icon), and nothing else breaks. Revoking `favicon` later keeps already-cached logos.
7. "Save all tabs" from the toolbar right-click menu opens the explanation window; Continue shows Chrome's dialog; after granting, the preview lists the source window's tabs (not the explanation window); Save creates the multi-dial and the Permissions screen shows `tabs` released. Declining saves nothing. Closing the window mid-flow also releases `tabs`.
8. With managed policy `disableSaveAllTabs: true` (set via `chrome://policy` test setup or registry), no entry point is visible and `tabs` is never requested; setting it while `tabs` is held releases it immediately.
9. With `ExtensionSettings` → `blocked_permissions: ["tabs"]`, the extension still installs and runs, and "save all tabs" fails gracefully at step 4.
10. Add 20 dials, reorder, restart: order persists. Export → import (replace) into a fresh profile reproduces everything.
11. Selection mode: select 30 dials with Shift-click and Ctrl/Cmd-click, move them to another group via "move to…": they appear at the end of the target group in the same relative order and are gone from the source. Undo restores both groups exactly. Moving to "new group…" creates the group. Dragging the selection onto a group-selector entry does the same.
12. World clock: three IANA zones plus one custom zone (from a POSIX string) display correctly with the network disabled, and the preview shows correct next transitions.

**Phase 3**
13. With the default server, the network inspector shows the extension contacting only that server (plus the add-time tab icon fetch). After switching to a self-hosted server, requests go only to the self-hosted server and none to the default one.
14. With managed policy `serverUrl` set, the server setting is locked to that address; with an unreachable or invalid `serverUrl`, server features fail clearly and nothing goes to the default server.
15. A self-hosted server with weather turned off: `/info` reports it, and the weather dial type is hidden.
16. Server lookup is off after install; with it off, the network inspector shows no `/meta` requests even when adding typed-URL dials. Turning it on shows the disclosure (naming the configured server) first; afterwards, adding `https://example.com/page?token=abc#x` sends only `https://example.com/page`. Adding `http://intranet/` or `http://192.168.1.10/` sends nothing.
17. A server database dump contains no readable URLs, titles, or images, and its auth hashes can't be checked against passphrase guesses without the secret key.
18. Signing in on a second device requires the secret key and the passphrase; a wrong passphrase or a mistyped secret key fails cleanly (a checksum error for typos, before key derivation runs); the right pair reproduces all dials. The recovery key alone restores access and forces a new passphrase.
19. The secret key differs between two setups given the identical typed seed.
20. Weather works with no account and no geolocation prompt.

## 13. Clean-room constraints
- Do not copy another product's name, logo, icons, colors, layout, tile styling, menu labels, or other user-facing text.
- Do not use another product's source code, bundles, or assets, and do not decompile or inspect them.
- Do not replicate its file formats or backup file names.
- Where this spec is silent, make your own original design decisions.

## 14. Phasing
- **Phase 1 (completely local, no server):** section 4 except 4.10 (including tab-strip add, logos, save-all-tabs, multi-select and mass move, search box, local backup), link, multi-link, and world-clock dials, fixed and dynamic layouts, permissions screen, managed policy keys that apply to Phase 1, manifest and network-module CI checks.
- **Phase 2 (still local, no server):** search dials, tags, group folders, focus mode, free grid.
- **Phase 3 (server):** the server program and default hosted instance, self-hosting guide and container image (8), server setting and `serverUrl` policy, weather proxy and place search, weather dial, title/logo lookup (4.10), E2EE sync (9). Organization accounts follow after individual sync works.

## 15. Verify first (build a throwaway prototype before Phase 1)
These behaviors are not fully specified in Chrome's documentation. Confirm them in current Chrome and Edge before relying on them:
1. Clicking a `"tab"`-context menu item on a **background** tab: is `activeTab` granted for that clicked tab (so its URL/title are readable), or only for the active one? If neither, use `info.pageUrl` if provided; otherwise ask the user to switch to that tab first.
2. `activeTab` and `contextMenus` work when declared in `optional_permissions` and are granted through `permissions.request` without a dialog.
3. `_favicon` behavior: works from the new-tab page and the service worker, works in Edge, never touches the network, which sizes return real data, and how to detect the generic placeholder icon reliably.
4. Whether `activeTab` lets the extension `fetch()` the tab's `favIconUrl` when the icon is on the tab's own origin (source 2 in 4.3). If it rarely works, drop the fetch and keep only `data:` icons; then Phase 1 makes no network requests at all, and `connect-src` can be `'none'` until Phase 3.
5. **Critical:** with `ExtensionSettings` → `blocked_permissions: ["tabs"]`, does Chrome/Edge only refuse the optional request, or does it disable the whole extension because `tabs` is listed in `optional_permissions`? How does `permissions.request` report the refusal (resolves `false` or throws)? If the extension gets disabled, the admin guide must recommend Layer 2 only.
6. `chrome.storage.managed` works with `storage` declared and is readable from both the service worker and extension pages.
7. PBKDF2 at 2,000,000 iterations through WebCrypto: measure time in Chrome and Edge on a low-end laptop. If it's well over 3 seconds there, lower the default, but never below 1,000,000.
