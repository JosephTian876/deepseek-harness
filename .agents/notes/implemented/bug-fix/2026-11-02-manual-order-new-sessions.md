# Agent Note: Manual order leads with Sessions it has not saved

Status: implemented

English | [中文](2026-11-02-manual-order-new-sessions.zh.md)

## Problem

A browser's Manual order is a saved sequence of Session ids, written when the user drags a row, pins one, or settles a new Session's position. It is a snapshot: any Session created afterwards — in another browser, another tab, or as a fork — is absent from it. `reconcileManualOrder()` used to place those absent members after the entire saved block, ordinary ones by recency, so they rendered only after every saved Session. That is invisible in practice: a Workspace keeping a long history shows just its first rows until the five-row fold is raised, so a Session created on another device landed hundreds of rows down and looked lost. The report was a live Session appearing at the position of very old Sessions on every fresh client.

## Decision

**Members without a saved position lead the saved sequence, newest first.** The emitted order is missing pins in pin-array order, then absent ordinary members by recency, then the saved relative positions, then missing archived members. Saved members keep their relative order and slots; a member already in the saved sequence stays in its slot even when it is also pinned. Supplementation remains in memory: deriving an order never writes one, so the saved sequence still records only what a drag, pin, or new-Session position write saved, and those writes continue to record the complete result.

## Alternatives considered

**Switch Manual to Last updated while a member is missing.** Rejected: it discards the layout the user built and changes the selected mode without consent.

**Persist the supplemented order when it is read.** Rejected: a read would write browser-local storage, which contradicts the explicitly saved order and would silently make one client's supplementation permanent for every reader.

**Keep appending missing members after the saved block.** Rejected: this is the reported defect — with a long history the new row stays unreachable behind the fold.

## Consequences

- A Session created by another browser, another tab, or a fork surfaces at the top of Manual on every fresh client, where the fold already shows it.
- Drag and pin writes still record the complete order, so a user can move the new row to the slot they want and that choice sticks.
- Rows may move once when supplementation lands: existing Sessions without a saved position jump from the end to the front of the saved block.
- Missing archived members still trail the list and missing pins still lead it; saved members' relative order and slots are unchanged.

## Testing

- `packages/client/ui-workspace/tests/tree.client.spec.ts` — `leads a stale saved order with Sessions created after it was saved` covers newer Sessions leading by recency, the saved relative order, a new pin leading, a new archive staying last, a member without a summary, and idempotence; the existing reconciliation, fork, and Ungrouped cases assert the new order.
- `packages/client/ui-workspace/tests/workspace-browser.client.spec.tsx` — `leads Manual with Sessions its saved order never recorded (created in another tab or browser)` is the rendered regression: it pins the visible row order and the five-row fold count, a saved archive keeping its slot while an unrecorded archive still trails, a Session arriving in the open tab landing at the head of the unrecorded block, expanding revealing the saved block in its saved relative order, an unrecorded selected blank still leading, and the saved sequence never being rewritten.
- `pnpm exec vitest run packages/client/ui-workspace/tests/tree.client.spec.ts --coverage --coverage.include='packages/client/ui-workspace/src/client/tree.ts'` keeps 100% branch, statement, function, and line coverage of `src/client/tree.ts`.
