# Monday Lane — Commercial Control design

Date: 2026-09-15  
Status: draft for user review  
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

They may share *ideas* later (queue, recon, hold). They do not share a database, UI, or recon engine.

The preview already showing “Monday Lane” is a **synthetic demo** of this product. It is not PM Suite.

## 3. Slices

**Slice 1 (first implementation plan):** watcher, OCR, queue, programme recon, weekly forecast, exception sheet, mail nudge.

**Slice 2 (designed here, not in the first plan):** join payment certificates onto the same line. `certified_no_actual` becomes a hold.

Do not build slice 2 in the first plan.

## 4. Architecture

```
RA SharePoint 3406
  Monday scans + monthly programme
           │
           ▼
        Watcher  (list + download only; never write/tidy)
           │
           ▼
          OCR  (line items + confidence)
           │
     ┌─────┴──────┐
     │ ≥ 0.82     │ < 0.82
     ▼            ▼
 confirmed     verify queue  ← you confirm / edit qty / reject
     │            │
     └─────┬──────┘
           ▼
        Recon  (planned × confirmed actual)
           │
     ┌─────┼──────────────┐
     ▼     ▼              ▼
   lane  exception      mail nudge
         sheet
```

SharePoint is the filing cabinet. Monday Lane is the board. Certificates occupy an empty `certified` field until slice 2.

**Watcher target (slice 1):** site `networkmaintenance-OshakatiMaintenanceRegion`, library path under `3406 District Administration / Oshakati District / Projects and Operations/`. Exact subfolder names for “this month’s programme” and “Monday return packets” are **configuration**, not inferred from filenames. If the configured folder is missing, that is a watch failure (packet stays `expected`), not a guess.

## 5. Components

| Unit | Does | Does not |
|---|---|---|
| **Watcher** | Lists `3406` for new Monday packets and the month programme. Downloads. | Upload, rename, move, or tidy SharePoint. |
| **OCR** | Scan → line items + confidence. | Guess a pay item it is not sure of. |
| **Queue** | Holds confidence < 0.82. Human confirm / edit qty / reject. | Put queued qty into totals. |
| **Recon** | Confirmed actual vs programme. Weekly burn, month-end forecast, findings. | Treat a certificate as actual (slice 2). |
| **Board** | Month lane, queue, exception sheet, mail nudge. | PM Suite, other projects, drag-drop upload. |

**Monday loop:** files land on SharePoint → watcher → OCR → confirmed or queue → you clear the queue → lane and exception sheet update → mail if anything is waiting.

## 6. Data

Three records:

### Programme line

From the monthly programme on SharePoint: pay item, activity, unit (`m2` \| `m3` \| `km` \| `no`), planned qty, contractor, road, month.

### Packet

One Monday file set: Monday date, SharePoint path, state `expected` \| `ingested` \| `failed`.

### OCR row

One extracted quantity: packet, programme line **or** `unplanned`, qty, confidence, status `queued` \| `confirmed` \| `rejected`, source path.

**Month total = sum of `confirmed` only.**

**Forecast (slice 1):** `(confirmed so far ÷ Mondays ingested) × Mondays in the month`.

If that activity still has queued qty, official status is **hold**. A second number “if you confirm the queue” may be shown. It is not the official forecast.

**Findings (slice 1):**

- `queued_not_in_total` — waiting on the human
- `forecast_over_planned` — official forecast > programme × 1.08
- `forecast_under_planned` — official forecast < programme × 0.82, after at least two Mondays
- `actual_no_planned` — OCR with no programme line. Slice 1: it stays queued until you **reject** it. There is no “confirm onto a pay item” action.

**Slice 2 adds** `certified` on the programme line. Finding `certified_no_actual` is then a hold.

**Mail nudge** is not a fourth record. Send when a packet ingests and the queue is non-empty.

## 7. Worked example (must hold in tests)

September 2026, four Mondays. Pothole patching programmed **439 m²**.

| Week | Confirmed | Queued |
|---|---|---|
| 7 Sep | 92 | — |
| 14 Sep | 118 | 47 (smudge) |

After two Mondays, official: confirmed 210, forecast 420, status **hold** because 47 is queued.  
If the 47 is confirmed: confirmed 257, forecast 514, status **over**.

Queued 47 is never in 210 and never in 420.

## 8. Failures (nothing silent)

| Failure | Board |
|---|---|
| Watch fails or lists 0 | Packet stays `expected`. Banner: not ingested. No invented qty. |
| Scan unreadable / OCR crash | Packet `failed`. Zero rows. Mail: packet landed, OCR failed. |
| Confidence < 0.82 | Queue. Not in total, forecast, or pay. |
| No programme for the month | Ingest still runs. Every line `actual_no_planned` / queued. |
| Duplicate SharePoint path | Skip. New file in that week = new packet. |
| Queue never cleared | Hold remains. Totals do not include the guess. Mail may nudge again. |
| Mail send fails | Lane and queue still update. Nudge marked failed. Mail is not the books. |

HTTP 200, a pretty OCR number, or an empty listing is not “done”. Confirmed qty is the only qty that counts.

## 9. Testing (slice 1)

Must pass:

1. Queued qty is absent from confirmed, forecast, and remaining.
2. Confirm / edit / reject recalculate the lane.
3. 439 programmed, 92+118 confirmed after 2 Mondays → forecast 420; hold if 47 queued.
4. Confirming 47 → forecast 514 → over.
5. Unplanned OCR cannot be confirmed onto a pay item.
6. Duplicate SharePoint path is not ingested twice.
7. Failed OCR → packet `failed`, zero rows.

Not in slice 1 tests: live SharePoint SSO, Tesseract accuracy, Gmail delivery, PM Suite, certificates.

## 10. Out of scope (this spec)

- PM Suite / UltraPavement integration
- Drag-drop upload into the app
- Auto-pay, auto-confirm, or treating OCR as certified
- SharePoint folder create / move / tidy
- Slice 2 certificate join (specified above, not implemented in the first plan)
- National vs regional books, CPA fuel/CPI (existing RA audit engines; not this board)

## 11. Success (Monday morning)

You open Commercial Control and, without uploading anything to the app:

1. This week’s packet is ingested or clearly `failed`/`expected`.
2. Each programme activity shows programmed / confirmed / queued / forecast.
3. The queue lists every low-confidence line; none of those quantities sit in the totals.
4. The exception sheet lists holds, over/under, and unplanned work.
5. Mail was attempted if a packet ingested and the queue is non-empty.
