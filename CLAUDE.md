# Cookie Shop - architecture notes for Claude

Single-file HTML app (index.html). No build step, no external JS deps.
Fonts only from Google Fonts CDN. Deployed via GitHub Pages.

## Storage
- `cookie-data-v1` (localStorage): full synced state {version, products, orders, trash, notes, settings}
- `cookie-sync-cfg-v1`: LOCAL-ONLY Firebase URL + store key (never synced, on purpose)
- `cookie-client-id`: random per-device id used for echo suppression
- `cookie-dirty-v1`: pending sync paths, survives reload so offline edits flush later
- `migrate()` deep-fills every missing field; any shape of garbage must load without crashing

## Sync (Firebase RTDB, no SDK)
- Base path: `{url}/stores/{storeKey}` - the store key IS the security (unguessable path)
- Writes: debounced (600ms) `fetch PUT {base}/{path}.json` per dirty path
  - paths: `products`, `settings`, `notes`, `trash`, `orders/{id}` (PUT null = delete)
- Reads: `EventSource({base}.json)` - handles `put` and `patch` events
- Conflict rules: every synced object carries `_c` (writer clientId) + timestamp
  (`_t` for singletons, `updatedAt` for orders). Newer wins per entity.
  Incoming with `_c === clientId` is our own echo -> ignored.
- Deletions use tombstones in `trash{id: deletedAt}` so an offline device
  cannot resurrect a deleted order (unless its edit is NEWER than the delete).
- Draft order (`draft` var) is per-device and never synced until saved.

## Business rules
- HST default 13% (editable in settings), delivery ALWAYS free (hardcoded FREE line on invoice)
- Payment plans: prepaid / cod-cash / cod-et. `paid` is a separate flag.
- Driver marking delivered with method != Prepaid and collected > 0 auto-sets paid=true.
- Statuses: 0 Ordered, 1 In Production, 2 Out for Delivery (= driver queue), 3 Delivered.
- Delivered quantities default to ordered qty; `order.delivery.qty[lineId]` only set when punched.

## Printing
- `#printArea` + `@media print` hides everything else. `printInvoice()` / `printReceipt()`
  build HTML, call `window.print()`. "Send receipt" uses `navigator.share` (text) with
  clipboard fallback - no PDF library, browser print dialog IS the PDF generator.

## Known landmines
- NEVER use alert/confirm/prompt (blocked in sandboxed previews). Toasts + inline confirms only.
- ASCII only inside JS string literals (multiplication sign, curly quotes = SyntaxError risk).
  HTML entities (&#127850;) are used in markup/strings instead of raw emoji in JS.
- Firebase rules must allow read/write on `/stores` or EventSource silently fails
  (onerror fires, chip shows "Reconnecting..." forever). Check rules first when sync "hangs".
- EventSource cannot send custom headers - only works because RTDB streams on Accept header.
  Do not "upgrade" to auth tokens in headers; use `?auth=` query param if ever needed.
- Two devices editing the SAME order simultaneously: whole-order last-write-wins.
  Fine for two people; do not promise field-level merging.
- Input fields are only re-rendered when NOT focused (see renderOrderBuilder) - a remote
  sync mid-typing must not clobber the active field. Preserve this pattern in new inputs.
- jsdom smoke tests: `node smoke.js` (strip font <link> tags first - already handled).

## Session ritual before every push
1. Extract <script> block, `node --check`
2. `node smoke.js` (jsdom, 30 assertions)
3. Push via GitHub API, tag version
