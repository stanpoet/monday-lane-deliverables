# Monday Lane Slice 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Commercial Control’s recon books honest: queued OCR never enters totals or forecast, SharePoint-shaped ingest/watch/nudge exist, and the board shows lane + exception sheet with no upload UI.

**Architecture:** Keep the existing TanStack Start preview. Split load-bearing logic out of the Zustand store into pure functions (`recon`, `ingest`, `watch`, `nudge`) with a fixture SharePoint adapter. The live Path D Edge SSO watcher is an adapter for later — slice 1 tests the contract with a fixture listing. Slice 2 `certified` is not implemented.

**Tech Stack:** React 19, TanStack Start, Zustand persist, node:test (`node --experimental-strip-types --test`), existing Monday Lane files under `src/lib/monday/` and `src/components/monday/`.

**Spec:** `docs/superpowers/specs/2026-09-15-monday-lane-commercial-control-design.md`

## Global Constraints

- Product name in UI: **Commercial Control**. Never “UltraPavement” or “PM Suite”.
- Confidence hold is **0.82**. Over band **1.08**. Under band **0.82** after **≥ 2** ingested Mondays.
- Month total = `confirmed` qty only. Queued qty is excluded from actual, forecast, and remaining.
- No drag-drop / file-picker upload. Ingest comes from Watcher (fixture in this plan).
- Packet states: `expected` | `ingested` | `failed` only (no `ocr-running` on the packet).
- Unplanned OCR (no programme line) cannot be confirmed onto a pay item; reject or leave queued.
- Slice 2 `certified` / `certified_no_actual`: do not add fields or UI.
- No git repo in this workspace: skip every Commit step if `.git` is missing.
- Tests run with: `node --experimental-strip-types --test src/lib/monday/*.test.ts`
- Add those files to the `test` script in `package.json` when the first test file exists.

## File structure

| File | Responsibility |
|---|---|
| `src/lib/monday/types.ts` | Types + constants (`CONFIDENCE_HOLD`, `OVER_BAND`, `UNDER_BAND`) |
| `src/lib/monday/recon.ts` | Pure lane + findings; takes `programme: Activity[]` |
| `src/lib/monday/ingest.ts` | Apply OCR rows to books; duplicate-path skip; failed packet |
| `src/lib/monday/watch.ts` | `WatcherPort`; fixture listing of 3406 |
| `src/lib/monday/nudge.ts` | `NudgePort`; send when ingested && queue non-empty |
| `src/lib/monday/store.ts` | Zustand wiring only |
| `src/lib/monday/seed.ts` | September 2026 fixture programme + W1/W2 packets |
| `src/lib/monday/*.test.ts` | node:test files next to the unit |
| `src/components/monday/exception-sheet.tsx` | Exception sheet |
| `src/components/monday/app.tsx` | Board chrome, tabs, product name |
| `src/components/monday/watch-panel.tsx` | Replaces upload/ingest-as-upload |
| `src/components/monday/verify-queue.tsx` | Queue actions; disable confirm on unplanned |
| `package.json` | Register monday tests |

Do not create a second recon engine. Do not touch PM Suite / UltraPavement repos.

---

### Task 1: Recon engine (439 m² example)

**Files:**
- Modify: `src/lib/monday/types.ts`
- Modify: `src/lib/monday/recon.ts`
- Create: `src/lib/monday/recon.test.ts`
- Modify: `package.json` (`test` script)
- Modify: `src/lib/monday/store.ts` and `src/components/monday/app.tsx` only if `allLanes` / `findings` signatures change — pass `activities` in.

**Interfaces:**
- Consumes: `Activity`, `OcrRow`, `MondayPacket` from `types.ts`
- Produces:
  - `export const CONFIDENCE_HOLD = 0.82`
  - `export const OVER_BAND = 1.08`
  - `export const UNDER_BAND = 0.82`
  - `export function weeksElapsed(packets: { status: string }[]): number`
  - `export function laneFor(activity: Activity, rows: OcrRow[], elapsed: number, weeksTotal: number): ActivityLane`
  - `export function allLanes(programme: Activity[], rows: OcrRow[], elapsed: number, weeksTotal: number): ActivityLane[]`
  - `export function findings(programme: Activity[], lanes: ActivityLane[], rows: OcrRow[]): Finding[]`
  - `ActivityLane.status` is `hold` if `pending > 0`, else `over` / `under` / `on-track` per bands. Official `forecast` uses confirmed only.

- [ ] **Step 1: Write the failing tests**

Create `src/lib/monday/recon.test.ts`:

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { findings, laneFor, weeksElapsed } from "./recon.ts";
import type { Activity, OcrRow } from "./types.ts";

const pothole: Activity = {
  id: "pothole",
  payItem: "42.08",
  name: "Pothole patching",
  unit: "m2",
  planned: 439,
  contractor: "Kambwa Construction",
  roadNo: "DR3660",
};

const rowsTwoMondays: OcrRow[] = [
  {
    id: "a",
    packetId: "w1",
    activityId: "pothole",
    qty: 92,
    confidence: 0.97,
    status: "confirmed",
    ocrText: "92",
  },
  {
    id: "b",
    packetId: "w2",
    activityId: "pothole",
    qty: 118,
    confidence: 0.96,
    status: "confirmed",
    ocrText: "118",
  },
  {
    id: "c",
    packetId: "w2",
    activityId: "pothole",
    qty: 47,
    confidence: 0.61,
    status: "queued",
    ocrText: "47?",
  },
];

describe("laneFor potholes 439", () => {
  it("excludes queued qty from actual and official forecast", () => {
    const elapsed = weeksElapsed([{ status: "ingested" }, { status: "ingested" }]);
    const lane = laneFor(pothole, rowsTwoMondays, elapsed, 4);
    assert.equal(elapsed, 2);
    assert.equal(lane.actual, 210);
    assert.equal(lane.pending, 47);
    assert.equal(lane.forecast, 420);
    assert.equal(lane.status, "hold");
    assert.equal(lane.forecastIfConfirmed, 514);
  });

  it("forecast 514 and over after the 47 is confirmed", () => {
    const confirmed = rowsTwoMondays.map((r) =>
      r.id === "c" ? { ...r, status: "confirmed" as const } : r,
    );
    const lane = laneFor(pothole, confirmed, 2, 4);
    assert.equal(lane.actual, 257);
    assert.equal(lane.forecast, 514);
    assert.equal(lane.status, "over");
    assert.equal(lane.pending, 0);
  });

  it("queued qty is not in remaining subtraction", () => {
    const lane = laneFor(pothole, rowsTwoMondays, 2, 4);
    assert.equal(lane.remaining, 229);
  });
});

describe("findings", () => {
  it("emits queued_not_in_total while 47 is queued", () => {
    const lane = laneFor(pothole, rowsTwoMondays, 2, 4);
    const notes = findings([pothole], [lane], rowsTwoMondays);
    assert.ok(notes.some((n) => n.check === "queued_not_in_total"));
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `node --experimental-strip-types --test src/lib/monday/recon.test.ts`

Expected: FAIL (`laneFor` arity / `allLanes` still closed over `seed.activities`, or `forecast` includes queued).

- [ ] **Step 3: Write minimal implementation**

In `types.ts` add:

```ts
export const OVER_BAND = 1.08;
export const UNDER_BAND = 0.82;
```

Change `PacketStatus` to `"expected" | "ingested" | "failed"`.

Rewrite `laneFor` / `allLanes` / `findings` to take `programme` and `weeksTotal`. Official forecast = `(actual / elapsed) * weeksTotal`. `pending > 0` ⇒ `hold`. Else if `forecast > planned * OVER_BAND` ⇒ `over`. Else if `elapsed >= 2` and `forecast < planned * UNDER_BAND` ⇒ `under`. Else `on-track`. `remaining = max(0, planned - actual)`.

Update call sites in `app.tsx` to `allLanes(activities, rows, elapsed, WEEKS_TOTAL)` and `findings(activities, lanes, rows)`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/recon.test.ts`

Expected: PASS

- [ ] **Step 5: Register tests and commit**

Add `src/lib/monday/*.test.ts` to `package.json` `test` script.

```bash
git add src/lib/monday/types.ts src/lib/monday/recon.ts src/lib/monday/recon.test.ts src/components/monday/app.tsx package.json
git commit -m "test: recon excludes queued qty from 439 m2 forecast"
```

Skip commit if no `.git`.

---

### Task 2: Ingest (OCR split, failed packet, duplicate path)

**Files:**
- Create: `src/lib/monday/ingest.ts`
- Create: `src/lib/monday/ingest.test.ts`
- Modify: `src/lib/monday/types.ts` — `MondayPacket.sharePointPath: string`

**Interfaces:**
- Consumes: `CONFIDENCE_HOLD`, `MondayPacket`, `OcrRow` from `types.ts`
- Produces:
  - `export type Books = { packets: MondayPacket[]; rows: OcrRow[] }`
  - `export function routeOcrRow(row: Omit<OcrRow, "status">): OcrRow` — `confidence < 0.82` ⇒ `queued`, else `confirmed`
  - `export function ingestPacket(books: Books, packet: MondayPacket, extracted: Omit<OcrRow, "status">[], outcome: "ok" | "ocr-failed"): Books`

Rules for `ingestPacket`:
- If `books.packets` already has the same `sharePointPath` and that packet is `ingested` or `failed`, return `books` unchanged.
- If `outcome === "ocr-failed"`: upsert packet with `status: "failed"` and add **zero** rows.
- If `outcome === "ok"`: upsert packet `status: "ingested"`, append `routeOcrRow` results.

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { ingestPacket, routeOcrRow } from "./ingest.ts";
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

describe("routeOcrRow", () => {
  it("queues below 0.82", () => {
    const row = routeOcrRow({
      id: "1",
      packetId: "w3",
      activityId: "pothole",
      qty: 12,
      confidence: 0.54,
      ocrText: "12?",
    });
    assert.equal(row.status, "queued");
  });
  it("confirms at 0.82", () => {
    const row = routeOcrRow({
      id: "2",
      packetId: "w3",
      activityId: "pothole",
      qty: 140,
      confidence: 0.82,
      ocrText: "140",
    });
    assert.equal(row.status, "confirmed");
  });
});

describe("ingestPacket", () => {
  it("failed OCR adds zero rows and marks failed", () => {
    const out = ingestPacket({ packets: [packet], rows: [] }, packet, [
      {
        id: "x",
        packetId: "w3",
        activityId: "pothole",
        qty: 1,
        confidence: 0.99,
        ocrText: "should not land",
      },
    ], "ocr-failed");
    assert.equal(out.packets[0].status, "failed");
    assert.equal(out.rows.length, 0);
  });

  it("duplicate sharePointPath is skipped", () => {
    const first = ingestPacket({ packets: [packet], rows: [] }, packet, [
      {
        id: "a",
        packetId: "w3",
        activityId: "pothole",
        qty: 140,
        confidence: 0.95,
        ocrText: "140",
      },
    ], "ok");
    const second = ingestPacket(first, { ...packet, id: "w3-dup" }, [
      {
        id: "b",
        packetId: "w3-dup",
        activityId: "pothole",
        qty: 999,
        confidence: 0.99,
        ocrText: "999",
      },
    ], "ok");
    assert.equal(second.rows.length, 1);
    assert.equal(second.rows[0].qty, 140);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `node --experimental-strip-types --test src/lib/monday/ingest.test.ts`

Expected: FAIL (`Cannot find module ./ingest.ts`)

- [ ] **Step 3: Write minimal implementation**

```ts
import { CONFIDENCE_HOLD, type MondayPacket, type OcrRow } from "./types.ts";

export type Books = { packets: MondayPacket[]; rows: OcrRow[] };

export function routeOcrRow(row: Omit<OcrRow, "status">): OcrRow {
  return {
    ...row,
    status: row.confidence < CONFIDENCE_HOLD ? "queued" : "confirmed",
  };
}

export function ingestPacket(
  books: Books,
  packet: MondayPacket,
  extracted: Omit<OcrRow, "status">[],
  outcome: "ok" | "ocr-failed",
): Books {
  const dup = books.packets.find(
    (p) =>
      p.sharePointPath === packet.sharePointPath &&
      (p.status === "ingested" || p.status === "failed"),
  );
  if (dup) return books;
  if (outcome === "ocr-failed") {
    return {
      packets: upsertPacket(books.packets, { ...packet, status: "failed" }),
      rows: books.rows,
    };
  }
  return {
    packets: upsertPacket(books.packets, { ...packet, status: "ingested" }),
    rows: [...books.rows, ...extracted.map(routeOcrRow)],
  };
}

function upsertPacket(packets: MondayPacket[], packet: MondayPacket): MondayPacket[] {
  const i = packets.findIndex((p) => p.id === packet.id);
  if (i === -1) return [...packets, packet];
  const next = packets.slice();
  next[i] = packet;
  return next;
}
```

Add `sharePointPath: string` to `MondayPacket` and to every object in `seed.ts`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/ingest.test.ts src/lib/monday/recon.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/ingest.ts src/lib/monday/ingest.test.ts src/lib/monday/types.ts src/lib/monday/seed.ts
git commit -m "feat: ingest routes OCR by confidence and skips duplicate paths"
```

Skip if no `.git`.

---

### Task 3: Unplanned rows cannot be confirmed onto a pay item

**Files:**
- Create: `src/lib/monday/queue.ts`
- Create: `src/lib/monday/queue.test.ts`
- Modify: `src/lib/monday/recon.ts` findings copy (reject, not classify)
- Modify: `src/lib/monday/store.ts` `confirmRow`

**Interfaces:**
- Consumes: `Activity`, `OcrRow` from `types.ts`
- Produces:
  - `export function confirmRow(rows: OcrRow[], programme: Activity[], id: string, qty?: number): OcrRow[]`
  - `export function rejectRow(rows: OcrRow[], id: string): OcrRow[]`

`confirmRow`: if the row’s `activityId` is not in `programme`, return `rows` unchanged. Otherwise set `status: "confirmed"` and optional `qty`.

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { confirmRow, rejectRow } from "./queue.ts";
import type { Activity, OcrRow } from "./types.ts";

const programme: Activity[] = [
  {
    id: "pothole",
    payItem: "42.08",
    name: "Pothole patching",
    unit: "m2",
    planned: 439,
    contractor: "K",
    roadNo: "DR3660",
  },
];

const sand: OcrRow = {
  id: "sand",
  packetId: "w2",
  activityId: "sand",
  qty: 14,
  confidence: 0.87,
  status: "queued",
  ocrText: "Sand removal 14 m3",
};

describe("confirmRow", () => {
  it("does not confirm unplanned onto a pay item", () => {
    const next = confirmRow([sand], programme, "sand");
    assert.equal(next[0].status, "queued");
  });
  it("confirms a programme line", () => {
    const row: OcrRow = { ...sand, id: "p", activityId: "pothole" };
    const next = confirmRow([row], programme, "p", 40);
    assert.equal(next[0].status, "confirmed");
    assert.equal(next[0].qty, 40);
  });
});

describe("rejectRow", () => {
  it("rejects unplanned", () => {
    const next = rejectRow([sand], "sand");
    assert.equal(next[0].status, "rejected");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `node --experimental-strip-types --test src/lib/monday/queue.test.ts`

Expected: FAIL (module missing)

- [ ] **Step 3: Write minimal implementation**

```ts
import type { Activity, OcrRow } from "./types.ts";

export function confirmRow(
  rows: OcrRow[],
  programme: Activity[],
  id: string,
  qty?: number,
): OcrRow[] {
  return rows.map((r) => {
    if (r.id !== id) return r;
    if (!programme.some((a) => a.id === r.activityId)) return r;
    return { ...r, status: "confirmed" as const, qty: qty ?? r.qty, note: undefined };
  });
}

export function rejectRow(rows: OcrRow[], id: string): OcrRow[] {
  return rows.map((r) => (r.id === id ? { ...r, status: "rejected" as const } : r));
}
```

Point `store.confirmRow` / `rejectRow` at these functions (pass `activities` from seed). In `findings`, unplanned copy must say it will not join a pay item; reject it — do not say “classify”.

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/queue.test.ts src/lib/monday/recon.test.ts`

Expected: PASS. Unplanned finding text must not contain `classify`.

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/queue.ts src/lib/monday/queue.test.ts src/lib/monday/store.ts src/lib/monday/recon.ts
git commit -m "feat: unplanned OCR cannot confirm onto a pay item"
```

Skip if no `.git`.

---

### Task 4: Watcher (fixture 3406 listing)

**Files:**
- Create: `src/lib/monday/watch.ts`
- Create: `src/lib/monday/watch.test.ts`

**Interfaces:**
- Consumes: `Books`, `ingestPacket` from `ingest.ts`; `MondayPacket` from `types.ts`
- Produces:
  - `export type ListingResult = { ok: true; files: WatchFile[] } | { ok: false; reason: "missing-folder" | "empty" | "unavailable" }`
  - `export type WatchFile = { sharePointPath: string; scanName: string; monday: string; weekNo: number; extracted: Omit<OcrRow, "status">[] | "ocr-failed" }`
  - `export type WatcherPort = { list(folder: string): Promise<ListingResult> }`
  - `export async function runWatch(books: Books, watcher: WatcherPort, folder: string): Promise<Books>`

`runWatch`:
- If listing not `ok`, do not invent packets or rows. For `unavailable` / `missing-folder` / `empty`, leave books unchanged (packets already `expected` stay `expected`).
- If `ok`, for each file call `ingestPacket` with `outcome` `ocr-failed` when `extracted === "ocr-failed"`, else `ok`.

Fixture folder string (verbatim):  
`3406 District Administration/Oshakati District/Projects and Operations`

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { runWatch, type WatcherPort } from "./watch.ts";
import type { MondayPacket } from "./types.ts";

const expected: MondayPacket = {
  id: "w3",
  weekNo: 3,
  monday: "2026-09-21",
  label: "21 Sep",
  status: "expected",
  scanName: "W3.pdf",
  sharePointPath: "/3406/returns/2026-09-21/W3.pdf",
};

const emptyBooks = { packets: [expected], rows: [] };

describe("runWatch", () => {
  it("does not invent qty when listing is empty", async () => {
    const watcher: WatcherPort = {
      async list() {
        return { ok: false, reason: "empty" };
      },
    };
    const out = await runWatch(emptyBooks, watcher, "3406 District Administration/Oshakati District/Projects and Operations");
    assert.equal(out.packets[0].status, "expected");
    assert.equal(out.rows.length, 0);
  });

  it("marks packet failed when OCR failed", async () => {
    const watcher: WatcherPort = {
      async list() {
        return {
          ok: true,
          files: [
            {
              sharePointPath: expected.sharePointPath,
              scanName: expected.scanName,
              monday: expected.monday,
              weekNo: 3,
              extracted: "ocr-failed",
            },
          ],
        };
      },
    };
    const out = await runWatch(emptyBooks, watcher, "3406 District Administration/Oshakati District/Projects and Operations");
    assert.equal(out.packets[0].status, "failed");
    assert.equal(out.rows.length, 0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `node --experimental-strip-types --test src/lib/monday/watch.test.ts`

Expected: FAIL (module missing)

- [ ] **Step 3: Write minimal implementation**

Implement `runWatch` + a `createFixtureWatcher(files: WatchFile[]): WatcherPort` used later by the store. Do not call live SharePoint.

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/watch.test.ts src/lib/monday/ingest.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/watch.ts src/lib/monday/watch.test.ts
git commit -m "feat: watcher fixture never invents qty on empty listing"
```

Skip if no `.git`.

---

### Task 5: Mail nudge (not the books)

**Files:**
- Create: `src/lib/monday/nudge.ts`
- Create: `src/lib/monday/nudge.test.ts`

**Interfaces:**
- Consumes: `Books` from `ingest.ts`; `OcrRow` from `types.ts`
- Produces:
  - `export type Nudge = { packetId: string; queuedCount: number; status: "sent" | "failed" }`
  - `export type NudgePort = { send(nudge: Omit<Nudge, "status">): Promise<"sent" | "failed"> }`
  - `export async function maybeNudge(before: Books, after: Books, port: NudgePort): Promise<Nudge | null>`

Rules:
- Nudge only if at least one packet **newly** became `ingested` (present in `after` as ingested, not ingested in `before`) **and** `after.rows` has `status === "queued"` length > 0.
- Failed OCR (`failed` packet) with zero rows: still nudge if we want “packet landed, OCR failed”? Spec: mail “packet landed, OCR failed”. So also nudge when a packet newly becomes `failed`.
- If `port.send` returns `failed`, return `{ ..., status: "failed" }`. Do **not** mutate books.

- [ ] **Step 1: Write the failing tests**

```ts
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { maybeNudge, type NudgePort } from "./nudge.ts";
import type { MondayPacket, OcrRow } from "./types.ts";

const expected: MondayPacket = {
  id: "w3",
  weekNo: 3,
  monday: "2026-09-21",
  label: "21 Sep",
  status: "expected",
  scanName: "W3.pdf",
  sharePointPath: "/3406/returns/2026-09-21/W3.pdf",
};
const ingested = { ...expected, status: "ingested" as const };
const queued: OcrRow = {
  id: "q",
  packetId: "w3",
  activityId: "pothole",
  qty: 12,
  confidence: 0.5,
  status: "queued",
  ocrText: "12?",
};

describe("maybeNudge", () => {
  it("sends when a packet newly ingested and queue non-empty", async () => {
    const port: NudgePort = { async send() { return "sent"; } };
    const n = await maybeNudge(
      { packets: [expected], rows: [] },
      { packets: [ingested], rows: [queued] },
      port,
    );
    assert.equal(n?.status, "sent");
    assert.equal(n?.queuedCount, 1);
  });

  it("failed send does not change books (returns failed)", async () => {
    const port: NudgePort = { async send() { return "failed"; } };
    const n = await maybeNudge(
      { packets: [expected], rows: [] },
      { packets: [ingested], rows: [queued] },
      port,
    );
    assert.equal(n?.status, "failed");
  });

  it("does not send when queue empty", async () => {
    const port: NudgePort = { async send() { return "sent"; } };
    const n = await maybeNudge(
      { packets: [expected], rows: [] },
      { packets: [ingested], rows: [] },
      port,
    );
    assert.equal(n, null);
  });
});
```

For the failed-OCR mail, add:

```ts
  it("sends on newly failed packet", async () => {
    const port: NudgePort = { async send() { return "sent"; } };
    const n = await maybeNudge(
      { packets: [expected], rows: [] },
      { packets: [{ ...expected, status: "failed" }], rows: [] },
      port,
    );
    assert.equal(n?.status, "sent");
    assert.equal(n?.queuedCount, 0);
  });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `node --experimental-strip-types --test src/lib/monday/nudge.test.ts`

Expected: FAIL (module missing)

- [ ] **Step 3: Write minimal implementation** matching the four tests.

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --experimental-strip-types --test src/lib/monday/nudge.test.ts`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/lib/monday/nudge.ts src/lib/monday/nudge.test.ts
git commit -m "feat: mail nudge on new ingest when queue is waiting"
```

Skip if no `.git`.

---

### Task 6: Board UI — exception sheet, watch, no upload

**Files:**
- Create: `src/components/monday/exception-sheet.tsx`
- Create: `src/components/monday/watch-panel.tsx`
- Modify: `src/components/monday/app.tsx`
- Modify: `src/components/monday/verify-queue.tsx`
- Modify: `src/lib/monday/store.ts`
- Delete or stop importing: `src/components/monday/ingest-panel.tsx` (replace; do not leave an upload CTA)
- Modify: `src/routes/__root.tsx` title → `Commercial Control`

**Interfaces:**
- Consumes: `findings`, `allLanes`, `runWatch`, `maybeNudge`, `confirmRow`/`rejectRow` from prior tasks; `activities` from seed; `createFixtureWatcher` from `watch.ts`
- Produces: the Monday morning board in the preview

UI rules (verbatim):
- Header product name: **Commercial Control**. Subtitle may say Monday Lane as the board nickname.
- Tabs: `Month lane` | `Verify queue` | `Exception sheet` | `Watch`
- No file input, no “upload”, no drag-drop.
- Watch tab: button **Run watch** (fixture listing for 21 Sep). Banner when a packet is `expected` or `failed`. Show last nudge `sent` | `failed`.
- Exception sheet: list findings with severity, check id, detail. Empty state: “No exceptions.”
- Verify queue: **Confirm as read** disabled when activity is not on the programme (unplanned). Reject still enabled.
- `ocrBusy` may exist on the store during watch; it is not a packet status.

Fixture watch file for 21 Sep (use `week3Rows` from seed as extracted, mapped to `Omit<OcrRow,"status">`).

Store additions:
- `lastNudge: { packetId: string; queuedCount: number; status: "sent" | "failed" } | null`
- `runWatch: () => Promise<void>` — `runWatch` then `maybeNudge` with an in-app port that resolves `"sent"` (preview has no Gmail). Record `lastNudge`.
- Remove `ingestWeek3` from the UI. Store may keep it unused; prefer deleting it.

- [ ] **Step 1: Write a failing UI smoke check**

No component test harness. After wiring, run:

```
node --experimental-strip-types --test src/lib/monday/*.test.ts
node scripts/browser-smoke.mjs
```

Until Watch / Exception sheet exist, smoke body will still be the old demo — treat this step as “see current baseline”, then implement.

- [ ] **Step 2: Implement exception sheet + watch panel + store wiring** as specified above. Use existing tokens (`bg-surface`, `text-ink`, `Badge`, `Button`). No emoji in chrome. No purple.

- [ ] **Step 3: Run unit tests**

Run: `node --experimental-strip-types --test src/lib/monday/*.test.ts`

Expected: PASS

- [ ] **Step 4: Smoke the preview**

Run: `node scripts/browser-smoke.mjs`

Expected: status 200, `bodyTextPrefix` contains `Commercial Control`, tabs include exception/queue, **no** substring `Upload` in body. No horizontal overflow. No console/page errors.

- [ ] **Step 5: Commit**

```bash
git add src/components/monday src/lib/monday/store.ts src/routes/__root.tsx
git commit -m "feat: commercial control board with exception sheet and watch"
```

Skip if no `.git`.

---

## Self-review (spec coverage)

| Spec requirement | Task |
|---|---|
| Queued qty not in total/forecast | 1 |
| 439 / 210 / 420 / hold; confirm 47 → 514 over | 1 |
| Confidence 0.82 split | 2 |
| Failed OCR → failed packet, zero rows | 2 |
| Duplicate SharePoint path skipped | 2 |
| Unplanned cannot confirm onto pay item | 3 |
| Watch empty/missing → no invented qty | 4 |
| Mail on ingest+queue; failed mail not books | 5 |
| Failed packet mail | 5 |
| Exception sheet | 6 |
| No upload UI | 6 |
| Product split copy | 6 |
| Slice 2 certified | **not in this plan** (correct) |
| Live SharePoint SSO / real Tesseract / Gmail | **not in this plan** (fixture + in-app sent) |

No TBD. Signatures are consistent: `Books`, `WatcherPort`, `NudgePort`, `confirmRow(rows, programme, id, qty?)`.
