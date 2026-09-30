---
name: g67-review
description: Use when reviewing or writing G67 Engine C++ changes that touch multi-threading, async/deferred work, object lifetime, resource release, or exception/early-return paths. Turns requirement intake, code-context collection, modification constraints, and static risk review into a repeatable gate, grounded in real future-branch bug patterns (C1..C10). Also use to record AI-review true positives, false positives, and misses.
---

# G67 Engine Code Review

Repeatable review gate for the G67 Engine (`G:\g67\AIClothFuture\Engine`). Built from 1170
future-branch bugfixes over ~1.5 years. Companion evidence lives in the bugfix archive
(`commits_index.md`, `commits_full_diff.txt`, `analysis_categorized.md`). Optimize for the four
recurring hard classes: thread safety, object lifetime, resource release, and exception/early-return
paths. Do not pad the review with style nits; find the crash and the leak.

## Core engine idiom: thread-affinity suffix

Method/field suffixes declare which thread the code must run on. Treat them as a contract:

| Suffix | Thread | Meaning |
|---|---|---|
| `_on_ot` / `_on_final_ot` | object/logic | gameplay/object thread |
| `_on_rdt` | render | render dispatch thread |
| `_on_uet` | update encoder / device | GPU upload / device work |
| `_on_dt` | dispatch / sync point | single-threaded hand-off between threads |
| `_on_any` | any | must be internally safe on any thread |

**Primary race smell:** the same non-atomic field is read/written from two different suffixes. That
is the root cause of most C1/C2 bugs (e.g. `9bdac3ec8e` split `mUpdateFunc` into `mUpdateFuncRdt` /
`mUpdateFuncUet`). Prefer per-thread ownership plus one explicit hand-off at an `_on_dt` point over
adding a lock.

## Step 1 — Requirement intake

- Restate the feature/bug in one line and capture the ticket (`refs #NNNNNN`). No ticket, no commit.
- Classify the change against the risk table. If it hits any row, run the full gate; otherwise do one
  static risk screen plus normal functional validation.

| Change touches | Run full gate |
|---|---|
| Any `_on_*` method, shared field, or cross-thread queue | Yes |
| Async/deferred/GC/callback, or work that spans frames | Yes |
| Object create/destroy, refcount, `Ghost`/`RenderProxy`, ownership | Yes |
| Allocation, pool, cache, handle, descriptor, or transient buffer | Yes |
| A disable/early-return/failure branch of a feature | Yes |
| Fixed-size buffer, message/log formatting, index math | Yes |
| Pure local logic, no state/thread/alloc change | No — one screen only |

## Step 2 — Collect code context (before editing)

For every symbol you change, collect:

- **Thread affinity:** which `_on_*` context calls it; list every caller thread.
- **Ownership:** who allocates, who owns, who releases; is it shared/refcounted (`TSharedObjectGhost`,
  `IRenderResource`, `IObject`/`IObjectR`).
- **Lifetime span:** create site, destroy site, and every queue it passes through
  (`GFinalizeQueue`, `GDeferredQueue`, ticker/dispatch queues).
- **External/async inputs:** anything from a callback, GC, device driver, script (Python), or another
  frame that can be null or stale.
- **Container aliasing:** any raw pointer/iterator taken into a growable container.

Record an allocation/ownership row for each new resource:

| Resource/handle | Alloc site (thread) | Owner | Release endpoint(s) | Fires on disable/fail path? |
|---|---|---|---|---|

## Step 3 — Modification constraints

- Do not widen a field's thread exposure. If two threads need it, split per thread + hand off at
  `_on_dt`; do not silently share.
- Prefer lock-free per-thread pools over a shared locked pool on hot paths (`6b21a292c9`).
- Never keep a raw pointer/iterator into a `vector`/buffer across an insert; `reserve()` exact size
  first or store an index (`917ee8d817`).
- Keep destroy and defer/schedule queues drained as a matched pair on background/foreground and world
  switch (`4f5babef7a`).
- Bound every copy into a fixed buffer on all branches, including "impossible" error/log branches
  (`170921a361`). Check transient/pack sizes against their max (`8002d5c02d`, 512K).
- Release on every exit: success, cancel, fail, disable, early return, world switch (`851f9d3a57`).
- English comments only; UTF-8. Follow existing naming and the suffix convention.

## Step 4 — Static risk review (the C1..C10 checklist)

Walk each item; cite the exact line when you flag one.

- **C1 Data race:** same non-atomic state across two `_on_*` suffixes? → per-thread split + hand-off.
- **C2 Init-ordering race:** is the object read before its owner thread finished construction/init?
  (`14c86f5c9a`, `478895ddd8`). Trace the init happens-before, not just the local diff.
- **C3 Lock contention/corruption:** two threads on one pool/cache? → independent per-thread backing.
- **C4 Interior-pointer invalidation:** pointer/iterator into a container that later grows? → reserve.
- **C5 Queue-ordering dangling:** is every enqueue matched by the correct drain on state transition?
- **C6 Refcount/dual-ownership:** does add/release balance? is the same payload owned twice
  (`cb82184f7a`)? is there a release far from the alloc that is now missing?
- **C7 Null on render/async path:** prim/shader/texture/mesh/material/scene possibly null when used
  on `_on_rdt`/async? Guard real external/async inputs; do NOT add redundant guards to already-validated
  invariants (avoid the noise trimmed in later "加保护" commits).
- **C8 Out-of-bounds/overflow:** buffer sizes, message length, index/stride math, oversized packs.
- **C9 Leak on disable/exception path:** every early-return and disabled-feature branch releases what
  it allocated? (`851f9d3a57`, `e746b5961b`).
- **C10 Async-callback lifetime:** can a deferred/GC/callback fire after the owner is gone or null?
  (`5ac165720a`, `0c42dc5053`). Capture by value/id, not a raw owner pointer, when it may outlive.

## Step 5 — Verify and report

- Trace at least one cross-thread and, if relevant, one cross-frame path end-to-end. Local diff
  reading alone misses C2/C5/C6.
- Compile the actually affected target; a Windows Hybrid build does not prove mobile or `_FINAL`.
- Report: category hits with file:line, the fix pattern applied, residual risks, and the paths you
  traced.

## Step 6 — Log AI-review outcomes

Append to `analysis_categorized.md` for every non-trivial AI-assisted review:

| Field | Record |
|---|---|
| True positives | Which C-classes the AI correctly caught, with commit/ticket. |
| False positives | Over-warnings (e.g. lock proposed where per-thread split fits; redundant null guard). |
| Misses | What the AI failed to see (usually non-local C2/C5/C6) and how it was found. |

AI is strong on local, pattern-matchable defects (C4, C7, C8, C9) and weak on non-local
lifecycle/ownership reasoning (C2, C5, C6). Force the cross-thread/cross-frame trace to cover the gap.

## Rationalization checks

| Temptation | Correction |
|---|---|
| "It's only read on one thread" | Prove it from the suffix/callers before assuming; a cross-suffix touch is a race |
| "Add a lock to be safe" | This codebase prefers per-thread ownership + `_on_dt` hand-off; lock is the last resort |
| "The pointer is fine, it's the same buffer" | Any insert can reallocate; reserve or index |
| "Add a null check everywhere" | Guard external/async inputs only; redundant guards are noise |
| "The error branch can't happen" | Bound its buffer and release its resources anyway |
| "Diff looks clean" | C2/C5/C6 live across subsystems/frames; trace lifecycle, not just the diff |
