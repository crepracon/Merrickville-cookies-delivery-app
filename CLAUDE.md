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

## Phomemo M832 printing (v1.1.0)
- "Phomemo receipt" button renders the receipt to a 1700px-wide canvas PNG
  (receiptCanvas) and shares it via navigator.share files (sharePhomemo).
  The Phomemo app accepts shared images and prints them - this skips the
  save-PDF-then-upload dance. Fallback when file-share unsupported: downloads
  the PNG and toasts to open it in the Phomemo app.
- The M832 is NOT AirPrint/Mopria: it never appears in OS print dialogs.
  Do not chase that; image-share to the Phomemo app is the supported path.
- Canvas uses Arial/sans-serif on purpose - web fonts may not be loaded when
  drawing and canvas does not wait for them.
- jsdom has no 2d canvas: receiptCanvas returns null there; smoke tests only
  assert wiring + graceful degradation. Manual visual check needed on print changes.
- Direct Web Bluetooth printing: parked. Android-Chrome-only, M832 BLE protocol
  undocumented, would need reverse-engineering against the physical printer.

## v1.2.0
- Phomemo path now covers INVOICES too: "Phomemo invoice" button on New Order
  tab (uses draft) + "Phomemo" action on saved order cards -> invoiceCanvas ->
  shareCanvasPng. shareCanvasPng is the single share/download helper for all
  canvas documents.
- LANDMINE (hit during this change): replacing a function opening line with a
  multi-function block orphans the old body below it. After any str_replace on
  a function signature, grep for the old body and remove it, then node --check.

## v1.3.0 - compact Phomemo layouts
- receiptCanvas/invoiceCanvas rewritten compact: same 1700px width (8.5in at
  200dpi) but height is content-driven, min 900/1000px (~4.5-5in). On roll
  paper the M832 only feeds what the image needs, roughly halving paper use.
  The window.print() paths are untouched and still full-page.
- Phomemo app friction (printer picker, extra taps) is the Phomemo app's own
  UX - not reachable from a web page. Do not promise fixing it.

## v1.4.0 - Merrickville Cookies rebrand
- White + blue scheme. NOTE: CSS var names still say --gold but hold BLUE
  values (#2166CB family) - renaming vars app-wide was riskier than the
  mismatch. Landmine for future edits: --gold means "brand accent", not gold.
- migrate() auto-upgrades business name 'Cookie Shop' -> 'Merrickville
  Cookies' so existing devices/Firebase pick up the rename without manual edit.
- Placeholder cookie logo: .logo (header), .p-logo (print), drawCookieLogo()
  (canvas - drawn shapes, prints black on thermal). When the real logo image
  arrives: swap header/.p-logo content for an <img>/background, and replace
  drawCookieLogo with drawImage of a preloaded Image (beware: canvas needs the
  image loaded BEFORE drawing - use img.onload or decode()).

## v1.5.0 - real catalog + pickup dropdowns
- Catalog model: size field now means CATEGORY (4" cookies, 7" cookies,
  Cookie dough, Buttertarts, Fudge); flavour = variant. 17 seeded items,
  cost 3.50, PRICE 0 (owner fills sell prices in Products tab).
- products.catalogV=2 flag: migrate() wipes any pre-v2 catalog and reseeds,
  then boot pushes it (catalogUpgraded flag). Bump catalogV for any future
  forced reseed - do NOT reuse 2.
- Pickups: {id, productId, desc, qty}; rendered as a catalog <select>
  (data-pusel). Legacy free-text pickups survive as a '__keep' option.
  Receipts keep reading pu.desc, so nothing downstream changed.

## v1.5.1
- catalogV=3: price 3.50 / cost 2.25. v2->v3 is an IN-PLACE field update
  (preserves owner edits and added products); pre-v2 reseeds directly to v3.

## Session docs
- STATUS.md = product scope, decisions, roadmap. Update BOTH files every push.
