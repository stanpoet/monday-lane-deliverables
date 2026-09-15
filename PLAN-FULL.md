# Commercial Control (Monday Lane) — Full Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the whole Commercial Control board: honest recon (queued OCR never paid), SharePoint-only ingest, human verify queue, exception sheet, mail nudge, then payment-certificate join (`certified_no_actual` = hold).

**Architecture:** One product (RA Commercial Control). Not UltraPavement / PM Suite. Pure engines (`recon`, `ingest`, `queue`, `watch`, `ocr`, `nudge`, `certs`) behind ports. Fixture adapters run in this preview; live SharePoint / Tesseract / Gmail are extra adapters on the same ports. Slice 2 adds `certified` on the same programme line — no second app.

**Tech Stack:** React 19, TanStack Start, Zustand, node:test (`node --experimental-strip-types --test`). SharePoint Path D (`sp-rest.mjs`) only as a `WatcherPort` adapter. Gmail only as a `NudgePort` adapter.

**Spec:** `docs/superpowers/specs/2026-09-15-monday-lane-commercial-control-design.md`

**Depends on:** complete `docs/superpowers/plans/2026-09-15-monday-lane-slice-1.md` first (Phase A, Tasks 1–6). This file is Phase B–E (everything the spec named that slice 1 deferred).

## Global Constraints

- UI name: **Commercial Control**. Never UltraPavement or PM Suite.
- Confidence hold **0.82**. Over **1.08**. Under **0.82** after ≥ 2 ingested Mondays.
- Month total = `confirmed` only. Queued qty is not in actual, forecast, remaining, or pay.
- No upload UI. Files enter only via Watcher.
- Packet states: `expected` | `ingested` | `failed`.
- Unplanned OCR: reject or leave queued. No confirm onto a pay item.
- Watcher never writes/renames/tidies SharePoint.
- HTTP 200, pretty OCR, or empty listing is not done.
- Skip `git commit` steps if `.git` is missing.
- Tests: `node --experimental-strip-types --test src/lib/monday/*.test.ts`
- Do not start Phase B until Phase A unit tests are green.

## File structure (full product)

| File | Responsibility |
|---|---|
| `src/lib/monday/types.ts` | Domain types + bands |
| `src/lib/monday/recon.ts` | planned × confirmed (+ certified in Phase E) |
| `src/lib/monday/ingest.ts` | Packet apply, duplicate path, failed OCR |
| `src/lib/monday/queue.ts` | confirm/reject |
| `src/lib/monday/watch.ts` | `WatcherPort` + `runWatch` |
| `src/lib/monday/watch-sharepoint.ts` | Path D listing adapter |
| `src/lib/monday/ocr.ts` | `OcrPort` — bytes → extracted lines + confidence |
| `src/lib/monday/nudge.ts` | `maybeNudge` |
| `src/lib/monday/nudge-gmail.ts` | Gmail adapter (server-safe payload) |
| `src/lib/monday/certs.ts` | Load certified qty onto programme lines |
| `src/lib/monday/config.ts` | Folder paths (not inferred) |
| `src/lib/monday/store.ts` | Zustand wiring |
| `src/lib/monday/*.test.ts` | node:test |
| `src/components/monday/*` | Board only |

---

# Phase A — Slice 1 (already planned)

- [ ] **Step 0: Execute slice 1 in order**

Open `docs/superpowers/plans/2026-09-15-monday-lane-slice-1.md`. Complete Tasks 1–6 (recon, ingest, queue, watcher fixture, nudge, board). Do not edit this full plan’s later tasks to “get it working” by weakening tests.

Done when:

```
node --experimental-strip-types --test src/lib/monday/*.test.ts
```

is green and `node scripts/browser-smoke.mjs` body contains `Commercial Control` and does not contain `Upload`.

---

# Phase B — OCR port

### Task 7: OCR port (no guessed pay item)

**Files:**
- Create: `src/lib/monday/ocr.ts`
- Create: `src/lib/monday/ocr.test.ts`

**Interfaces:**
- Consumes: `OcrRow` fields except `status`; `CONFIDENCE_HOLD`
- Produces:
  - `export type ExtractedLine = { activityId: string | "unplanned"; qty: number; confidence: number; ocrText: string }`
  - `export type OcrResult = { ok: true; lines: ExtractedLine[] } | { ok: false; reason: "unreadable" }`
  - `export type OcrPort = { extract(args: { bytes: Uint8Array; scanName: string; programmeIds: string[] }): Promise<OcrResult> }`
  - `export function toPacketExtracted(packetId: string, lines: ExtractedLine[]): Omit<import("./types.ts").OcrRow, "status">[] | "ocr-failed"` helper used by watch: if `OcrResult.ok === false`, callers pass `"ocr-failed"` into `ingestPacket`.

Rules:
- If a line’s `activityId` is not in `programmeIds`, it must be `"unplanned"` — the port must not pick a nearby pay item.
- Confidence is required on every line (0–1).
- Fixture port for tests: parse a tiny JSON UTF-8 payload `{ lines: ExtractedLine[] }` or `{ unreadable: true }`. Real Tesseract is Task 8.

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { createJsonOcr, toPacketExtracted } from "./ocr.ts";

const programmeIds = ["pothole", "blading"];

describe("createJsonOcr", () => {
  it("marks unknown activity unplanned instead of guessing", async () => {
    const ocr = createJsonOcr();
    const bytes = new TextEncoder().encode(
      JSON.stringify({
        lines: [{ activityId: "sand", qty: 14, confidence: 0.9, ocrText: "sand 14" }],
      }),
    );
    const result = await ocr.extract({ bytes, scanName: "w2.pdf", programmeIds });
    assert.equal(result.ok, true);
    if (result.ok) {
      assert.equal(result.lines[0].activityId, "unplanned");
    }
  });

  it("returns unreadable", async () => {
    const ocr = createJsonOcr();
    const bytes = new TextEncoder().encode(JSON.stringify({ unreadable: true }));
    const result = await ocr.extract({ bytes, scanName: "blur.pdf", programmeIds });
    assert.equal(result.ok, false);
    if (!result.ok) assert.equal(result.reason, "unreadable");
  });

  it("toPacketExtracted maps unreadable to ocr-failed", () => {
    assert.equal(toPacketExtracted("w3", { ok: false, reason: "unreadable" }), "ocr-failed");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --experimental-strip-types --test src/lib/monday/ocr.test.ts`

Expected: FAIL (`Cannot find module`)

- [ ] **Step 3: Write minimal implementation** of `createJsonOcr` and `toPacketExtracted`. If `activityId` not in `programmeIds`, emit `"unplanned"`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/ocr.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/ocr.ts src/lib/monday/ocr.test.ts
git commit -m "feat: OCR port never guesses a pay item"
```

Skip if no `.git`.

---

### Task 8: Wire OCR into watch (bytes from listing)

**Files:**
- Modify: `src/lib/monday/watch.ts` — files may carry `bytes` instead of pre-extracted lines
- Create: `src/lib/monday/watch-ocr.test.ts`

**Interfaces:**
- Consumes: `OcrPort`, `runWatch` / `ingestPacket`, `toPacketExtracted`
- Produces:
  - `export type WatchFileRaw = { sharePointPath: string; scanName: string; monday: string; weekNo: number; bytes: Uint8Array }`
  - `export async function runWatchWithOcr(books: Books, files: WatchFileRaw[], ocr: OcrPort, programmeIds: string[]): Promise<Books>`

For each file: `ocr.extract` → `toPacketExtracted` → `ingestPacket`. Keep existing `runWatch` for pre-extracted fixture files. Do not remove Task 4 tests.

- [ ] **Step 1: Write the failing test**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { runWatchWithOcr } from "./watch.ts";
import { createJsonOcr } from "./ocr.ts";
import type { MondayPacket } from "./types.ts";

const packet: MondayPacket = {
  id: "w3",
  weekNo: 3,
  monday: "2026-09-21",
  label: "21 Sep",
  status: "expected",
  scanName: "W3.pdf",
  sharePointPath: "/3406/returns/2026-09-21/W3.pdf",
};

describe("runWatchWithOcr", () => {
  it("failed OCR marks packet failed with zero rows", async () => {
    const bytes = new TextEncoder().encode(JSON.stringify({ unreadable: true }));
    const out = await runWatchWithOcr(
      { packets: [packet], rows: [] },
      [{ sharePointPath: packet.sharePointPath, scanName: packet.scanName, monday: packet.monday, weekNo: 3, bytes }],
      createJsonOcr(),
      ["pothole"],
    );
    assert.equal(out.packets[0].status, "failed");
    assert.equal(out.rows.length, 0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --experimental-strip-types --test src/lib/monday/watch-ocr.test.ts`

Expected: FAIL (`runWatchWithOcr` not exported)

- [ ] **Step 3: Implement `runWatchWithOcr`** in `watch.ts` using `ingestPacket` + `toPacketExtracted`. Map `activityId: "unplanned"` to OCR row `activityId: "unplanned"` (string).

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/watch-ocr.test.ts src/lib/monday/watch.test.ts src/lib/monday/ingest.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/watch.ts src/lib/monday/watch-ocr.test.ts src/lib/monday/ocr.ts
git commit -m "feat: watch runs OCR port per file"
```

Skip if no `.git`.

---

# Phase C — Live SharePoint adapter

### Task 9: Configured folders (never inferred)

**Files:**
- Create: `src/lib/monday/config.ts`
- Create: `src/lib/monday/config.test.ts`

**Interfaces:**
- Produces:
  - `export type SpConfig = { site: string; programmeFolder: string; returnsFolder: string }`
  - `export const DEFAULT_SP_CONFIG: SpConfig` with:
    - `site`: `networkmaintenance-OshakatiMaintenanceRegion`
    - `programmeFolder`: `3406 District Administration/Oshakati District/Projects and Operations`
    - `returnsFolder`: `3406 District Administration/Oshakati District/Projects and Operations`
  - `export function missingFolder(listingOk: boolean, itemCount: number): "missing-folder" | "empty" | null` — `listingOk === false` ⇒ `"missing-folder"`; `itemCount === 0` ⇒ `"empty"`; else `null`.

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { DEFAULT_SP_CONFIG, missingFolder } from "./config.ts";

describe("DEFAULT_SP_CONFIG", () => {
  it("points at Oshakati 3406, not /sites/Oshakati", () => {
    assert.equal(DEFAULT_SP_CONFIG.site, "networkmaintenance-OshakatiMaintenanceRegion");
    assert.match(DEFAULT_SP_CONFIG.programmeFolder, /3406 District Administration/);
    assert.doesNotMatch(DEFAULT_SP_CONFIG.site, /^Oshakati$/);
  });
});

describe("missingFolder", () => {
  it("treats failed list as missing-folder", () => {
    assert.equal(missingFolder(false, 0), "missing-folder");
  });
  it("treats zero items as empty", () => {
    assert.equal(missingFolder(true, 0), "empty");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --experimental-strip-types --test src/lib/monday/config.test.ts`

Expected: FAIL (module missing)

- [ ] **Step 3: Write `config.ts`**

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/config.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/config.ts src/lib/monday/config.test.ts
git commit -m "feat: SharePoint folders are configuration not guesses"
```

Skip if no `.git`.

---

### Task 10: SharePoint listing adapter (read-only)

**Files:**
- Create: `src/lib/monday/watch-sharepoint.ts`
- Create: `src/lib/monday/watch-sharepoint.test.ts`

**Interfaces:**
- Consumes: `WatcherPort`, `ListingResult`, `missingFolder`, `DEFAULT_SP_CONFIG`
- Produces:
  - `export type ListFolderFn = (path: string) => Promise<{ ok: boolean; files: { path: string; name: string }[] }>`
  - `export function createSharePointWatcher(listFolder: ListFolderFn, folder: string): WatcherPort`

`WatcherPort.list` calls `listFolder(folder)`. If `!ok` return `{ ok: false, reason: "missing-folder" }`. If `ok` and `files.length === 0` return `{ ok: false, reason: "empty" }`. Otherwise `{ ok: true, files: mapped WatchFile with extracted: [] }` — this adapter lists only; OCR bytes are Task 8’s `runWatchWithOcr`. For listing-only `WatcherPort` used by `runWatch`, `extracted` may be empty arrays (zero qty) — **do not use this adapter with `runWatch` until bytes exist**. Prefer returning `{ ok: true, files }` only for `createSharePointListing` used by a new function:

  - `export async function listReturns(listFolder: ListFolderFn, folder: string): Promise<ListingResult & { raw?: { path: string; name: string }[] }>`

Keep `WatcherPort` for fixture. SharePoint adapter implements **listing**, not ingest of invented OCR.

Hard rule in code comment: never POST/PUT/MOVE/delete.

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { listReturns } from "./watch-sharepoint.ts";

describe("listReturns", () => {
  it("unavailable list is missing-folder", async () => {
    const result = await listReturns(async () => ({ ok: false, files: [] }), "3406 District Administration/Oshakati District/Projects and Operations");
    assert.equal(result.ok, false);
    if (!result.ok) assert.equal(result.reason, "missing-folder");
  });

  it("zero files is empty, not invented packets", async () => {
    const result = await listReturns(async () => ({ ok: true, files: [] }), "3406 District Administration/Oshakati District/Projects and Operations");
    assert.equal(result.ok, false);
    if (!result.ok) assert.equal(result.reason, "empty");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --experimental-strip-types --test src/lib/monday/watch-sharepoint.test.ts`

Expected: FAIL (module missing)

- [ ] **Step 3: Implement `listReturns`**

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/watch-sharepoint.test.ts src/lib/monday/watch.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/watch-sharepoint.ts src/lib/monday/watch-sharepoint.test.ts
git commit -m "feat: SharePoint listing adapter fails closed on empty"
```

Skip if no `.git`.

**Live Edge SSO:** not required to pass tests. A later operator may pass `listFolder` that shells to `scripts/sp-rest.mjs`. If Edge is missing, keep using `createFixtureWatcher`. Do not fake a successful live list.

---

# Phase D — Gmail nudge adapter

### Task 11: Gmail port (payload only)

**Files:**
- Create: `src/lib/monday/nudge-gmail.ts`
- Create: `src/lib/monday/nudge-gmail.test.ts`

**Interfaces:**
- Consumes: `NudgePort`, `Nudge` from `nudge.ts`
- Produces:
  - `export type MailSender = (args: { subject: string; body: string }) => Promise<"sent" | "failed">`
  - `export function createGmailNudgePort(send: MailSender): NudgePort`
  - `export function nudgeCopy(nudge: Omit<Nudge, "status">): { subject: string; body: string }`

Subject: `Commercial Control: Monday packet waiting`  
Body must include packet id and `queuedCount`. Must not include UltraPavement.

`createGmailNudgePort`: call `nudgeCopy` then `send`. Return send’s result. Never throws into books.

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { createGmailNudgePort, nudgeCopy } from "./nudge-gmail.ts";

describe("nudgeCopy", () => {
  it("names Commercial Control and the queue count", () => {
    const c = nudgeCopy({ packetId: "w3", queuedCount: 3 });
    assert.match(c.subject, /Commercial Control/);
    assert.match(c.body, /w3/);
    assert.match(c.body, /3/);
    assert.doesNotMatch(c.subject + c.body, /UltraPavement/i);
  });
});

describe("createGmailNudgePort", () => {
  it("returns failed when mailer fails", async () => {
    const port = createGmailNudgePort(async () => "failed");
    const status = await port.send({ packetId: "w3", queuedCount: 1 });
    assert.equal(status, "failed");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --experimental-strip-types --test src/lib/monday/nudge-gmail.test.ts`

Expected: FAIL (module missing)

- [ ] **Step 3: Implement `nudgeCopy` and `createGmailNudgePort`**

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/nudge-gmail.test.ts src/lib/monday/nudge.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/nudge-gmail.ts src/lib/monday/nudge-gmail.test.ts
git commit -m "feat: gmail nudge copy for commercial control"
```

Skip if no `.git`.

Preview store keeps the in-app `"sent"` port. Wire `createGmailNudgePort` only when a real `MailSender` exists. Do not block the board on Gmail.

---

# Phase E — Slice 2 certificates

### Task 12: Certified qty on the programme line

**Files:**
- Modify: `src/lib/monday/types.ts` — `Activity.certified?: number`
- Create: `src/lib/monday/certs.ts`
- Create: `src/lib/monday/certs.test.ts`
- Modify: `src/lib/monday/recon.ts` — findings

**Interfaces:**
- Consumes: `Activity`, `ActivityLane`, `Finding`, `OcrRow`
- Produces:
  - `export function applyCertified(programme: Activity[], certs: { activityId: string; certified: number }[]): Activity[]` — sets `certified` on matching ids; unknown ids ignored (not a new pay item)
  - `export function certFindings(lanes: ActivityLane[], programme: Activity[]): Finding[]`

`certified_no_actual` (severity `critical`, check `certified_no_actual`) when `activity.certified` is a number **and** (`lane.actual` is 0 or there is no confirmed actual). Status of that lane becomes `hold` if this finding exists.

Do **not** add certified into `actual` or `forecast`.

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { applyCertified, certFindings } from "./certs.ts";
import { laneFor } from "./recon.ts";
import type { Activity, OcrRow } from "./types.ts";

const pothole: Activity = {
  id: "pothole",
  payItem: "42.08",
  name: "Pothole patching",
  unit: "m2",
  planned: 439,
  contractor: "K",
  roadNo: "DR3660",
};

describe("applyCertified", () => {
  it("does not create a new programme line for unknown activity", () => {
    const next = applyCertified([pothole], [{ activityId: "ghost", certified: 10 }]);
    assert.equal(next.length, 1);
    assert.equal(next[0].certified, undefined);
  });
});

describe("certFindings", () => {
  it("holds certified with no confirmed actual", () => {
    const prog = applyCertified([pothole], [{ activityId: "pothole", certified: 100 }]);
    const lane = laneFor(prog[0], [], 2, 4);
    const notes = certFindings([lane], prog);
    assert.ok(notes.some((n) => n.check === "certified_no_actual"));
    assert.equal(lane.actual, 0);
  });

  it("does not fire when confirmed actual exists", () => {
    const prog = applyCertified([pothole], [{ activityId: "pothole", certified: 100 }]);
    const rows: OcrRow[] = [
      {
        id: "a",
        packetId: "w1",
        activityId: "pothole",
        qty: 92,
        confidence: 0.97,
        status: "confirmed",
        ocrText: "92",
      },
    ];
    const lane = laneFor(prog[0], rows, 1, 4);
    const notes = certFindings([lane], prog);
    assert.ok(!notes.some((n) => n.check === "certified_no_actual"));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --experimental-strip-types --test src/lib/monday/certs.test.ts`

Expected: FAIL (module missing)

- [ ] **Step 3: Implement `applyCertified` and `certFindings`.** Optionally set `lane.status` to `hold` inside a new `laneWithCerts(lane, programme)` helper if `certified_no_actual` applies — keep `laneFor` itself unchanged so slice 1 tests stay green.

```ts
export function laneWithCerts(lane: ActivityLane, programme: Activity[]): ActivityLane {
  const notes = certFindings([lane], programme);
  if (notes.some((n) => n.check === "certified_no_actual")) {
    return { ...lane, status: "hold" };
  }
  return lane;
}
```

Add a test that `laneWithCerts` is `hold` when certified and actual is 0.

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/certs.test.ts src/lib/monday/recon.test.ts`

Expected: PASS (slice 1 recon still green)

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/certs.ts src/lib/monday/certs.test.ts src/lib/monday/types.ts src/lib/monday/recon.ts
git commit -m "feat: certified_no_actual hold when certificate has no daily actual"
```

Skip if no `.git`.

---

### Task 13: Exception sheet shows certificate holds

**Files:**
- Modify: `src/lib/monday/recon.ts` or board to concatenate `findings(...)` + `certFindings(...)`
- Modify: `src/components/monday/exception-sheet.tsx`
- Modify: `src/components/monday/programme-table.tsx` — optional column Certified; `—` when undefined
- Modify: `src/lib/monday/store.ts` / `seed.ts` — fixture: pothole `certified: 100` **off** by default; add `seedCerts` used only after a **Load certificates** watch of a fixture JSON, or seed one demo cert on pothole as `undefined` until a `loadCertsFixture()` store action.

Do not silently set certified on seed W1/W2 (would change the 439 demo hold story). Add store action `loadCertsFixture()` that applies `{ activityId: "pothole", certified: 300 }` so the exception sheet can show `certified_no_actual` **only after** that action — wait, W1/W2 already have 210 actual, so 300 certified would NOT fire `certified_no_actual`.

Use **concrete** fixture for the board: `loadCertsFixture()` applies `{ activityId: "reserve", certified: 20 }` and road reserve confirmed actual stays 11 from W1 only if W2 has no reserve wait — seed W1 reserve 11 confirmed, W2 none. After 2 Mondays actual 11, certified 20 → has actual, no `certified_no_actual`.

For a visible hold: apply certified to an activity with **zero** confirmed rows, e.g. introduce no extra activity — use a programme line that has no confirmed OCR. Seed does not have that. `loadCertsFixture()` should apply `{ activityId: "signs", certified: 50 }` and the test/board relies on queued-only signs (9 queued, 18 confirmed W1). Signs have confirmed 18 — would not fire.

Add programme activity `guardrail` only in cert fixture? Spec says unknown ids ignored.

Simplest board demo: `loadCertsFixture()` sets certified on `pothole` after user **rejects all pothole confirmed**? Too cute.

**Do this:** `loadCertsFixture()` applies `{ activityId: "pothole", certified: 100 }` **and** the exception sheet also shows a **warning** `certified_vs_actual` later — out of spec. Stick to spec: `certified_no_actual` only.

Add to seed a programme line used only for slice 2:

```ts
{
  id: "edge",
  payItem: "99.01",
  name: "Edge marker posts",
  unit: "no",
  planned: 10,
  contractor: "Road Signs JV",
  roadNo: "DR3660",
}
```

No OCR rows. `loadCertsFixture()` → `{ activityId: "edge", certified: 8 }` ⇒ `certified_no_actual`. Slice 1 recon tests that build their own `pothole` Activity are unaffected. `allLanes(activities, ...)` will show edge as 0 actual — on-track/under until certs loaded, then hold.

- [ ] **Step 1: Add `src/lib/monday/certs-board.test.ts` failing test** that `allLanes` + `laneWithCerts` + `certFindings` on edge+certified 8 and no rows yields hold + `certified_no_actual`.

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { applyCertified, certFindings, laneWithCerts } from "./certs.ts";
import { laneFor } from "./recon.ts";
import type { Activity } from "./types.ts";

const edge: Activity = {
  id: "edge",
  payItem: "99.01",
  name: "Edge marker posts",
  unit: "no",
  planned: 10,
  contractor: "Road Signs JV",
  roadNo: "DR3660",
};

it("edge markers certified 8 with no daily actual is hold", () => {
  const prog = applyCertified([edge], [{ activityId: "edge", certified: 8 }]);
  const lane = laneWithCerts(laneFor(prog[0], [], 2, 4), prog);
  const notes = certFindings([lane], prog);
  assert.equal(lane.status, "hold");
  assert.ok(notes.some((n) => n.check === "certified_no_actual"));
});
```

- [ ] **Step 2: Run test to verify it fails** if `laneWithCerts` missing

- [ ] **Step 3: Seed `edge` activity. Store `loadCertsFixture()`. Exception sheet concatenates cert findings. Programme table shows Certified column. Button on Watch tab: `Load certificate fixture` (not an upload of files — fixture only). Copy: certificates come from SharePoint in production; this button is the fixture.**

- [ ] **Step 4: Run**

```
node --experimental-strip-types --test src/lib/monday/*.test.ts
node scripts/browser-smoke.mjs
```

Expected: unit PASS. Smoke 200, no `Upload`, body can contain `Exception`.

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday src/components/monday
git commit -m "feat: exception sheet holds certified-without-actual"
```

Skip if no `.git`.

---

## Self-review (everything the spec named)

| Spec item | Where |
|---|---|
| Queued not in totals / 439 example | Phase A Task 1 |
| OCR split 0.82, failed packet, duplicate path | Phase A Task 2 |
| Unplanned reject-only | Phase A Task 3 |
| Watch empty/missing, no invented qty | Phase A Task 4, Phase C Task 10 |
| Mail nudge + failed mail not books | Phase A Task 5, Phase D Task 11 |
| Exception sheet, no upload, product name | Phase A Task 6, Phase E Task 13 |
| OCR never guesses pay item | Phase B Task 7–8 |
| Folders are configuration; site not `/sites/Oshakati` | Phase C Task 9 |
| Watcher read-only | Phase C Task 10 |
| `certified_no_actual` hold | Phase E Task 12–13 |
| Not PM Suite | Global + UI copy |
| Live Edge SSO / real Tesseract / live Gmail delivery | Adapters exist; tests use fakes. Live calls optional. |

No TBD. `certified` does not feed `actual` or `forecast`.
