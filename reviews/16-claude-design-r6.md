# 16 — Round-6 Claude Design Review

Reviewer: Claude (pty-1)
Date: 2026-05-07
Subject: `docs/02-sw-install-design.md` v5.1 + companion docs + `yang.act` v5

Read fresh, in this order:

1. `docs/00-orientation.md`
2. `docs/01-software-install-logic.md`
3. `docs/02-sw-install-design.md` v5.1
4. `docs/adr/cli-driver.md`
5. `stratoweave/sw-install/src/sw_install/yang.act`

Then sanity-checked against live stratoweave:

- `stratoweave/stratoweave/src/stratoweave/ttt.act`
- `stratoweave/stratoweave/src/stratoweave/app.act`
- `stratoweave/stratoweave/src/stratoweave/device.act`
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act`
- `stratoweave/sorespo/src/sorespo/rfs.act`
- `stratoweave/sorespo/src/sorespo/layers/t_2.act`

I formed the assessment below before reading prior round reviews; afterwards I checked the v5.1 action items in `docs/reviews/14-integration-r5.md` and the freshly committed `15-codex-design-r6.md`. Convergence with codex r6 is noted at the end.

---

## Summary / verdict

**Conditional yellow-light.** The architecture is sound and the same as r5 — plain `ttt.Transform`, nested `scp-port`, dynstate restore via post-`load_from_db` recompute, fire-and-forget `update_dynstate` with batched Tier-A invariant — and almost all 13 r5 integration items landed. But two of the v5.1 changes are internally inconsistent in ways that will mislead a Phase 4 implementer:

1. The action-ref stash that CL5_1 was meant to add is described in two different ownership shapes; one of them (§3.6 "Code shape") puts the install back inside the `DeviceRunner` actor body, which is exactly the cross-actor mutation the fix was supposed to remove. §7.3's factory has it right.
2. The `missing-global-config` timer is described in three mutually inconsistent ways across §7.2 prose, §7.2 state diagram, and the YANG `runner-status` description; none of them is implementable as written, and one of them (the prose) cannot detect "missing forever".

These are localized doc/skeleton fixes, not substrate redesigns. I would not green-light Phase 4 from this exact text — an implementer copying §3.6 verbatim ships the bug — but a small v5.2 doc pass with these two tightenings (plus a few minor cleanups) gets there.

---

## HIGH 1 — Action-ref stash has two mutually inconsistent install sites

The CL5_1 action-ref-push fix only half-applied:

- **§3.6 "Code shape" (lines 255-265)** declares `actor DeviceRunner(..., transform_fn: SwInstallTransform, ...)` and includes, at the top of the actor body:
  ```
  transform_fn.stash_cb = self._stash_dynstate
  ```
  This statement runs as part of `DeviceRunner`'s actor initialization — i.e. inside the runner's actor context. `SwInstallTransform` is owned by the `_TransformTransaction` actor (it was constructed in `_TransformTransaction.__init__` via `self.function = function(log_handler)` at `ttt.act:1939`, inside the Layer-actor / Transaction-actor chain). A write from `DeviceRunner` to `transform_fn.stash_cb` is therefore **cross-actor mutation of a non-`value` class instance** — the precise pattern v5.1 was supposed to eliminate per `14-integration-r5.md` A1/CL5_1.

- **§7.2 (lines 458-462) and §7.3 factory (line 562)** instead put the install in the `act` callback:
  ```
  fn.stash_cb = runner._stash_dynstate
  ```
  `act` runs inside `_TransformTransaction.init_dynstate` → `RFSFunction.init_dynstate` (`ttt.act:2011-2012`), in the same actor that owns `fn`. This is same-actor mutation; safe. `runner._stash_dynstate` is an action ref, sendable across actor boundaries; `transform_wrapper` later invokes it from the same Transaction actor (`ttt.act:1942-1943`). No cross-actor field write occurs at any point. Good.

These two installs cannot both be canonical. If both run, you have two writers racing on `stash_cb`. If only the actor-body one runs, you have the cross-actor write. If only the act one runs, the `transform_fn` constructor parameter on `DeviceRunner` is dead code with a stale comment.

**Recommendation.** Make §7.3 the single source of truth. Drop `transform_fn` from `DeviceRunner`'s constructor signature in §3.6 and §8.2, and remove the actor-body assignment. The runner does not need a back-reference to the function instance — only the function instance needs a forward-reference (the `stash_cb` action ref) to the runner. §3.6's prose ("the runner installs an action-ref stash_cb…") should be reworded as "the act callback installs the runner's `_stash_dynstate` action ref into the function instance's `stash_cb` field, before transform_wrapper can fire."

---

## HIGH 2 — `missing-global-config` watchdog is internally inconsistent and has at least one unimplementable phrasing

Three sources describe the timer differently:

1. **§7.2 prose (line 474)**: "the 5s `missing-global-config` timer **starts at 'first non-None on_global_config callback'**, **NOT** at 'first on_local_config + 5s.'"
2. **§7.2 state diagram (lines 484-486)**: `starting / ok ──(cb sequence: on_global_config(None) repeats past 5s timeout from runner construction)──▶ missing-global-config`.
3. **YANG `runner-status` description (`yang.act:244-246`)**: "the lower layer's `/software-install/` subtree didn't arrive within 5s of first config callback".

Each phrasing has a different anchor (first non-None global cb / runner construction / first config cb) and a different exit condition. None of the three is a complete spec.

Worse, phrasing #1 is **functionally incorrect**: a real layer-topology error returns `None` forever (because `merge_subscription_tree` returns `None` while no `LayerSharedSubscription.latest` is non-None — `ttt.act:436-445`), which means the cb never fires with non-None data, which means the timer (per phrasing #1) never starts, which means the runner never transitions to `missing-global-config`. That is precisely the alarm condition the timer exists to surface.

There is also a real subscription-delivery semantics issue under all three phrasings. Tracing `Layer._add_owner_spec` → `_sub_tick` → `_owner_publish` (`ttt.act:713-722, 689-711, 663-672`):

- The synchronous `_sub_tick` fires when the first owner declares a spec.
- It calls `get_data(filt)`, sets `LayerSharedSubscription.latest`, then `_owner_publish` does `merged = merge_subscription_tree(...)`.
- For a fresh new owner, `last_merged = None`. If `merged` is also `None`, the comparison `merged != owner.last_merged` is **False**, so `owner.cb` is **not called**.
- Therefore the runner's `on_global_config` cb does **not** receive a synchronous `None` at startup; it just doesn't fire at all until a sample produces a non-None merged tree. v5.1's prose ("delivering `None`") is based on a slightly inaccurate model of `_owner_publish`.

A workable spec needs three things together: (a) an unambiguous timer anchor that always exists (e.g. "T0 = runner construction" or "T0 = first call to `on_local_config`"); (b) acknowledgment that a wrong topology may produce no callbacks at all, so the timer fires the alarm even with zero callbacks; (c) a grace large enough to absorb at least one full subscription period plus `Layer.load_from_db()` latency. Pinning `period=5.0` only covers (c) if the timer is also ≥ one period.

**Recommendation.** Pick one algorithm and write it as pseudocode in §7.2; mirror the same semantics in the YANG description. Suggested shape (no `await`, runner-internal):

```
# At runner construction:
after MISSING_GLOBAL_CONFIG_GRACE: self._check_global_config()

action def on_global_config(merged, err):
    if merged is not None and contains_software_install_subtree(merged):
        global_config_cache = parse(merged)
        global_config_seen = True
        if runner_status in {STARTING, MISSING_GLOBAL_CONFIG}:
            runner_status = OK
            self._publish_oper()

action def _check_global_config():
    if not global_config_seen:
        runner_status = MISSING_GLOBAL_CONFIG
        self._publish_oper()
```

with `MISSING_GLOBAL_CONFIG_GRACE` ≥ `subscription_period + load_from_db_budget` (suggest 15s). Then update YANG `runner-status` enum description to match: "the runner did not observe `/software-install/` in a non-None merged tree within `MISSING_GLOBAL_CONFIG_GRACE` of construction."

---

## MEDIUM 1 — §4.1 `create-request` is "(unchanged)" but §3.7 references reconciliation rules that live there

§3.7 (line 299) says: "the §4.1 reconciliation rule becomes: 'if the trigger's `request-generation` value equals the current request's `materialized_by_request_generation`, the current request already corresponds to this trigger — no-op.'"

§4.1 in the doc is literally just the heading `### 4.1 create-request (unchanged)` followed by no body. The actual reconciliation algorithm — the centerpiece of the v5.1 idempotency-anchor fix per A1 / CR-5 H1 — is not in the doc. The Tier-A discussion in §3.7 is the closest thing, but it states the invariant; it does not give the pseudocode.

This is a documentation gap on the highest-risk crash-recovery path. A Phase 4 implementer must reconstruct the rule from §3.7's prose plus §15.5 #21 plus their own reading of `01-software-install-logic.md` §3.1. They will get it wrong in subtle ways (when does `materialized_by_request_generation` get set on requests that were materialized by *pack-change*, not by `request-generation`? what is the precedence between pack-equality, last-cancelled, and request-generation triggers?).

**Recommendation.** Replace `### 4.1 create-request (unchanged)` with concrete pseudocode covering:

- The three independent triggers for "materialize a new request": pack-change, last-cancelled forces new, `request-generation > last_request_generation`.
- The precedence among them.
- What value `materialized_by_request_generation` takes for each path (default `last_request_generation` when materialization is *not* request-generation-driven, vs the observed generation when it is).
- The §3.7 idempotency rule as a literal pseudocode `return` branch.

---

## MEDIUM 2 — §8.4 step ordering between `_stash_dynstate` and `on_local_config` is glossed over

§8.4 line 635 asserts: "by now both `_stash_dynstate(dynstate)` and `on_local_config(cfg, mem)` have fired".

The actual call order in `_TransformTransaction`'s recompute path:

1. `compute()` calls `transform_wrapper(merged, linked, memory, dynstate)` (`ttt.act:1942-1943`). Inside, `self.stash_cb(dynstate)` enqueues an action message to the runner's mailbox. The transform_wrapper returns synchronously.
2. The recompute machinery proceeds; eventually `_TransformTransaction.finalize` runs and **synchronously** calls `self.function.on_conf(self.get(), self.memory)` (`ttt.act:1989-1993`). `on_conf` calls the lambda returned by `act`, which is `lambda cfg, mem: runner.on_local_config(cfg, mem)` — that enqueues an `on_local_config` action message to the runner's mailbox.

Both messages land in the runner's mailbox in this order: `_stash_dynstate`, then `on_local_config`. With FIFO actor mailbox processing, `_stash_dynstate` is processed first and `dynstate_initialized` flips to True before `on_local_config` runs — fine.

But §3.6's `on_local_config` body (lines 273-278) has a fallback that says "if `not dynstate_initialized`, treat as fresh install":

```
action def on_local_config(cfg, mem):
    if not self.dynstate_initialized:
        self.dynstate_initialized = True
        self._check_restore_consistency()
    # ... reconciliation ...
```

If actor scheduling guarantees FIFO processing of in-mailbox actions, this fallback is dead code on the restore path (because `_stash_dynstate` is enqueued first). If it isn't guaranteed (or if the platform reroutes through different actors with different latencies — Acton's `proc def init_dynstate` chain crosses Transaction → TransformFunction → runner actor), there is a race where `on_local_config` could process before `_stash_dynstate`, and the fallback would treat an existing dynstate as "fresh install" — silently dropping the restored state.

**Recommendation.** Either (a) commit to the in-mailbox FIFO guarantee in §8.4 explicitly and remove the §3.6 fallback (the fallback exists only for the genuinely-fresh case, so distinguish that case differently — e.g. a separate `_signal_no_restore()` action sent on a `restore-empty` path); or (b) keep the fallback but require an explicit two-input barrier — `on_local_config` defers reconciliation until *either* `_stash_dynstate` has fired *or* a known "no dynstate to restore" signal has fired. Today the design relies on FIFO without saying so.

---

## MEDIUM 3 — Topology-error fail-loud needs an integration test, not just an oper enum

§2 / §7.2 makes the layer topology constraint explicit: `/software-install/` must be in the layer below the sw-install transform's host layer. The failure mode is "runner publishes `runner-status: missing-global-config`". That is a runtime signal an operator may or may not look at.

An app integrating sw-install can wire layers wrong with no compile-time and no startup error. The first signal is the runner-status oper enum after the watchdog fires. Without a test in the sw-install repo that asserts this signal in a deliberately-misconfigured topology, this regression is one app composition away.

**Recommendation.** §10 (Testing strategy) should explicitly require a `test_topology_misconfigured.act` that builds a layer stack without the `/software-install/` pass-through, asserts the runner reaches `runner-status=missing-global-config`, and serves as a copy-paste template for app integrators. This is cheap test infrastructure and closes the only "operator can shoot foot" surface in the design.

---

## LOW items

### LOW 1 — Stale "v5" / "round-5" tail framing

`docs/00-orientation.md` line 8 still says "(current: v5)"; `docs/02-sw-install-design.md` §16 is still titled "Round-5 review" with "Stop here for round-5 review"; `yang.act:26-27` revision text still leads with "v5 design — reflects round-4 review integration on top of v4." None block Phase 4 but the doc set is the implementation input — stale framing complicates "what's normative."

### LOW 2 — §3.3 IOS-XR diagnostic prose contradicts the YANG

§3.3 line 197-198: "v5 YANG models only the common+SROS subset." But the YANG includes `op-id-add/prepare/activate/commit` under IOS-XR `when` constraints (`yang.act:600-615`). The accurate phrasing: "v5 YANG models common diagnostics + SROS `rebooted` + IOS-XR `op-id-*`. IOS-XR `packages` and `reload-required`, plus Junos per-RE diagnostics, are deferred to Phase 6."

### LOW 3 — §8.7 still warns about Q2 even though §12 marks it resolved

§8.7 line 671: "`dev_registry.get(devname)` call shape (per CL4_8): one-line check during Phase 4 skeleton — verify whether returned `DeviceMgr` is value-typed-synchronous or async-future. … ❓Round-5 question Q2." But §12 line 710 marks Q2 resolved by precedent (`ttt.act:2071`, `testing.act:38`). Either remove the §8.7 warning or recast it as confirmation; do not leave a `❓` and a "verify" instruction next to a confirmed-synchronous API.

### LOW 4 — §7.1 helper comparison cites the wrong existing helper

§7.1 line 447: "Cannot reuse `_DeviceTransaction.devname_from_device_path` directly — that helper expects `path: PathKey`, but our `params.path` is a `PathElem` with a PathKey parent."

The closer-shaped existing helper is `devname_from_rfs_path` (`ttt.act:2042-2053`), which already walks ancestors searching for a `PathElem("rfs")`. The reasons sw-install can't reuse it are (a) it lacks the namespace check the design adds for hardening, and (b) it's not part of the public substrate API. The doc would be more useful citing 2042 here than 2206. Trivial doc-only nit.

### LOW 5 — `fn_holder: list[SwInstallTransform] = []` is an ad-hoc single-cell

§7.3 uses `list[T]` as a closure-shared mutable cell. It works but reads unidiomatically; a comment explaining why a list is used (no first-class `Cell[T]`/`mut[T]` and tuples are immutable — closures need a mutable container) would head off the inevitable code-review question. This is a Phase 4 implementation note, not a design defect.

---

## Verification of the 13 r5 integration items

Out of the items in `14-integration-r5.md`, after a fresh read against the live platform code:

| # | Item | Status |
|---|------|--------|
| 1 | §3.6 + §7.3 action-ref push | **Partially landed** — §7.3 correct; §3.6 reintroduces the cross-actor write (HIGH 1). |
| 2 | §3.7 Tier-A batching invariant + `materialized_by_request_generation` | Landed (concept). But the §4.1 reconciliation pseudocode that consumes this anchor is missing (MEDIUM 1). |
| 3 | §3.6 + §8.4 restore lifecycle prose | Landed; ordering matches `app.act:138-152` and `ttt.act:1942-1993`. The implicit FIFO assumption is a residual risk (MEDIUM 2). |
| 4 | §7.2 runner-status guard rephrase + period=5 + precedence + no-pack-bound | Period pinned ✓, precedence ✓, no-pack-bound documented ✓; **timer semantics inconsistent across prose, diagram, YANG (HIGH 2)**. |
| 5 | §7.3 factory skeleton | Landed. |
| 6 | §7.1 devname helper hardening | Landed; only nit is which existing helper to compare against (LOW 4). |
| 7 | §7.1 `PathContainer` → `PathElem` | Landed. |
| 8 | §3.6 transform_wrapper returns `None` for memory | Landed (`return (gdata.Container(), None)`). |
| 9 | §15.5 +3 entries (runner-status, action→generation, no-pack-bound) | Landed (#20, #21, #22). |
| 10 | §12 mark Q1+Q2 resolved | Landed; but §8.7 still has stale Q2 prose (LOW 3). |
| 11 | §14 platform observation: ttt.Transform `lower` kwarg dead code | Landed in §14.1 — confirmed against `ttt.act:1907-1914`. |
| 12 | YANG stale `scp-port survives` comment + §3.3 → §3.4 reference | Both landed. |
| 13 | ADR duplicate `## Context` heading | Landed (single heading at line 14). |

Architectural claims I re-verified positive against the live source:

- `_RFSTransaction.finalize` empty-output suppression at `ttt.act:2151-2158` is real — the `if not (isinstance(output, gdata.Container) and not output.presence and len(output.children) == 0)` guard would silently swallow sw-install's empty-Container output. Plain `Transform` is the correct workaround.
- `RFSFunction.init_dynstate` at `ttt.act:2184-2185` constructs `TransformActorParams(path, update_dynstate, update_oper, dev)` with no `lower` argument — confirms sw-install needs plain `Transform`, where `init_dynstate` does pass `lower` (`ttt.act:2011-2012`).
- `ttt.Transform`'s outer `lower=` kwarg at `ttt.act:1907` is shadowed by `_create_transform_node(path, lower)` at line 1908. §14.1's observation is accurate.
- `DeviceRegistry.get(name) -> DeviceMgr` at `device.act:77-81` is synchronous (`def`, not `proc def`/`action def`); precedent at `_RFSTransaction.__init__` (`ttt.act:2071`) and `testing.act:38`. §12 Q2 correctly resolved.
- `ttt.act:2042-2053` `devname_from_rfs_path` is the closer-shaped existing helper (LOW 4).
- The yang.act augment header (lines 192-208) is the v5-corrected nested rationale; CR5_4 fixed cleanly.
- ADR `## Context` is single (line 14); CR5_5 fixed.

Architectural claims that did not survive scrutiny:

- The "subscription tick fires synchronously with `None`" model in §7.2's reasoning is slightly off — `_owner_publish` does not deliver `None` at startup; it doesn't deliver anything until `merge_subscription_tree` returns non-None. This affects HIGH 2's algorithm choice (the "first non-None on_global_config" anchor cannot detect "missing forever").

---

## Convergence with Codex r6 (`15-codex-design-r6.md`)

I read codex r6 only after writing the body of this review. Strong convergence, with both reviewers independently finding the same two HIGHs:

- Codex HIGH 1 ↔ this review's HIGH 1 (action-ref stash dual install).
- Codex HIGH 2 ↔ this review's HIGH 2 (timer semantics inconsistency).
- Codex MEDIUM 1 (§8.4 ordering / restore-callback barrier) ↔ this review's MEDIUM 2.
- Codex MEDIUM 2 (§4.1 reconciliation pseudocode missing) ↔ this review's MEDIUM 1.
- Codex MEDIUM 3 (topology footgun needs a test) ↔ this review's MEDIUM 3.
- Codex LOW 1 (stale v5 / round-5 tail text) ↔ this review's LOW 1.
- Codex LOW 2 (§3.3 IOS-XR contradiction) ↔ this review's LOW 2.
- Codex LOW 3 (§8.7 stale Q2 warning) ↔ this review's LOW 3.

This review adds, beyond codex r6:

- LOW 4: §7.1 cites `_DeviceTransaction.devname_from_device_path` where `devname_from_rfs_path` is the closer-shaped reference.
- LOW 5: `fn_holder: list[T] = []` as a single-cell deserves an inline justification comment.
- A more explicit grounding for HIGH 2: the `_owner_publish` traceback showing why the "synchronous `None` callback" model is wrong, and why "first non-None" cannot detect "missing forever".

No divergences in conclusions. Convergent r5→r6 narrowing pattern: 3+3 HIGHs at r5 → 2 HIGHs at r6, both reviewers, both doc-precision items. No substrate redesign required.

---

## Recommendation

Do a **v5.2 doc/skeleton pass** before Phase 4 implementation, addressing:

1. (HIGH 1) Make stash-callback installation single-owner. Drop `transform_fn` from `DeviceRunner`; install in `act` only. Reword §3.6 prose.
2. (HIGH 2) Replace the prose / diagram / YANG description of the `missing-global-config` watchdog with a single concrete algorithm, anchored at runner construction (or first `on_local_config`) with a grace ≥ subscription period. Mirror in `yang.act` description.
3. (MED 1) Replace `### 4.1 create-request (unchanged)` with concrete reconciliation pseudocode that consumes `materialized_by_request_generation`.
4. (MED 2) Either commit explicitly to FIFO action-mailbox ordering and drop §3.6's fallback, or add an explicit barrier so `on_local_config` waits for either `_stash_dynstate` or a `restore-empty` signal.
5. (MED 3) Add `test_topology_misconfigured.act` to §10 as a required Phase 4 test.
6. (LOW 1-5) Cleanups.

After v5.2 lands these — none of which is architectural — Phase 4 is unblocked. **No platform prerequisite has emerged in r6.** The convergence pattern (HIGH count 6 → 5 → 3+3 → 2+2) suggests v5.2 is most likely the lock-in point.
