# 11 — Integration of Round-4 Design Reviews

Consolidates:
- `docs/reviews/09-codex-design-r4.md` — Codex (pty-2)
- `docs/reviews/10-claude-design-r4.md` — Claude (pty-3)

Both reviewers explicitly: v4 is **not yet ready for Phase 4 implementation**. v5 needed.

The encouraging news: round 4 issues are smaller and more concrete than round 3's. Round 1-3 surfaced architectural redesigns; round 4 surfaces lifecycle subtleties, type-shape mismatches, and operability hardening. We're converging in *kind* even if not yet in *count*.

Both reviewers explicitly endorse the architectural direction: dynstate-only ownership, plain Transform substrate, control-subtree generation triggers, three-tier write classification, cancel two-lane, file abstraction split, DeviceOps facade, §15.5 conscious-deviations consolidation. The v5 changes refine within that direction; no architectural pivots.

---

## Convergent items (both reviewers raised independently)

| # | Issue | Codex r4 | Claude r4 | Decision |
|---|-------|----------|-----------|----------|
| **A1** | **Lifecycle: actor construction happens before `Layer.load_from_db()`**. v4's §3.6 D3a "platform addition + restore at construction" is broken: `params.dynstate` would be `None` at actor construction even with the field added. The D3b fallback (transform_wrapper stash) is actually closer to the live lifecycle because `transform_wrapper(..., dynstate)` runs post-restore via the forced recompute. | HIGH 1 | H1 (type-shape angle) + H5 (plain-Transform-in-RFS-list angle) | **Accept fully. Promote D3b to primary.** Drop the §3.6 D3a "platform addition + restore at construction" path. The runner reads its initial dynstate from `transform_wrapper`'s first post-restore call (which stashes it on the `TransformFunction` instance). v5 §14 Phase-4 prerequisite list: nothing required. v2.0 ask: a cleaner `on_restored_dynstate(dynstate)` actor callback (or restructure init to happen post-restore) — captured but not blocking. |
| **A2** | **Tier A "synchronous, await ack" is not implementable** against the existing `update_dynstate(?gdata.Node) -> None` fire-and-forget API. There's no per-call commit ack the runner can await. | HIGH 2 | H2 | **Accept. Restructure §3.7 around idempotent-side-effects, not synchronous-persist.** v5 §3.7: drop "await ack" wording. Tier A becomes "publish dynstate update before side effect, **AND** design the side effect to be safe under re-fire after crash" — leaning on idempotency. Concrete consequence: IOS-XR `op_id_*` (Phase 6) moves out of Tier A into Tier B with a documented recovery story via `o.get_current_install_request()` device-side observation (matches the Python `SoftwareCommit.pre_check` pattern). v2.0 ask: `update_dynstate(node, cb: action() -> None)` ack variant — captured. |
| **A3** | **Global config subscription wiring constraint is a footgun.** A fresh integrator could place `/software-install/` and `/sw-rfs:rfs/software-pack/` in the same host layer; `params.lower` then doesn't see the global config; runner sits idle reading defaults silently. | HIGH 4 | H4 | **Accept. Add runtime fail-loud guard.** v5 §7.2: runner declares the subscription AND publishes `runner-status: enum {ok, missing-global-config, restore-inconsistent, ...}` to oper if no relevant data arrives within a 5s startup window after first `on_local_config`. Also add a concrete layer-topology example to the §2 integration guide showing the required separation. v2.0 ask: "subscribe to current layer root" platform API — captured. |
| **A4** | **§15.5 #10 contradicts the ADR's no-stubs decision.** The ADR (and v3 reviewer-driven update) said no per-method `NotImplementedError` stubs; §15.5 #10 still says "CLI strategy methods exist as stubs in Phase 4." | M5 | M5-equiv | **Accept. Rewrite §15.5 #10:** "CLI strategy is not selected in Phase 4; per-OS `Ops` modules carry a `cli_session: ?CliSession` field but no CLI methods are stubbed. Phase 5 adds CLI methods alongside the existing NETCONF ones." |
| **A5** | **`devname_from_path(params.path)` cannot reuse the existing `devname_from_device_path` helper** verbatim. The existing helper expects a `PathKey`; sw-install's path is `PathContainer("software-pack")` whose parent is the `PathKey` for the RFS entry. | MEDIUM 1 | M1 | **Accept. Specify the helper concretely in v5 §7.1:** walk `params.path` looking for the `PathKey` whose parent is `/sw-rfs:rfs`, take `name`. Or: `params.path.parent` is the RFS entry's `PathKey`; apply `devname_from_device_path` to that. Document with code snippet. |

---

## Codex-r4-unique catches

| # | Issue | Decision |
|---|-------|----------|
| **CR4_1** | **HIGH 3: Transform can't see sibling `scp-port`.** v4 placed `scp-port` at `/sw-rfs:rfs[name]/scp-port` (sibling of `software-pack`). With the transform attached at `software-pack`, `cfg` only carries the `software-pack` subtree — the runner doesn't see `scp-port`. | **Accept. Move `scp-port` inside `software-pack/`** (back to v3 nested placement). Reasoning: scp-port is sw-install-specific configuration anyway; the "non-sw-install consumer can share the leaf" rationale was speculative (no concrete consumer exists today). The v3 nested placement loses "scp-port survives unbinding the pack" — that's fine; if a user removes the pack, they're disabling sw-install for the device, and re-binding restores both leaves. Update YANG, §9.6, §15.5 #14. |
| **CR4_2** | **MEDIUM 2: `dev.get_modules()` returns `(dict[str, ModCap], ?str)`, not just `dict`.** §8.7 sketch uses `if len(modules) > 0:` against the wrong shape. | Accept. Fix: `modules, _modset_id = dev.get_modules(); if len(modules) > 0:`. Sorespo's pattern uses `dev.get_modules().0` for the dict portion. Also tighten readiness rule: "non-empty modset AND adapter is not NoAdapter". |
| **CR4_3** | **MEDIUM 3: §3.3 mentions `packages` and `reload-required` IOS-XR diagnostics; v4 YANG is missing them.** | Accept partial: explicitly defer those two to Phase 6 YANG changes. Note in §3.3 that current YANG models common + SROS-only diagnostics; IOS-XR-specific leaves beyond `op-id-*` arrive in Phase 6. |
| **CR4_4** | **MEDIUM 4: Junos per-RE diagnostics described in §3.3 but not modeled in YANG.** | Accept. Same treatment as CR4_3 — explicitly defer to Phase 6. |
| **CR4_5** | **MEDIUM 6: `auto-execute-after-confirm` path has no idempotency anchor.** If `auto-execute-after-confirm` is true and confirmations are present, no `start-generation` is consumed — restart could repeatedly auto-start. | Accept. Add a per-request `auto_started_after_confirm: bool` flag in dynstate. First auto-start sets it to `true` (Tier A persist before run); subsequent restarts see it set and skip auto-start (request resumes via the existing restart story). |
| **CR4_6** | **LOW 1+2: Stale Phase-3 decision language in `00-orientation.md` and `01-software-install-logic.md`** still reads as open questions. | Accept. Add "resolved in 02-sw-install-design v5 §X" cross-reference notes; or wrap in a "historical extraction context" header. |
| **CR4_7** | **LOW 3: YANG `request` description references "design doc 02 §3.3"; v4 retention is §3.4.** | Accept. Trivial fix in yang.act. |

---

## Claude-r4-unique catches

| # | Issue | Decision |
|---|-------|----------|
| **CL4_1** | **HIGH 1: §8.2 declares `update_oper`/`update_dynstate` as `action(...)`, but the platform exports them as `proc(...)`.** Will not typecheck. | **Accept. Trivial fix.** Change to `proc(...)` in §8.2 and §7.2. |
| **CL4_2** | **HIGH 3: §3.5 restore-consistency check description doesn't match what it actually catches.** The check is dynstate-internal (`max(history.id, current.id) ≥ next_request_id`), not cross-cutting config-vs-dynstate. The "config newer than dynstate" framing is misleading; that case is genuinely undetectable with current platform persistence (no oper persistence, no independent record). | Accept. v5 §3.5 rewrite: demote to "internal-consistency invariant guard for dynstate-blob restore"; note that cross-cutting backup-restore safety is a v2.0 follow-up requiring an independent record (e.g., a config-side `last-request-id-hint`). Update §15.5 #5 wording correspondingly. |
| **CL4_3** | **HIGH 5: Plain `Transform` inside `/sw-rfs:rfs` has no platform precedent.** Sorespo's per-list-entry pattern uses `RFSTransform`. A future reader will see `swi_factory` wired alongside `BBInterfaceTransform` and reasonably expect RFS-style semantics (e.g., that `params.dev` is populated). It isn't. | Accept. v5 §7 add an explicit one-paragraph note: "deliberate departure from convention because (a) `_RFSTransaction.finalize` suppresses on_conf for empty output, (b) `RFSFunction.init_dynstate` doesn't thread `lower`. Trade-off: runner extracts devname from path and calls `dev_registry.get` itself instead of receiving `params.dev`." Add v2.0 platform ask: parameterize the empty-output suppression OR thread `lower` through `RFSFunction.init_dynstate`. |
| **CL4_4** | **MEDIUM 3: `confirm-all` `by-user` value unspecified** — what value does the runner stamp in the oper `confirmed.by-user` projection when triggered by `confirm-all-generation`? | Accept. Use sentinel `"<confirm-all>"` (operationally legible; RESTCONF-friendly). |
| **CL4_5** | **MEDIUM 4: Single-slot `last-trigger-result` is racy for fast successive triggers.** Cancel-then-clear-run-log overwrites the cancel result. | Accept partial: keep single slot but document the limitation in §4.6 and YANG. The dedicated "ring of last 5" widening is a v2.0 enhancement; for v1, the run-log itself records each trigger consumption (the runner emits a swi_*-tagged log entry) — that's the per-call audit trail. |
| **CL4_6** | **MEDIUM 5: `_drain_notify` token comparison should be `<` not `+1 ==`.** With the current logic, if the runner has started a new run after cancel-then-restart, the drain check fails and the watchdog has to fire (600s wait). Wider comparison handles drain-from-multi-generations-ago. | Accept. Fix in §8.2 sketch to use `stale_token < current.generation_token`. |
| **CL4_7** | **MEDIUM 6: Oper data is not platform-persisted.** After a platform restart, oper is empty until first reconciliation publishes it. RESTCONF clients polling during the gap see no requests. | Accept. Add to §8.4 restart sequence: "first `on_local_config` is when oper is first populated; clients polling during the gap see empty data and should retry." Add §15.5 entry: "oper data is not platform-persisted across restart, unlike Python NSO CDB oper." |
| **CL4_8** | **MEDIUM 7: `dev_registry.get(devname)` call shape — synchronous or async?** Need to verify before Phase 4 codes the `act()` callback as if synchronous. | Accept partial: flag in §8 as a one-line check during Phase 4 skeleton; if async, add `waiting-for-device-mgr` pre-state. |
| **CL4_9** | **MEDIUM 8: Stale callback drops op_id_* mutations.** Chicken-and-egg with Tier A: can't persist op-id before issuing the RPC because we don't know it yet. | Accept. Move IOS-XR `op_id_*` out of Tier A into Tier B with explicit recovery story via `o.get_current_install_request()` device-side observation. (Same fix as A2.) |
| **CL4_10** | **LOW 7: Phase 4 RunLogHandler should leave a hook for Phase 5 secret redaction.** Add `swi_redacted: True` flag honored by the handler. | Accept. Add to §6.6 `RunLogHandler` contract: "skips records with `swi_redacted=True` in their structured-data dict." |
| **CL4_11** | **L1: `00-orientation.md` says "actor Layer owns a ttt.Layer instance"; Layer IS the actor.** | Accept. Trivial wording fix. |
| **CL4_12** | **L4: IOS-XR `Cleanup` is also a no-op when `FileTransfer.caps().delete == false`** (Phase 4 Noop), parallel to CheckFiles. | Accept. Add §15.5 entry for parallel deviation. |
| **CL4_13** | **Onboarding: 4-row table (config / memory / dynstate / oper) in `00-orientation.md` would save a cross-doc trip.** | Accept. Add to orientation. |
| **CL4_14** | **Onboarding: design doc references `docs/reviews/08-integration-r3.md` from §"Status" line; not useful for fresh readers.** | Accept. Move to footnote / drop the inline reference. |
| **CL4_15** | **Onboarding: `adr/cli-driver.md` Phase 4 obligations are buried at the end; lift to top.** | Accept. ADR restructure. |
| **CL4_16** | **Onboarding: docs need a reading-order index.** Consider a `docs/README.md` or pinning order at the top of `00-orientation.md`. | Accept. Add reading-order to top of `00-orientation.md` (lighter than a separate README). |

---

## Decisions on the load-bearing items

### D5 — Promote D3b (transform_wrapper stash) to primary; drop D3a as not-fixable-with-platform-addition (per A1)

Codex's HIGH 1 lifecycle finding is correct: actor init runs before `Layer.load_from_db()`, so even with `TransformActorParams.dynstate`, the field would be None at construction. The D3b fallback (`transform_wrapper` stashes dynstate on the `TransformFunction` instance; runner reads it on first `on_conf`) actually leverages the existing post-restore recompute and works without platform changes.

v5 §3.6: D3b becomes primary. §14 Phase-4 prerequisite list becomes empty (no platform addition required for v1). v2.0 platform ask: a cleaner `on_restored_dynstate(dynstate)` actor callback that runs after restore, before first config reconciliation. This is more invasive than the v4 "five-line additive change" v3 framing — it requires platform lifecycle restructuring.

**Implication: v5 unblocks Phase 4 without any platform addition.** This is actually a strong win — no external dependency on the platform team for Phase 4 to start.

### D6 — Tier A semantics: drop "synchronous await ack"; lean on side-effect idempotency (per A2)

The platform's `update_dynstate` is fire-and-forget. v5 §3.7 reframes Tier A around: "publish dynstate update before side effect, AND design the side effect to be safe under re-fire after crash." Concretely:

- **Generation high-water marks** (last_*_generation) — publish before work; re-fire is no-op (the work is itself idempotent: re-creating a request with same pack returns same id; re-starting an unprocessed request is a no-op in §4.2).
- **`next_request_id`** — publish before externalizing; re-fire would assign a new id, but that's caught by §3.5's restore-inconsistency check.
- **Backoff/`next_wake_at`** — publish before scheduling `after`; re-fire would schedule another `after` (the older `after` is lost on restart, so the new one is correct).
- **IOS-XR `op_id_*` (Phase 6)** — moves out of Tier A entirely. Persist after the device returns the op-id (Tier B). Recovery via `o.get_current_install_request()` per Python.

§15.5 entry: "Tier A semantics are 'publish-before-side-effect + idempotent-on-re-fire,' not 'await commit.' The platform's update_dynstate is fire-and-forget."

### D7 — Move scp-port back inside software-pack/ (per CR4_1)

The v3→v4 split (sibling) was driven by "scp-port should survive pack unbinding" — speculative benefit. Codex's HIGH 3 catches the actual cost: the runner can't see scp-port when it's a sibling.

v5: move scp-port back inside `software-pack/`. Update YANG, §9.6, §15.5 #14. Loses the survives-pack-unbinding property; gains: runner can read it directly.

### D8 — Add `runner-status` oper enum + 5s startup-window guard (per A3)

v5 §7.2: runner declares the global subscription AND publishes `runner-status` to oper. Initial value `starting`. After 5s if no relevant `/software-install/` data has arrived from the subscription → `runner-status = missing-global-config`. After successful first reconciliation with global config seen → `runner-status = ok`. On §3.5 restore inconsistency → `runner-status = restore-inconsistent`. Operators get fail-loud feedback for the most common misconfiguration.

YANG addition: `runner-status: enum {starting, ok, missing-global-config, restore-inconsistent, paused-by-enabled, ...}` under `software-pack/runner-status` (config false).

---

## What v4 got right (carry forward unchanged)

Per both reviewers' explicit endorsements:

- Plain `ttt.Transform` substrate (vs RFSTransform) — verified by both reviewers via direct code reading.
- Dynstate-only state ownership consolidation.
- Three-tier write classification framing (Tier B and Tier C are fine; Tier A wording fix per A2).
- Generation-counter triggers + per-request `*-target-id` scoping.
- `last-trigger-result` for fail-loud feedback.
- `cancelling` enum + drain watchdog.
- `LocalFileInspector` / `RemoteFileInspector` / `FileTransfer` three-way split.
- DeviceOps facade with CLI ADR-deferred.
- §15.5 conscious-deviations consolidation.
- `(when, seq)` run-log keying.
- `enabled` state machine.
- Plain Transform inside `/sw-rfs:rfs` (with the deliberate-departure note per CL4_3 H5).

These survive v5 unchanged.

---

## v5 action items (priority-ordered)

In rough order of size:

1. **§3.6 rewrite — D3b becomes primary** (A1). Drop D3a "platform addition + restore at construction" path. Phase-4 prerequisite list (§14) becomes empty.
2. **§3.7 rewrite — Tier A semantics reformulated** (A2 + D6). Idempotent-on-re-fire instead of synchronous-ack. Move IOS-XR `op_id_*` out of Tier A.
3. **§3.5 rewrite — clarify what the restore-consistency check actually catches** (CL4_2). Demote cross-cutting backup-restore safety to v2.0 follow-up.
4. **§7 add deliberate-departure note** for plain-Transform-in-RFS-list (CL4_3). v2.0 platform ask added to §14.
5. **§7.2 + §2 — concrete layer-topology example** + runner-status oper guard (A3 + D8). YANG add `runner-status` enum.
6. **YANG: move scp-port back inside `software-pack/`** (CR4_1 + D7). Update §9.6, §15.5 #14.
7. **§8.2 — fix action(...) → proc(...) types** (CL4_1).
8. **§8.2 — fix _drain_notify token comparison `<` not `+1 ==`** (CL4_6).
9. **§8.7 — fix `dev.get_modules()` shape** (CR4_2). Tighten readiness rule.
10. **§4.2 — auto_started_after_confirm flag** (CR4_5).
11. **§4.3 — confirm-all by-user sentinel `"<confirm-all>"`** (CL4_4).
12. **§7.1 — concrete devname-from-path helper** (A5).
13. **§6.6 — RunLogHandler swi_redacted hook** (CL4_10).
14. **§15.5 #10 rewrite** (A4). Add new entries: tier-A-semantics, oper-not-persisted, IOS-XR Cleanup also no-op (CL4_12), single-slot trigger-result limitation.
15. **00-orientation.md polish:** 4-row state-table, Layer-actor wording fix, reading-order at top, drop inline review link (CL4_11/13/14, CL4_16).
16. **adr/cli-driver.md:** Phase 4 obligations to top (CL4_15). Drop §15.5 #10 contradiction (A4).
17. **01-software-install-logic.md §12 / 00-orientation §3:** add "resolved in v5" cross-references (CR4_6).
18. **YANG: §3.4 reference fix** (CR4_7). Phase 6 IOS-XR/Junos diagnostic leaves explicitly deferred (CR4_3+CR4_4).

---

## Round-5 expectation

After v5 lands, both reviewers should converge on a "ready for Phase 4" verdict if:

- The primary-D3b shift survives — neither reviewer raises a substrate-blocking issue.
- The Tier A reformulation reads cleanly.
- The runner-status guard pattern is judged sufficient for v1 wiring safety.
- The remaining items don't surface a new HIGH.

If round 5 surfaces another batch of HIGHs, we go to round 6. Quality bar is "both reviewers no longer have HIGH or MEDIUM items"; pace continues to converge round-over-round (r3→r4 was 6 HIGH → 5 HIGH, with smaller scope each round).

The "unblocks Phase 4 without platform changes" outcome from D5 is a meaningful milestone: v5 is implementable on the existing platform with no external dependencies. That makes round 5 the most likely point at which both reviewers converge to "ready."
