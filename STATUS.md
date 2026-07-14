# Merrickville Cookies Delivery - project status

> For Claude: read this + CLAUDE.md at the start of every session, and UPDATE
> BOTH with every push. This file is product scope; CLAUDE.md is technical
> architecture and landmines.

**Live app:** https://crepracon.github.io/Merrickville-cookies-delivery-app/
**Repo:** crepracon/Merrickville-cookies-delivery-app (GitHub Pages, main branch)
**Current version:** v1.5.1 (2026-07-14)

## What this is
Single-file web app for a small cookie business (Merrickville Cookies).
Two users: the owner (builds orders, invoices, tracks status) and a delivery
driver (runs stops on a phone, punches in drop-offs/pickups, collects payment,
prints receipts). Live sync between their devices via Firebase.

## Business rules (decided, do not change without owner)
- All prices CAD; HST 13% on all invoices (rate editable in settings)
- Delivery is ALWAYS free - no delivery fee anywhere
- Payment: prepaid, cash on delivery, or e-transfer on delivery; tracked per
  order, never assumed; separate paid/unpaid flag
- Catalog: categories 4" cookies / 7" cookies / Cookie dough / Buttertarts
  (variants: Chocolate Chip, Cookies and Cream, Garbage Can, Caramel) and
  Fudge (Chocolate Mint only). Sell $3.50 / cost $2.25 per unit.
- Branding: "Merrickville Cookies", white with blue highlights, cookie logo
  (currently a drawn placeholder - real logo image still to come from owner)

## Feature state
DONE
- Products tab: editable catalog (flavour/category/cost/price), synced
- New Order: customer info, line items, discounts, HST, save, invoice print
- Orders tab: search, status flow (Ordered > In Production > Out for
  Delivery > Delivered), paid flag, open/duplicate/delete, JSON backup
- Delivery tab: stops from "Out for Delivery" orders, per-item delivered
  quantities, pickups (catalog dropdown), payment collection, delivery notes,
  mark delivered, map + phone links
- Live sync: Firebase RTDB, instant via EventSource, offline queue, tombstoned
  deletes; store key acts as shared PIN
- Printing: browser print/PDF (full page) AND Phomemo M832 path - canvas PNG
  via share sheet, compact half-height layouts for invoice + delivery receipt

PARKED / NEXT
- Real logo image (waiting on owner) - swap points documented in CLAUDE.md
- Direct Web Bluetooth printing to M832: parked; Android-Chrome-only and needs
  protocol reverse-engineering. Revisit only if share-sheet flow annoys driver.
- Driver phone OS still unknown (affects Bluetooth feasibility)
- Possible two-step product picker (category then flavour) if catalog grows
- Consider periodic JSON backups habit (Orders > Backup > Export)

## Environment facts
- Firebase RTDB: https://merrickville-cookies-delivery-default-rtdb.firebaseio.com
  Rules allow read/write under /stores/$store only. Store key = shared PIN,
  entered in app settings on each device (never committed to the repo).
- Owner deploys nothing manually: Claude pushes via GitHub API with a
  fine-grained token the owner creates per session (Contents read/write).
  GitHub Pages serves main branch root.
- Printer: Phomemo M832 thermal, 8.5" paper, prints via Phomemo app only
  (never appears in OS print dialogs - not AirPrint/Mopria).

## Working style (owner preference)
- Batch multiple changes per message; structured options over open questions
- Brief plain-language explanations when new tech concepts come up
- Iterative loop: owner tests live, reports, Claude diagnoses and patches
- Every push: ritual in CLAUDE.md (syntax check, smoke tests, push, tag)
