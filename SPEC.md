# Monday Lane — Commercial Control design

Date: 2026-09-15  
Product: **Commercial Control (Monday Lane)** — Roads Authority Namibia, Oshakati  
Not: UltraPavement / PM Suite

## 1. Purpose

Monday returns are scanned and uploaded to SharePoint. Programmes are monthly. Payment certificates come later. Today a smudged qty can be paid because nothing parks it.

Monday Lane is the RA commercial control board:

- You only file on SharePoint (`3406 District Administration`). The board watches. There is no upload UI.
- OCR extracts quantities. Confidence below **0.82** goes to a **human verify queue**. Queued qty is not in the month total, not in the forecast, not payable.
- Confirmed actual is tracked against the monthly programme every week (example: 439 m² potholes programmed).
- A Monday morning is: open the board, see programmed / confirmed / queued / forecast, clear the queue, read the exception sheet, get a mail nudge if a packet landed and lines are waiting.

## 2. Two products (do not merge)

| Product | Whose books | This spec |
|---|---|---|
| **Commercial Control (Monday Lane)** | Roads Authority — Oshakati SharePoint | In |
| **PM Suite / UltraPavement** | Other standalone projects (e.g. C43) | Out |

## 3. Slices

**Slice 1:** watcher, OCR, queue, programme recon, weekly forecast, exception sheet, mail nudge.

**Slice 2:** join payment certificates onto the same line. `certified_no_actual` becomes a hold.

## 4. Architecture

SharePoint 3406 → Watcher (list + download only) → OCR → confirmed or verify queue → Recon → lane / exception sheet / mail nudge.

Confidence hold **0.82**. Month total = confirmed only.

Watcher site: `networkmaintenance-OshakatiMaintenanceRegion` under `3406 District Administration / Oshakati District / Projects and Operations/`.

Full spec is also `SPEC.md` in this repo; implementation plans follow as PLAN-SLICE-1.md and PLAN-FULL.md.
