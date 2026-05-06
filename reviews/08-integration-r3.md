# 08 — Integration of Round-3 Design Reviews

Consolidates:
- `docs/reviews/06-codex-design-r3.md` — Codex (pty-2)
- `docs/reviews/07-claude-design-r3.md` — Claude (pty-3)

Both reviewers explicitly said v3 is **not ready for Phase 4 implementation** — for the same reasons. The convergence is the strongest signal of this round: independently verifying live `ttt.act` / `device.act` / `adapters/adapter.act`, both reviewers landed on four substrate-level issues that v3 didn't catch. Two of those (H1, H2) are real Phase 4 blockers; two (H3, H4) are documentation drift that's trivial to fix.

Tone note: both reviews went deep into the platform code this round (codex read 3 platform files + sorespo's `t_2.act`/`base_2.act`; claude verified 6+ platform code paths and cited line numbers). The substrate-level findings are the kind of thing only a code-grounded read surfaces. We're past the "is the design plausible" stage — the docs now have to match what the platform actually does.

---

## Headline takeaway

Both reviewers reached a "do not start Phase 4" verdict for the same two reasons (H1 + H2). Both reviewers also caught the embarrassing **`DeviceMgr.get_dmc()` already exists** issue independently — neither r1 nor r2 noticed it, but r3's deeper code-read surfaced it.

The architectural direction (dynstate-only ownership, per-device runner, control-subtree triggers, three-way file split, ADR-deferred CLI) **is sound** — both reviewers explicitly endorse it. The blockers are about *where the design's claims meet the platform code*: the implementation path I sketched in §3 / §7 / §8 doesn't survive contact with `ttt.act`'s actual signatures.

v4 needs to land before round-4 review. Most of v4 is local-impact corrections; H2 may require a small platform addition (or a deliberate architectural compromise).

---

## Convergence matrix — issues both reviewers raised independently

| # | Issue | Source | Decision |
|---|-------|--------|----------|
| **H1** | **Empty downward output suppresses on_conf in `RFSTransform`** AND **`params.lower` is never populated for RFS-shape transforms** (RFSFunction.init_dynstate doesn't pass it). The current `RFSTransform` substrate is incompatible with sw-install's observer-only shape. | Codex r3 HIGH 2; Claude r3 H1 | **Accept fix: switch from `RFSTransform` to plain `ttt.Transform`.** Plain `Transform`: (a) accepts empty output without suppressing on_conf (no equivalent guard in `_TransformTransaction.finalize`), (b) does populate `params.lower` (`TransformFunction.init_dynstate` passes it through), (c) does NOT populate `params.dev` — runner extracts devname from `params.path` and calls `dev_registry.get(devname)` itself (same pattern as `_DeviceTransaction.devname_from_device_path`, ttt.act:2206). Zero platform change. Rewrite §7. |
| **H2** | **Restored `dynstate` doesn't reach the runner actor.** `_TransformTransaction.restore` sets `self.dynstate` on the transaction; `TransformActorParams` doesn't include dynstate; `on_conf(cfg, memory)` only delivers `memory`. v3 §3 / §8.4's "runner constructed with restored dynstate injected" is not implementable. | Codex r3 HIGH 1; Claude r3 H2 | **Accept fix: ask for a small additive platform change.** Add `dynstate: ?gdata.Node` field to `TransformActorParams`; thread `self.dynstate` through `_TransformTransaction.init_dynstate` and `_RFSTransaction.init_dynstate`. Existing callers (sorespo) ignore the new field — purely additive. Add this to §14 platform prerequisites; document the v4 design as conditional on this 5-line platform patch landing. **Alternative if platform team rejects:** stash dynstate on the `TransformFunction` instance from inside `transform_wrapper`'s first call after restore, runner reads it from there. Ugly but doesn't require platform change. Default to additive platform change; fall back to stash if needed. |
| **H3** | **`DeviceMgr.get_dmc()` already exists** (`device.act:401`). v3 §9.5/§14/Q2 frames it as a platform ask. Neither r1 nor r2 caught this. Pure docs drift. | Codex r3 MEDIUM 4; Claude r3 H3 | **Accept fully.** Drop the ASSUMPTION in §9.5; drop Q2 from §12; drop item 3 from §14. Keep "read DMC at use time" guidance. Frame as "uses the existing `DeviceMgr.get_dmc()` API." Note codex's sub-point: `get_dmc()` is an actor method; verify the call shape compiles cleanly in Phase 5 (probably fine; one-liner). |
| **H4** | **YANG augments `/sw-rfs:rfs` but design narrative says `/devices/device/...`.** Two distinct lists: `/sw-rfs:device` (device meta config) vs `/sw-rfs:rfs` (RFS-layer per-device list — sorespo augments this). YANG is correct (sw-install belongs on `/sw-rfs:rfs`). Design doc is wrong. | Codex r3 HIGH 4; Claude r3 H4 | **Accept fix: update design doc, no YANG change.** §7.1 wiring sketch, §4.8 table, §9.6 rationale, §15.5 #14 all need updating to say `/sw-rfs:rfs[name]/`. The "scp-port survives unbinding the pack" rationale (§9.6, CL_R2_5) is half-true — leaf does survive pack unbinding (it's a sibling, not nested), but doesn't survive the device being removed from the RFS list. Reword precisely. |

---

## Codex-r3-unique catches

| # | Issue | Decision |
|---|-------|----------|
| **CR3_1** | **Cancel callback guard prevents the cancelling→cancelled transition.** Token-bump no-ops the in-flight callback; the completion needs to BOTH not mutate plan/state AND notify drain so cancelling→cancelled can fire. Two-lane rule needed. | Accept. Rewrite §8.5: stale callbacks no-op for plan-state mutations but invoke a `_drain_notify(token)` path that handles the cancelling→cancelled transition. Add adapter-level timeout contract (or watchdog) so a never-returning RPC doesn't make `cancelling` permanent. |
| **CR3_2** | **Phase 4 pre-staged-image story conflicts with `CheckFiles`.** Step list starts with `CheckFiles` (LocalFileInspector); if operator pre-stages on the device only, controller-side check fails before RemoteFileInspector can prove the copy is unnecessary. | Accept partial: clarify that Phase 4 SROS still requires controller-side files for `CheckFiles` (matches Python original); document this limitation explicitly in §15.5 + the YANG description. Phase 5 (with FileTransfer real) restores the upload path. **Or** make `CheckFiles` a no-op when `FileTransfer.caps().put == false`, with the understanding that `CopyImage.pre_check` then takes over file verification via RemoteFileInspector — this is a conscious deviation from Python's flow and needs §15.5 entry. **Lean: the latter — it's the only sensible Phase 4 user story for "pre-stage and run." Add to §15.5 as a deviation.** |
| **CR3_3** | **Coalesced dynstate writes endanger restart-critical fields.** v3 §8.3 says high-frequency state writes coalesce — fine for run-log entries, NOT fine for `op_id_*` / `rebooted` / `boot_time` / `destination_paths` / generation high-water marks before side effects begin. | Accept. Rewrite §8.3 to classify dynstate fields into three tiers: **(a) MUST-PERSIST-BEFORE-SIDE-EFFECT** — generation high-water marks, op_id_* values, request status transitions; **(b) PERSIST-AT-STEP-BOUNDARY** — destination_paths, rebooted, boot_time, plan; **(c) BEST-EFFORT TELEMETRY** — run-log entries, polling progress. Document the rule per dynstate field. |
| **CR3_4** | **Backoff type/rounding unspecified** — decimal factor → fractional seconds → `uint32` YANG leaf. | Accept. Specify: `error_count.backoff` rounded to ceiling integer seconds; published in oper as `uint32`. Calculation internally uses decimal; rounding at oper-projection time. |
| **CR3_5** | **Trigger target errors should not be silent.** Operator sets cancel-target-id 7 + cancel-generation+1; if no request 7 exists, cancel just vanishes. | Accept. Add `last-trigger-result` oper leaf alongside `last-create-result`: `{generation, target-id, kind: accepted|rejected, reason, at}`. Single-slot, last-writer-wins; preserves the simple model. |
| **CR3_6** | **Request history retention overcomplicated** — top-level `last_pack_snapshot` AND retention rules in `history` are doubly-stored idempotency baselines. | Accept. Top-level `last_pack_snapshot` is authoritative; simplify §3.3 retention to "keep `current` + last terminal-of-each-status + up-to-N additional entries"; remove the "never drop the entry holding the snapshot" rule (the snapshot is at the dynstate root, not in history). |
| **CR3_7** | **Confirmations as config need pruning semantics.** What happens to `control/confirmation` entries for a request id that's been pruned from history? Or for a future request id that doesn't exist yet? | Accept. Add §4.3 sub-rules: (a) entries for pruned request ids are silently retained in config (no runner-side cleanup of user-controlled config) but no-op; (b) entries for not-yet-existing request ids are observed when the matching request materializes; (c) `confirm-all-generation` does NOT create persistent `control/confirmation[]` entries — it sets internal `confirmed-implicitly` markers in dynstate per-step. |
| **CR3_8** | **`waiting-for-device` wakeup mechanism unspecified.** | Accept partial: spec polling at fixed interval (e.g. 30s) for v1; add v2.0 platform ask in §14 for `DeviceMgr.on_status_change(cb)` event. |
| **CR3_9** | **`request-target-id` for `request-generation` is confusing** — it doesn't make sense to scope "create a new request" to an existing id. | Accept. Drop `request-target-id` from the YANG control subtree. Keep target-ids on start/cancel/confirm-all/clear-run-log. |
| **CR3_10** | **ADR per-method CLI stubs are dead surface.** Phase 4 doesn't need per-method `NotImplementedError`-raising stubs; just having `cli_session: ?CliSession` field on `SrosOps` is enough. | Accept. Rewrite ADR's "Phase 4 obligations" section: "`SrosOps` has a `cli_session` field; CLI strategy is not selected in Phase 4 (NETCONF only). No per-method stubs." |

---

## Claude-r3-unique catches

| # | Issue | Decision |
|---|-------|----------|
| **CL3_1** | **Generation-counter restore semantics covers only the safe direction.** v3 §3.5 addresses `cfg < dynstate` (false-non-trigger). The DANGEROUS direction is `cfg > dynstate` (false-trigger): old dynstate restored against new config → spurious trigger fires. Worse, `next_request_id` collides with already-published request ids. | Accept. Rewrite §3.5 + §15.5 #5 to cover both directions. Add a runner-side defensive startup check: "if max published request id ≥ `dynstate.next_request_id`, log error and refuse to materialize new requests until operator manually resets dynstate or bumps `next_request_id` higher." Fail-loud, not fail-silent. |
| **CL3_2** | **Callback-mailbox contract has no enforcement mechanism described.** §6.2 says cb must dispatch on the runner mailbox, but step methods are `class Step(object)` (not actors) so they can't accidentally violate it. | Accept. Tighten §6.2 wording: "Steps are ordinary classes (not actors). The runner constructs `cb` as an `action def` defined on itself; closing over `self` makes the callback dispatch on the runner mailbox automatically. Steps that need helper actors must terminate them before invoking `cb`." |
| **CL3_3** | **Acton's logging doesn't have ContextVar/Filter equivalents.** No thread-local context, no `logging.Filter` chain. Step authors will forget to add `swi_*` keys in structured-data dict. | Accept. Add §6.6 plumbing: `StepLogger` injected into step methods, pre-bound to swi_* attributes; step authors call `step_log.info(msg, extra)` and the swi_* keys are added automatically. Concrete `RunLogHandler` class consumes records with the swi_component attribute and writes them into the bounded ring; non-tagged records pass through to other handlers. |
| **CL3_4** | **`enabled` + backoff: `after backoff: _start_run()` fires regardless of enabled state.** | Accept. Rewrite §8.6: `_start_run()` first checks `current_global_config.enabled`; if disabled, transitions request to `paused` (preserving `next_wake_at`) instead of starting. On `enabled` flip back to true, replay the schedule. |
| **CL3_5** | **Diagnostic projection leaves are always present in YANG regardless of OS.** `op-id-*` shows up on SROS request components as absent leaves; `rebooted` shows on IOS-XR. YANG-tooling-noisy. | Accept. Add `when "../../software-pack-data/os = '<os>'"` constraints to the OS-specific leaves. SROS-only: `rebooted`. IOS-XR-only: `op-id-add/prepare/activate/commit`, `packages`, `reload-required`. Junos per-RE projections deferred to Phase 6. |
| **CL3_6** | **`internal-state` deviation phrasing in §15.5 #1 overstates loss.** Diagnostic projections ARE NETCONF/RESTCONF-visible — that's the whole point. The opaque JSON blob is what's gone. | Accept. Reword §15.5 #1: "`internal-state` (opaque JSON blob) is dropped; operationally-useful fields are now typed leaves under `request/component/` (still RESTCONF-visible). Fields not surfaced as named leaves are no longer externally inspectable." |
| **CL3_7** | **Step protocol type ambiguity.** `cb: action(StepResult, NewState, ?Exception)` — `NewState` not formalized; convention for `exc=None` vs `exc=Some` not stated. | Accept. Formalize: `NewState = ?State` (None means "no state change"); convention "`exc` is non-None iff the step body raised; runner logs traceback and treats as FAILURE." |
| **CL3_8** | **No `on_status_change` event on DeviceMgr.** §8.7's "subscription on the device's status (or polling)" — only polling is viable. | Accept; aligns with CR3_8. Spec polling at 30s; v2.0 platform ask for `DeviceMgr.on_status_change(cb)`. |
| **CL3_9** | **LocalFileInspector needs `file.FileCap` injected.** Caps aren't ambient in Acton; the factory needs `file.FileCap` from the app. | Accept. Add `file_cap: file.FileCap` to `make_sw_install_transform` factory args. |
| **CL3_10** | **Logic spec / orientation doc stale pointers** to "open Phase 3 questions" that are now resolved in v3+ design. | Accept. Add cross-reference notes to `01-software-install-logic.md §12` and `00-orientation.md §3` saying "resolved in 02-sw-install-design v3 §X" with concrete section pointers. |
| **CL3_11** | **`(when, seq)` collision after clear** — clarify that ring is empty post-clear so collision-against-existing isn't possible. | Accept. One-line clarification in §6.6 / yang.act run-log description. |

---

## Where I push back / decide differently

None. The reviewers' findings are all substantive and correct. The closest thing to disagreement is whether v4 handles **CR3_2 (CheckFiles vs pre-staged image)** by making `CheckFiles` no-op when `caps().put == false` (my preference) or requires controller-side files always (codex's first option). I'm going with the no-op-when-no-transfer approach because the "pre-stage on device" workflow is the only sensible Phase 4 user story; documenting it as a conscious deviation is honest.

---

## Two strategic decisions before v4 writing

### Decision D3: How to fix H2 (dynstate restore to actor)?

Two viable paths:

**(a) Platform addition** (preferred): add `dynstate: ?gdata.Node` to `TransformActorParams`; thread through `_TransformTransaction.init_dynstate` and `_RFSTransaction.init_dynstate`. ~5 lines in `ttt.act`. Additive — sorespo and other current callers ignore it. Clean separation.

**(b) Architectural workaround**: `transform_wrapper(cfg, linked, memory, dynstate)` does receive dynstate. Stash it on `self` (the `TransformFunction` instance, which is shared with the runner via the `_on_conf` setup); runner reads it on first `on_conf`. Ugly: makes `transform_wrapper` impure, couples timing of "runner sees dynstate" to "first compute fires," requires careful ordering with the first runner-side `update_dynstate` write.

**Decision: pursue (a), fall back to (b) only if platform team rejects.** Add to §14 as a stated platform prerequisite. Frame v4 design as conditional on the patch landing.

### Decision D4: Use plain `Transform` instead of `RFSTransform`?

Both reviewers recommend yes; the reasons converge:
- Plain `Transform` accepts empty output without suppressing `on_conf` (no `_RFSTransaction.finalize`-style guard).
- Plain `Transform` populates `params.lower` (the `declare_subscriptions` path).
- Plain `Transform` does NOT populate `params.dev`, but the runner can derive devname from `params.path` and call `dev_registry.get(devname)` — established pattern (`_DeviceTransaction.devname_from_device_path`, ttt.act:2206).

The original v3 reasoning ("matches sorespo's per-list-entry RFS pattern") is wrong because sorespo's RFS transforms DO produce per-device downward config — they're not observers. Sw-install IS an observer; plain `Transform` is the right substrate.

**Decision: switch to plain `ttt.Transform`.** Rewrite §7.

---

## Action items flowing into v4 design

In rough order of size:

1. **§7 rewrite** — switch from `RFSTransform` to plain `Transform`; specify devname extraction from `params.path`; clarify `params.lower.declare_subscriptions(...)` for global config; remove §7.4 fallback (with platform addition the choice is unambiguous). Resolves H1.
2. **§3 / §8.4 rewrite** — runner receives restored dynstate via the new `TransformActorParams.dynstate` field (D3a). Add §14 entry for the platform patch. Resolves H2.
3. **§9.5 / §12 / §14 cleanup** — drop `get_dmc()` ASSUMPTION + Q2 + §14 alt, frame as existing API. Resolves H3.
4. **§7.1 / §4.8 / §9.6 / §15.5 path narrative** — `/sw-rfs:rfs[name]/software-pack`, not `/devices/device/...`. Reword scp-port "survives unbinding" precisely. Resolves H4.
5. **§8.5 cancel two-lane callback rule** — stale completions don't mutate plan, but invoke `_drain_notify(token)` to enable cancelling→cancelled. Adapter-level timeout contract or watchdog. Resolves CR3_1.
6. **§8.3 dynstate-write classification** — three-tier rule (must-persist-before-side-effect / persist-at-step-boundary / best-effort). Resolves CR3_3.
7. **§3.5 + §15.5 #5 dual-direction restore semantics** — fail-loud check on startup if config and dynstate are inconsistent. Resolves CL3_1.
8. **§3.3 history retention simplification** — top-level `last_pack_snapshot` is authoritative; drop the duplicate retention rule. Resolves CR3_6.
9. **§4.3 confirmations pruning rules**; §4.6 drop `request-target-id`; new `last-trigger-result` oper leaf. Resolves CR3_7, CR3_5, CR3_9.
10. **§6.6 logging plumbing** — `StepLogger` per-step; concrete `RunLogHandler` class. Resolves CL3_3.
11. **§6.2 callback contract clarity** — steps are classes, runner closes over `action def`. Resolves CL3_2.
12. **§8.6 `enabled` gate in `_start_run`**. Resolves CL3_4.
13. **§8.7 polling spec** — 30s interval; v2.0 platform ask. Resolves CR3_8 / CL3_8.
14. **§9.4 CheckFiles vs pre-staged image** — `CheckFiles` no-ops when `FileTransfer.caps().put == false`; document as deviation. Resolves CR3_2.
15. **YANG `when` constraints** for OS-specific diagnostic projections. Resolves CL3_5.
16. **YANG drop `request-target-id`**. Resolves CR3_9.
17. **YANG add `last-trigger-result`** oper leaf. Resolves CR3_5.
18. **Backoff rounding spec** — uint32 ceiling. Resolves CR3_4.
19. **§15.5 wording fix** for internal-state deviation. Resolves CL3_6.
20. **Step protocol type formalization**. Resolves CL3_7.
21. **`file.FileCap` in factory**. Resolves CL3_9.
22. **ADR Phase 4 obligations cleanup** — drop per-method CLI stubs language. Resolves CR3_10.
23. **Stale-pointer cleanup in `00-orientation.md` and `01-software-install-logic.md`** — cross-reference resolved questions. Resolves CL3_10.

Plus smaller polish items (§6.1 four/five count, NewState type, run-log clear semantics).

---

## Round-4 expectation

After v4 lands, both reviewers should converge on a "ready for Phase 4" verdict if:

- The substrate-level fixes (D3 platform addition, D4 substrate switch) hold up.
- The remaining 21 items are addressed with no new high-priority surprises.

If round 4 surfaces another batch of substrate-level issues, we go to round 5. Quality bar remains "both reviewers no longer have HIGH or MEDIUM items."

The platform addition (TransformActorParams.dynstate) is the single load-bearing piece that's outside this design's control. If the platform team rejects it, v4 has to fall back to D3b (transform_wrapper stash) — which both reviewers flagged as unclean but both acknowledged as workable. Either way we're not stuck.

---

## What round 3 confirmed that's working

Per both reviewers' explicit endorsements:

- **State ownership consolidation in dynstate (§3)**: clean fix for the v2 memory/dynstate confusion. Right call.
- **Control-subtree generation counters (§4)**: internally coherent. Per-request scoping addresses the CR/§4 round-2 finding.
- **`cancelling` / `paused` / `waiting-for-device` enum additions**: appropriate state-machine refinement.
- **Three-way file abstraction split (§9)**: resolves the v2 NoopFileTransfer/CopyImage incoherence cleanly.
- **CLI ADR lift**: both reviewers ratify the boundary; ADR scope is appropriate.
- **`§15.5` consolidated deviations**: claude r3 explicitly praised this; "the kind of artifact a design doc should contain — keep it."
- **YANG control subtree shape, paused/cancelling/waiting-for-device enums, run-log `(when, seq)` keying, last-create-result oper**: all good.

These survive v4 unchanged.
