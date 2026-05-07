# 24 — Claude Round-9 Design Review (v5.3.2 self-containment verification)

**Reviewer:** Claude (pty-1)
**Subject:** `docs/02-sw-install-design.md` v5.3.2 — content-only readability pass on top of v5.3.1 lock-in
**Method:** read v5.3.2 end-to-end as a fresh reader; cross-referenced the inlined sections against the round-8 reviews (`21-codex-design-r8.md`, `22-claude-design-r8.md`) for semantic drift; checked internal consistency between the newly-inlined sections and the v5.x-fleshed-out sections (Tier A/B classification, dynstate schema, §4.1 algorithm, etc.); spot-checked the file for any remaining "unchanged"/"carried" pointers.

---

## Verdict

**Pass with one notable gap.** The 14 inlined sections the user listed (§4.5, §5/§5.1/§5.2, §6.1–6.5, §6.7, §8.1, §8.5, §8.6, §9.0–9.4, §9.7, §13, §15) are all present with full prose; I read each one and could not find any semantic drift versus what the round-1..8 reviews validated. The doc reads coherently end-to-end and a fresh reader can now follow the design without bouncing into git history for the listed sections.

The gap: **four other sections still contain stub-pointer placeholders** that the v5.3.2 pass did not inline. These leave readers in the same git-diving position the readability pass was meant to eliminate. I'd call this a MED — non-blocking for Phase 4 (the architecture is locked and the gaps are in less-load-bearing prose), but if v5.3.2's stated goal is "the doc stands alone for a fresh reader without git-diving" (line 3), that goal is not fully met until these four are also lifted.

No other issues. No semantic regressions. No contradictions between newly-inlined content and the previously-fleshed-out v5.x sections.

---

## What I checked

### (1) Inlined content matches what rounds 1–8 validated

I walked each of the 14 listed sections and traced its content back to what reviewers had been operating on:

- **§4.5 clear-run-log** (line 460–466): trigger + behavior + complement-not-replace note — consistent with the v3 spec; the `(when, seq)` keying invariant is preserved (run-log is empty post-clear, so no collision).
- **§5 / §5.1 / §5.2 typed data model** (lines 488–576): pack types, state types, `reset()` semantics, per-OS state subclasses (`StateSros`, `StateIosXr`, `StateJunos`, `RouteEngine`). Field set matches §3.2 dynstate's `RequestState.states[<component>]` and §3.3 diagnostic projections. `dual_re` cross-run invariant preserved (cross-checked vs §6.4 Junos `ValidatePlatform` invalidation rule — CL10 from round-2). ✓
- **§6.1 Step protocol** (lines 582–628): six-parameter step signature (`state`, `ops`, `lfi`, `rfi`, `ft`, `step_log`) + `cb` callback. `StepResult` enum (SUCCESS/FAILURE/SKIP_STEP/SKIP_COMPONENT/WAIT) and `StepKey` (with `re_id` for Junos per-RE addressing) — matches the validated shape. The `cb(result, new_state, exc)` triple convention is consistent with §6.2's `exc` invariant. ✓
- **§6.2 step contract invariants** (lines 630–635): five-bullet list. The "callbacks dispatch on the runner mailbox" item is the load-bearing claim that the §8 generation-token check leans on; it's preserved. ✓
- **§6.3 ComponentPlan invariants** (lines 637–648): refresh discipline (A8), monotonicity (A8), flush ordering (CL8) — all three round-2 annotations preserved. The flush-ordering five-step is the crash-safety invariant for `pre_check`/`execute` semantics; the FAILURE-discards-NewState rule is exactly the "load-bearing invariant for crash safety" described in the round-2 reviews. ✓
- **§6.4 per-OS step lists** (lines 650–655): SROS list per logic-spec §6.1, IOS-XR list per §6.2 (with `Cleanup` IOS-XR-only marker), Junos per §6.3 (with `StepKey(name, re_id=None)` for the trailing `Done` per CL12, and the `dual_re`-changes-invalidates-request rule per CL10). VRP fail-clean. ✓
- **§6.5 status mapping** (lines 657–672): five-branch pseudocode. The "consecutive counters" rule (FAILURE resets `transient`, WAIT resets `other`, DONE clears all of `error_count` including `backoff`) matches the §6.7 retry-budget semantics. ✓
- **§6.7 retry budget** (lines 678–687): per-class budget rule, backoff formula (CR2: exact spec, not `factor * max(...)`), CR3_4 ceiling rounding with the example `(10.0, 12.0, 14.4, 17.28, …)` → `(10, 12, 15, 18, …)`. The example arithmetic checks out; the rule pairs cleanly with §8.6's `backoff_decimal` precise state. ✓
- **§8.1 lease scope** (lines 947–965): the honest downgrade prose (system-wide `MaapiLocker.lock_partial` → sw-install-internal serialization), v1 contract, the README/§15.5 #4/runtime-warning trifecta, the §14 v2.0 platform ask. All consistent with prior reviews' framing. ✓
- **§8.5 cancel implementation** (lines 1027–1053): `processing → cancelling` on `cancel-generation`, generation-token bump, `CANCEL_DRAIN_TIMEOUT = 600s` (matches IOS-XR's `_monitor_operation_log` 600s ceiling — CR3_5 / Q7), the immediate-cancel branches for `(waiting-confirmation, paused, failed-transient, unprocessed)`, and the §8.2 `_drain_notify` handoff with the v5.1 CL4_6 `<` fix. The "stale token < current.generation_token" wording matches §8.2 verbatim. ✓
- **§8.6 backoff** (lines 1055–1085): per-class budget exhaustion → terminal status, `backoff_decimal` precise state, `error_count.backoff` ceiling-rounded for `uint32` projection, `_start_run`'s CL3_4 firing-time `enabled` re-check, restart-resume from `next_wake_at`. Consistent with §6.7 and §6.5. ✓
- **§9 op coverage table + §9.2 LFI + §9.3 RFI + §9.4 FT** (lines 1115–1183): table maps Phase 4 vs Phase 5 vs Phase 6 cleanly. `LocalFileInspector` / `RemoteFileInspector` / `FileTransfer` (with `NoopFileTransfer`) — the three-way split that lets Phase 4 SROS verify pre-staged images via `RemoteFileInspector` while `FileTransfer.caps()` returns all-false. The CR3_2 §9.2 deviation (`CheckFiles.execute` skips controller-side check when `file_transfer.caps().put == false`) is preserved and consistent with §15.5 #15. ✓
- **§9.7 DeviceOps facade** (lines 1193–1212): strategy boundary preserved from the Python `NokiaSrosOperationsProto` design; per-run capability snapshot (CR6 from round-3); Phase 4 = NETCONF-only, no CLI stubs raising `NotImplementedError`; Phase 5 adds CLI alongside. ADR pointer for `cli-driver.md`. ✓
- **§13 deferred details** (lines 1247–1253): four-item list (lmdb key layout, `update_oper` snapshot frequency, logging-handler glue, filesystem primitives). All low-stakes "resolves at the keyboard" items; matches what reviewers consistently said could be deferred. ✓
- **§15 deferred features** (lines 1278–1289): eight-item list. `software-install-matrix` dropped, CLI/TextFSMPlus → Phase 5, IOS-XR archive parsing → Phase 6, VRP fail-clean, Snabb/ONS-TL1/HGW dropped, approval-required out-of-scope, lease API → v2.0, acton-utils textfsm → parallel workstream. All consistent with prior cross-references in §6.4, §11, §14, §15.5. ✓

**No semantic drift.** Every claim in the inlined sections either matches the v3-era prose verbatim (per the user's note that the content was lifted from v3) or aligns with the v5.x corrections that subsequently piled on top. I didn't catch any place where the inlining accidentally rolled back a later fix.

### (2) Doc reads coherently end-to-end as a fresh reader

I read top to bottom with a clean mental model. The flow holds:

- §1 module shape → §2 module boundary + layer topology → §3 state ownership / dynstate schema / restore consistency / how runner receives restored state / Tier classification — each subsection builds on the prior one. The §3.6 lifecycle-ordering trace through `app.act:138-152` and `ttt.act:1232-1331` is concrete enough that a fresh reader can follow it without prior round context.
- §4 control surface (§4.1 reconciliation algorithm with three triggers + crash-recovery example, §4.2 auto-execute idempotency, §4.3–4.7 the smaller triggers, §4.8 YANG diff). The §4.1 algorithm and the §3.7 Tier-A invariant interlock cleanly: §3.7's batching invariant explains *why* the §4.1 trigger materialization works under crash; §4.1's two-condition idempotency check (the v5.3 fix surviving into v5.3.2) explains *what* the anchor enforces. The §4.1 crash recovery example walking through six steps is genuinely useful for a fresh reader.
- §5 typed data model is now self-contained (was the biggest "stand-alone gap" from v5.3.1).
- §6 plan + step semantics is now self-contained — the Step protocol code and the ComponentPlan invariants flow directly into §7 substrate and §8 runner. ✓
- §7.1 deliberate-departure note + §7.2 watchdog algorithm + §7.3 factory body are mutually reinforcing — the closure-shared `fn_holder` pattern reads naturally now that §3.6's lifecycle prose is intact.
- §8 runner architecture builds cleanly: §8.1 lease honest-downgrade → §8.2 actor type fixes → §8.3 four invariants → §8.4 restart story (now carrying the v5.1 corrected lifecycle) → §8.5 cancel → §8.6 backoff → §8.7 device readiness.
- §9 transport scope: the §9.0/§9.1 intro + table land before the per-component sections, which is the right order for a fresh reader. The §9.2/§9.3/§9.4 three-way split is now visible without git-diving.
- §10–§16 wrap up cleanly.

The doc is now substantially better as a "land here, read top-to-bottom, get the design" artifact. The 976 → 1334 line growth is honest content density, not bloat.

### (3) No contradictions between newly-inlined and previously-fleshed-out sections

I cross-checked every place the inlined sections reference (or are referenced by) the v5.x material:

- §3.2 dynstate fields ↔ §3.7 Tier A/B/C classification ↔ §6.5 status mapping ↔ §6.7 retry budget ↔ §8.6 backoff: all four agree on the field names (`error_count.{transient, other, backoff}`, `current.next_wake_at`, `auto_started_after_confirm`, `materialized_by_request_generation`). The Tier A list at line 326 mentions `auto_started_after_confirm` (CR4_5); §4.2's third clause references the same field. ✓
- §3.7's `materialized_by_request_generation` bullet (line 325) — the round-8 MED 1 fix — now reads "if `current.materialized_by_request_generation == cfg_request_gen` AND `target_pack == last_pack_snapshot`, …; otherwise allow Trigger A or Trigger B (pack-change) to materialize." This matches §4.1's algorithm verbatim. ✓ (The round-8 MED 1 hazard — §3.7 contradicting §4.1 — is closed.)
- §6.1 Step protocol's `(state, ops, lfi, rfi, ft, step_log)` signature ↔ §6.4 per-OS step lists ↔ §9.2/§9.3/§9.4 inspector classes ↔ §9.7 DeviceOps. All four sections agree on the parameter set; `lfi`/`rfi`/`ft` correspond exactly to the §9.2/§9.3/§9.4 abstractions. ✓
- §6.4 IOS-XR `Cleanup`-only note ↔ §15.5 #15 IOS-XR `Cleanup` no-op when `FileTransfer.caps().delete == false` parallel deviation. ✓
- §6.4 Junos `dual_re` cross-run invariant (CL10) ↔ §5.2 `StateJunos.dual_re: ?bool` field. ✓
- §8.5 `CANCEL_DRAIN_TIMEOUT = 600s` ↔ §12 ❓Q7 "600s" answer. ✓
- §9.4 `NoopFileTransfer` Phase 4 default ↔ §11 phasing ↔ §15.5 #15 deviation ↔ §9.2 CR3_2 `CheckFiles` skip rule. Four-way consistent. ✓
- §15 deferred features ↔ §15.5 conscious deviations ↔ §14 v2.0 platform asks. The cross-references (e.g., §15 lease API → §14 item 1) resolve correctly. ✓

I found **no contradictions**.

---

## Findings

### MEDIUM 1 — Self-containment goal not fully met: four sections still contain "(unchanged)"/"(carried)" stub pointers

**Location:** four sections that the v5.3.2 pass did not inline.

The v5.3.2 status note (line 3) promises "the doc stands alone for a fresh reader without git-diving." The 14 sections the user listed are all genuinely inlined. But four other sections still contain stub-pointer placeholders, leaving readers in the exact git-dive position v5.3.2 was meant to eliminate:

1. **§4.3 `confirm-step` ↔ writeable confirmations** (line 452):
   ```
   (Unchanged shape from v4.)
   ```
   Followed only by the v5 `<confirm-all>` `by-user` increment. A fresh reader sees nothing about what `confirm-step` does, what the writeable confirmations list contains, or how the CL4_4 sentinel works — they'd have to git-blame back to v4.

2. **§4.7 `enabled` state machine — inter-step gate** (line 476):
   ```
   (Rest of v4 §4.7 carried unchanged.)
   ```
   Only the v5 invariant-4 addition is inlined. The bulk of the `enabled` state machine — what states `enabled = false` puts the runner in, transition rules, recovery — is opaque without git-diving.

3. **§4.8 YANG diff vs Python** (line 482):
   ```
   | (carry v4 entries unchanged unless noted) | |
   ```
   Inside the table — only two of the YANG-diff entries (scp-port, runner-status) are spelled out. The v4 baseline entries are implicit. A fresh reader can't reconstruct "what YANG changed from Python" without prior version access.

4. **§10 Testing strategy** (line 1218):
   ```
   (Mostly carried from v3.)
   ```
   Only the v5.2 `test_topology_misconfigured.act` requirement is inlined. The v3 testing-strategy bulk (unit-test layout, integration-test scope, mocking discipline, coverage targets if any) is not in the doc.

**Why this matters / why it doesn't.** Given v5.3.2's stated goal, this is a coverage gap — the readability pass missed four sections. Whether the gap is *blocking* depends on the load-bearingness:

- §4.3 is a documented control-surface element and Phase 4 will need the writeable-confirmations semantics. **Mildly load-bearing** for implementers.
- §4.7 ties to §8.3 invariant 4 (which IS inlined and self-contained). The §4.7 stub is mostly redundant with §8.3, so this one is closer to **cosmetic**.
- §4.8 is the YANG-diff summary. The actual YANG truth lives in `yang.act`, so this section is more of a narrative aid. **Cosmetic.**
- §10 is the testing-strategy section. If Phase 4 begins TDD-style (per §16's "first TDD cycle (`pack.act` + `test_pack.act`)"), the v3 strategy guidance would be useful to have inline. **Mildly load-bearing.**

None of these are architecture issues. None of them affect whether Phase 4 can start. But they ARE the same flavor of "(unchanged from v3)" stubs that motivated v5.3.2 in the first place — the user noticed ~14 of them and lifted those 14, and now there are 4 more that escaped the pass.

**Recommendation.** Either:

- **Fix:** lift §4.3, §4.7, §4.8, and §10 from v3/v4 the same way the listed 14 were lifted. Probably 30–60 minutes of editing. The v3 prose was the source of truth for the listed-14 lift; the same source should produce these four.
- **Accept:** explicitly amend the v5.3.2 status note (line 3) to say "14 of the 18 'unchanged' stubs are inlined; §4.3, §4.7, §4.8, §10 retain pointer-style references because [reason]" — so a fresh reader at least knows the gap is intentional. This is the cheaper option but trades against the stated self-containment goal.

I'd lean toward Fix. The whole point of v5.3.2 was "stand alone." Half-standing-alone is awkward, especially when the residue is exactly the type of stub the pass set out to eliminate.

This is **non-blocking for Phase 4**. The architecture is locked, the load-bearing material is intact, and these four are the lighter-weight sections. But surfacing as MED because it directly contradicts v5.3.2's promise.

---

### LOW 1 — §16 lock-in note still says "v5.3.1 is the lock-in version"

**Location:** §16, line 1320:

> v5.3.1 is the lock-in version. Round-8 review produced zero HIGH findings; both reviewers (codex r8, claude r8) independently surfaced the same 1 MED + 2 LOW items, all of which v5.3.1 lands.

This is technically still true — v5.3.1 *was* the lock-in version, and v5.3.2 is described in the header as a "content-only readability pass" on top. So §16 reading "v5.3.1 is the lock-in version" is descriptively accurate.

But a fresh reader landing on §16 sees a doc titled "v5.3.2 — self-contained" with §16 declaring v5.3.1 as the lock-in. Without knowing v5.3.2 is purely cosmetic, this looks like the §16 conclusion lags the cover.

**Recommendation.** One-line addition at the end of §16's first paragraph, e.g.:

> v5.3.2 is a content-only readability pass on top of v5.3.1; no architecture, algorithm, or decision changed.

(The line-3 status note already says this; mirroring it in §16 closes the loop for readers who skim to the end.)

Cosmetic — does not affect implementation.

---

### LOW 2 — Pre-existing minor under-specifications, surfacing because the inlined content brings them into view

These are *not* drift introduced by v5.3.2 — they were latent in v5.3.1 too. But the inlining brings them onto the same page where they're easier to notice. Flagging in case a polish pass picks them up:

- **`ErrorCount` type fields not fully spelled out.** §3.2 shows `error_count: ErrorCount` on `RequestState`, but `ErrorCount` itself is referenced only by its members (`transient`, `other`, `backoff` per §3.7; `backoff_decimal` per §8.6). A reader can derive the set, but a one-line `class ErrorCount(value): transient, other, backoff: u32; backoff_decimal: float` definition near §3.2 would be the same density as the surrounding code blocks.

- **§6.7's formula uses `error_count.backoff`, §8.6 uses `error_count.backoff_decimal`.** The two are reconciled by §6.7's "internally compute as decimal" sentence + the (10, 12, 15, 18, …) example, but a strict reader could find the mismatch jarring. §6.7 could read `backoff = (error_count.backoff_decimal or 10.0) * factor` to match §8.6 verbatim and keep the ceiling-rounding sentence as the explicit projection rule.

Both pre-existed v5.3.2; not v5.3.2's fault. Logging here only because they're now easier to notice with the inlined sections side-by-side.

---

## v5.3.2 fix verification

| Item user listed as inlined | Inlined? | Semantic match? |
|-----------------------------|----------|-----------------|
| §4.5 clear-run-log | ✓ | ✓ |
| §5 typed data model intro | ✓ | ✓ |
| §5.1 pack types | ✓ | ✓ |
| §5.2 state types | ✓ | ✓ |
| §6.1 Step protocol | ✓ | ✓ |
| §6.2 step contract invariants | ✓ | ✓ |
| §6.3 ComponentPlan invariants | ✓ | ✓ |
| §6.4 per-OS step lists | ✓ | ✓ |
| §6.5 status mapping | ✓ | ✓ |
| §6.7 retry budget | ✓ | ✓ |
| §8.1 lease scope | ✓ | ✓ |
| §8.5 cancel implementation | ✓ | ✓ |
| §8.6 backoff | ✓ | ✓ |
| §9.0 transport intro | ✓ | ✓ |
| §9.1 op coverage matrix | ✓ | ✓ |
| §9.2 LocalFileInspector | ✓ | ✓ |
| §9.3 RemoteFileInspector | ✓ | ✓ |
| §9.4 FileTransfer | ✓ | ✓ |
| §9.7 DeviceOps facade | ✓ | ✓ |
| §13 deferred details | ✓ | ✓ |
| §15 deferred features | ✓ | ✓ |
| Cosmetic "(unchanged from v4)" suffixes dropped | ✓ (none remaining in body) | n/a |

**Sections still using stub-pointer prose** (per MED 1 above): §4.3, §4.7, §4.8, §10. All four remain in the doc but the `(unchanged from v4)` / `(carried from v3)` markers suggest they were missed rather than deliberately retained.

---

## Recommendation

**Pass.** v5.3.2 delivered on the listed-14 self-containment lift. No semantic drift, no architecture regressions, no contradictions. The fresh-reader experience for the 14 listed sections is materially better.

The MED finding is honest scope: four sections still want the same lift, and the v5.3.2 status note's "stands alone" claim is partially overstated until they're done. Fixing those four is mechanically the same exercise — lift v3/v4 prose, drop the stub pointer.

**Phase 4 readiness is unchanged from v5.3.1's lock-in.** The architecture, algorithms, decisions, lifecycle, FIFO + Tier-A invariants, idempotency anchor, watchdog, scp-port placement — all the load-bearing material — is intact. v5.3.2 is purely additive prose for ~70% of the previously-stubbed sections; nothing got worse.

If a v5.3.3 polish pass is wanted, the natural payload is:
1. Lift §4.3, §4.7, §4.8, §10 from v3/v4 (~30–60 min). [MED 1]
2. One-line §16 amendment about v5.3.2 being readability-only. [LOW 1]
3. Optional: fully spell out `ErrorCount` near §3.2 and align §6.7's formula naming with §8.6's. [LOW 2]

Or just start `acton build` on the scaffold, since none of this is blocking.

---

## Note on independence

I formed this verdict from the v5.3.2 doc + cross-references to round-8 reviews; I did not consult any other reviewer's r9 output (none exists yet at the time of this review). The MED 1 finding was discovered by `grep`-ing the body for residual `(unchanged|carried)` markers after walking the listed-14 sections — it's the kind of completeness check that catches "the pass missed some."
