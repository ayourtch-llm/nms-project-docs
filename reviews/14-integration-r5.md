# 14 — Integration of Round-5 Design Reviews

Consolidates:
- `docs/reviews/12-codex-design-r5.md` — Codex (pty-2)
- `docs/reviews/13-claude-design-r5.md` — Claude (pty-3)

Both reviewers are at **"green-light Phase 4 after a v5.1 doc pass"** — the strongest convergence so far. Both verified v5's substrate claims against live platform code; both agree the architecture is sound; both flag a small set of localized fixes.

This is dramatic narrowing from r3 → r4 → r5: HIGH count fell from 6 → 5 → 3 + 3 (with significant overlap). The remaining issues are doc precision (idempotency framing), one cross-actor mutable read (claude H1), and timer/period race in the runner-status guard. None require architectural redesign.

---

## Convergent r5 issues

| # | Issue | Codex r5 | Claude r5 | Decision |
|---|-------|----------|-----------|----------|
| **A1** | **§3.7 Tier A idempotency framing is too strong for some triggers.** Codex caught `request-generation` (designed to force a NEW request — non-idempotent by design); claude caught `auto_started_after_confirm` (Tier A flag persists separately from Tier B status; crash window). | HIGH 1 | HIGH 2 | **Accept both fixes.** Add a per-request `materialized_by_request_generation: u64` anchor — runner suppresses duplicate materialization if a current request already records this generation. Add a §3.7 invariant: "Tier A field updates are always batched into the same `update_dynstate(...)` snapshot as the consequences they trigger." For auto_started_after_confirm specifically: co-publish with `current.status = processing` in one snapshot. |
| **A2** | **runner-status guard interacts badly with subscription `period`.** Codex: data semantics for "global config seen" undefined. Claude: first `_sub_tick` fires synchronously before `load_from_db`, delivering `None`; if `period > 5s`, every well-wired startup transiently false-alarms. | HIGH 3 | HIGH 3 | **Accept fix: rephrase the guard.** The 5s timer starts not at "first on_local_config" but at "first call to on_global_config with `merged != None`" (or, equivalently, the runner enters `ok` only after observing non-None global config). Pin `period = 5.0` in the §7.2 example. Define "global config seen" as: the cb received a non-None `gdata.Node` containing the `/software-install/` container at any depth (not just an empty container). Strengthen §14 v2.0 ask #4 (Layer "current root" subscribe API) — would close this race entirely. |
| **A3** | **§3.6 / §8.4 restore order is misdescribed.** Codex pointed out the actor is constructed BEFORE Layer.load_from_db (during rootgen); my doc said the opposite. | HIGH 2 | A1 (verified the timing trace) | **Accept fix.** Rewrite §3.6 / §8.4 prose to reflect the live ordering: (1) Layer rootgen constructs SwInstallTransform + DeviceRunner; runner starts with empty in-memory dynstate. (2) Layer.load_from_db restores `_TransformTransaction.dynstate`. (3) `app.StartupBootstrap.recompute(force=True)` triggers `transform_wrapper(...)` with restored dynstate; stashed there. (4) `_TransformTransaction.finalize` calls `on_conf` if `self.running` is non-empty; runner reads stash on first `on_local_config`. **Caveat (claude A1):** if no software-pack is bound (no running config), `finalize` skips `on_conf`; runner stays at `runner-status=starting` indefinitely. This is benign-no-work-to-do but worth documenting. |
| **A4** | **`PathContainer` doesn't exist in ttt.act** — should be `PathElem`. Doc bug from §7.1's deliberate-departure note. | (didn't catch) | LOW 8 | Accept; trivial rename. |
| **A5** | **Q2 should be marked resolved by existing precedent.** `dev_registry.get(devname)` synchronous use is precedented in `_RFSTransaction.__init__` and `testing.act`. | (verified in §"Q2 - Yes") | LOW 9 | Accept; mark Q2 resolved in §12. |

---

## Codex-r5-unique catches

| # | Issue | Decision |
|---|-------|----------|
| **CR5_1** | **MEDIUM 1: devname helper too weak.** v5's helper does `path.parent` and checks for `PathKey`, but doesn't verify the parent's parent is `/sw-rfs:rfs`. Could mismatch under future path nesting. | Accept. Strengthen helper: walk ancestors; require the `PathKey`'s parent to be a `PathElem` named `rfs` in the stratoweave-rfs namespace. Reference: `ttt.act:devname_from_rfs_path`. |
| **CR5_2** | **MEDIUM 2: `runner-status` precedence rules.** When multiple non-ok states apply (e.g., enabled=false + waiting-for-device), which wins? | Accept. Add precedence in §7.2: `restore-inconsistent` > `missing-global-config` > `waiting-for-device` > `paused-by-enabled` > `starting` > `ok`. |
| **CR5_3** | **MEDIUM 3: §15.5 missing the action→generation-counter behavioral deviation.** | Accept. Add a §15.5 entry. |
| **CR5_4** | **MEDIUM 4: Stale "scp-port survives" prose comment in YANG augment header.** Contradicts the v5 nested decision and the leaf's own description. | Accept. Trivial fix. |
| **CR5_5** | **LOW 1: Duplicate `## Context` heading in ADR.** | Accept. |
| **CR5_6** | **LOW 2: YANG `request` description references "design doc 02 §3.3" but v5 retention is §3.4.** | Accept. |

---

## Claude-r5-unique catches

| # | Issue | Decision |
|---|-------|----------|
| **CL5_1** | **HIGH 1: `transform_fn.stashed_dynstate` is a cross-actor mutable read.** Class instance shared between `_TransformTransaction`'s actor (writes via transform_wrapper) and DeviceRunner actor (reads on first on_local_config). Breaks actor isolation. | **Accept fix: action-ref push, not field stash.** During the `act` callback, the runner installs a `stash_cb: action(?gdata.Node) -> None` on the SwInstallTransform instance via an init-time setter. transform_wrapper calls `self.stash_cb(dynstate)` instead of writing the field. The action lands inside the runner's actor context — no shared-state race. |
| **CL5_2** | **MEDIUM 4: `make_sw_install_transform` factory body is hand-waved.** Concrete code shape needed for the closure-sharing between `function_factory` and `act` (so they share the same SwInstallTransform instance for the stash plumbing). | Accept. Add a 10-line skeleton in §7.3 showing the closure-sharing pattern. |
| **CL5_3** | **MEDIUM 5: `transform_wrapper` returns `(Container(), memory)`.** Should be `(Container(), None)` to make the "memory unused" claim airtight and avoid surprising LMDB writes via `update_memory`. | Accept. One-line fix. |
| **CL5_4** | **MEDIUM 6: §15.5 missing two deviations:** runner-status oper enum (no Python equivalent); trigger mechanism action→generation-counter (input side). | Accept. Add both entries. |
| **CL5_5** | **MEDIUM 7: No-software-pack-bound steady state parks runner at `runner-status=starting` forever.** Either document explicitly or add `unbound` enum value. | Accept partial: document explicitly in §7.2 ("`starting` is also the steady-state value when no software-pack is bound"); don't add a separate enum value because operators can disambiguate via `request[]` being empty. |
| **CL5_6** | **LOW 10: `ttt.Transform`'s `lower=` kwarg is dead code** (shadowed by `_create_transform_node(path, lower)`). Platform-side observation; doesn't affect sw-install correctness. | Accept; flag in §14 platform notes. |

---

## v5.1 action items (priority-ordered)

In rough order of impact on Phase 4 readiness:

1. **§3.6 + §7.3: action-ref push for stash** (CL5_1 HIGH 1). The single biggest doc change in v5.1. SwInstallTransform gets a `stash_cb` field; `transform_wrapper` calls it. DeviceRunner installs the cb at construction.
2. **§3.7: Tier A invariant** — "Tier A field updates are always batched into the same `update_dynstate(...)` snapshot as the consequences they trigger" (A1). Plus add `materialized_by_request_generation` anchor for the request-generation case.
3. **§3.6 + §8.4: rewrite restore lifecycle prose** (A3). Reflect the live ordering: rootgen construction → load_from_db → recompute(force=True) → transform_wrapper stash → on_conf → first on_local_config reads stash.
4. **§7.2 runner-status guard rephrase** (A2). Timer starts at "first non-None on_global_config", not "first on_local_config + 5s". Pin `period = 5.0` in example. Add precedence rules (CR5_2). Document `starting` steady-state for no-pack-bound (CL5_5).
5. **§7.3 factory skeleton** (CL5_2). 10-line concrete code shape.
6. **§7.1 devname helper hardening** (CR5_1). Walk ancestors; verify `/sw-rfs:rfs` parent.
7. **§7.1 doc fix:** `PathContainer` → `PathElem` (A4).
8. **§3.6 transform_wrapper returns `None` for memory** (CL5_3).
9. **§15.5 add runner-status, trigger-mechanism, action→generation deviations** (CL5_4 + CR5_3).
10. **§12 mark Q2 resolved** (A5).
11. **§14 add platform observations:** ttt.Transform lower kwarg dead code (CL5_6).
12. **YANG: stale scp-port augment-header comment** (CR5_4); §3.3 → §3.4 reference fix (CR5_6).
13. **ADR: deduplicate `## Context` heading** (CR5_5).

---

## Round-6 expectation

If v5.1 lands these 13 items cleanly, **round 6 is most likely the convergence point**. Both reviewers explicitly said Phase 4 implementation can start after these doc-only fixes (with the small action-ref-push code-shape change in §7.3). v5.1 is doc/skeleton work; no architectural revisions; no platform changes.

If round 6 surfaces a fresh batch of HIGHs at substrate level (unlikely given the sharp narrowing across r3→r4→r5), we go to v6. Otherwise we lock in v5.1 and start Phase 4 implementation.

Strategic note: the **round-over-round narrowing is the converge signal we've been looking for**. r3 found 6 HIGHs requiring architectural redesigns; r4 found 5 HIGHs requiring substrate-level fixes; r5 found 3+3 HIGHs all of which are doc/skeleton-shape changes. r6 is expected to surface either zero HIGHs ("ready") or 1-2 polish-level items at most.

---

## What r5 confirmed is solid

Per both reviewers' explicit endorsements:

- **Plain `ttt.Transform` substrate** — verified independently by both reviewers via direct reads of `_RFSTransaction.finalize`'s empty-output suppression and `RFSFunction.init_dynstate`'s missing `lower`. Trade-off correctly assessed.
- **`scp-port` reverted to nested.** Round-4 catch was real; v5 fix is correct.
- **`get_modules()` shape and NoAdapter detection** confirmed correct.
- **`dev_registry.get(devname)` synchronous use** precedented and confirmed.
- **`transform_wrapper` post-restore timing** (claude verified end-to-end through `app.act`'s StartupBootstrap path).
- **`type-shape `proc(...)` (not `action(...)`)** correct.
- **`_drain_notify` token comparison `<` not `==`** correct.
- **`auto_started_after_confirm` flag concept** correct (only the Tier-batching wording needs tightening).
- **Phase 4 NETCONF-only / Phase 5 CLI+FileTransfer / Phase 6 IOS-XR+Junos phasing** — clean.
- **DeviceOps boundary** — clean.
- **§15.5 consolidated deviations** — useful pattern; needs two more entries.

These survive v5.1 unchanged.
