# Round-6 Codex Design Review

Reviewer: Codex
Date: 2026-05-07

Scope read in order:

1. `docs/00-orientation.md`
2. `docs/01-software-install-logic.md`
3. `docs/02-sw-install-design.md` v5.1
4. `docs/adr/cli-driver.md`
5. `stratoweave/sw-install/src/sw_install/yang.act`

Then sanity-checked against live Stratoweave:

- `stratoweave/stratoweave/src/stratoweave/ttt.act`
- `stratoweave/stratoweave/src/stratoweave/device.act`
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act`
- `stratoweave/sorespo/src/sorespo/rfs.act`
- `stratoweave/sorespo/src/sorespo/layers/t_2.act`

I read prior round-5 reviews only after forming the assessment above, then checked whether the 13 integration items from `docs/reviews/14-integration-r5.md` landed.

## Summary

v5.1 is close. The main architecture still looks sound: plain `ttt.Transform` is the right substrate on today's platform, nested `scp-port` is correct, `dev_registry.get()` is synchronous by precedent, the post-restore `transform_wrapper(..., dynstate)` path exists, and the Tier-A batching invariant is the right recovery model for fire-and-forget `update_dynstate`.

I would not quite green-light Phase 4 from this doc set as-is, because two v5.1 fixes are internally inconsistent in ways an implementer can copy incorrectly:

1. the action-ref stash is described in two different ownership shapes, one of which reintroduces the cross-actor mutable-field concern that v5.1 was meant to remove;
2. the `missing-global-config` timer is described as starting at first non-`None` global callback, but the state diagram and YANG still describe a timer from runner construction / first config callback.

These are localized doc/skeleton fixes, not a substrate redesign.

## Findings

### HIGH 1 - Action-ref stash still has contradictory ownership and one version is actor-smelly

`docs/02-sw-install-design.md` §3.6 says the runner installs `transform_fn.stash_cb = self._stash_dynstate` inside the `DeviceRunner` actor body (lines 255-265). §7.2 / §7.3 then show the factory/`act` callback installing `fn.stash_cb = runner._stash_dynstate` after constructing the runner (lines 456-463 and 559-562).

Those are materially different. The §7.3 factory skeleton is the safer shape: the `_TransformTransaction` side constructs the function instance, constructs the runner actor ref, and writes the function object's `stash_cb` field before returning from `act`. Later, `_TransformTransaction.compute()` reads that field in `transform_wrapper()` (`ttt.act:1942-1943`) and calls the runner action. The runner never reads or writes transform-owned mutable state.

The §3.6 `DeviceRunner` body version instead has the runner actor mutating a field that `_TransformTransaction` later reads. That is not as bad as v5's pull-based `stashed_dynstate` field, but it is still cross-actor mutation of a shared class instance. It also conflicts with the concrete factory skeleton, so Phase 4 has two examples to choose from.

Recommendation: make §3.6 match §7.3. Remove `transform_fn` from the `DeviceRunner` constructor if it is only "for installing stash_cb", and install `fn.stash_cb = runner._stash_dynstate` exactly once in the `act` callback / factory closure. If the constructor keeps `transform_fn` for some other reason, say so, but do not have the actor body mutate it.

### HIGH 2 - `missing-global-config` timer semantics are still inconsistent

The prose at §7.2 line 474 says the 5s timer starts at "first non-None on_global_config callback". The state diagram at lines 479-486 says `on_global_config(None)` repeats past "5s timeout from runner construction" and then transitions to `missing-global-config`. The YANG description still says `/software-install/` did not arrive "within 5s of first config callback" (`yang.act:244-246`).

Those cannot all be true:

- If the timer really starts at first non-`None`, a wrong topology that returns `None` forever never starts the timer and never becomes `missing-global-config`.
- If the timer starts at runner construction, the earlier round-5 race returns: the synchronous subscription tick can deliver `None` before restore (`ttt.act:713-722`), and the next sample is period-delayed (`ttt.act:680-687`).
- If it starts at first local config callback, the YANG has the old v5 behavior that v5.1 says it fixed.

The design needs one precise algorithm. A workable version would be: ignore the first synchronous `None` before restore/local reconciliation, pin subscription period to 5s, and after the runner is initialized with local config, require a non-`None` merged tree containing `/software-install/` by the next subscription period plus a small grace. Another workable version is a construction-time watchdog that is explicitly greater than one full subscription period. But "first non-None" alone cannot detect "missing forever".

This should be fixed in both §7.2 and the YANG `runner-status` description.

### MEDIUM 1 - §8.4 overstates callback ordering between `_stash_dynstate` and `on_local_config`

The lifecycle ordering is mostly corrected, but §8.4 line 635 says "by now both `_stash_dynstate(dynstate)` and `on_local_config(cfg, mem)` have fired" and the runner has restored dynstate.

`transform_wrapper()` calls `stash_cb(dynstate)`, which queues an action to the runner. `_TransformTransaction.finalize()` then synchronously calls `function.on_conf(...)` in the same transaction finalization path (`ttt.act:1989-1993`). Depending on Acton action scheduling, `on_local_config` can plausibly run before the queued `_stash_dynstate` action is processed. §3.6's code already anticipates this by letting `on_local_config` initialize empty dynstate if `dynstate_initialized` is false (lines 273-278), but that fallback would be wrong if a restored dynstate action is merely queued behind it.

The runner needs an explicit two-input barrier: do not reconcile until local config has arrived and either restored dynstate has arrived or the implementation can prove no restored dynstate exists. If the platform cannot signal "no dynstate was restored", the stash action should be sent even with `None` during the forced recompute, and `on_local_config` should wait for that marker rather than assuming fresh install.

### MEDIUM 2 - `request-generation` anchor is added, but §4.1 does not actually define the reconciliation rule

`RequestState.materialized_by_request_generation` is in §3.2, and §3.7 says "the §4.1 reconciliation rule becomes..." (line 299). But §4.1 is still just a heading, "`create-request` (unchanged)", with no concrete rule.

This is a documentation gap around one of the highest-risk crash-recovery semantics. Phase 4 implementers should not have to reverse-engineer the intended rule from the Tier-A notes. Put the actual `request-generation` reconciliation pseudocode in §4.1: compare pack-change idempotency separately from explicit force-new generation; suppress duplicate materialization when current/history already records `materialized_by_request_generation == observed_generation`; define what value is stored for implicit pack-change-created requests.

### MEDIUM 3 - The global-config subscription topology remains a footgun, even if documented

The design requires `/software-install/...` to be authored above, transformed through the host layer, and visible in the lower layer because `params.lower` points below the sw-install transform. The doc is now explicit, but the architecture still couples a per-device observer to a lower-layer pass-through sentinel. A missing pass-through does not fail at construction; it only becomes an oper status after a timer.

This is acceptable for Phase 4 if the guard is fixed, but tests must cover the topology failure. I would add at least one integration test where the host app wires the sw-install transform without passing `/software-install/` into the lower layer and asserts `runner-status=missing-global-config`. Without that, this will regress easily during app integration.

### LOW 1 - v5.1 cleanup did not fully update stale version / review tail text

Several stale references remain:

- `docs/00-orientation.md` still says the design doc is "current: v5".
- `docs/02-sw-install-design.md` §16 is still "Round-5 review" and says "Stop here for round-5 review" even though this is v5.1 pending round 6.
- `stratoweave/sw-install/src/sw_install/yang.act` revision text still says "v5 design - reflects round-4 review integration".

Not blocking, but this doc set is being used as implementation input; stale review framing makes it harder to tell what is normative.

### LOW 2 - IOS-XR diagnostic projection prose contradicts the YANG

`docs/02-sw-install-design.md` §3.3 says IOS-XR `op-id-*` diagnostics are modeled, then immediately says "v5 YANG models only the common+SROS subset" (lines 197-198). The YANG does include `op-id-add`, `op-id-prepare`, `op-id-activate`, and `op-id-commit` under IOS-XR `when` constraints. The intended statement seems to be: v5 YANG models common + SROS `rebooted` + IOS-XR `op-id-*`, while IOS-XR `packages` / `reload-required` and Junos per-RE diagnostics are deferred.

### LOW 3 - §8.7 still carries a resolved Q2 warning

`docs/02-sw-install-design.md` §8.7 line 671 says Phase 4 should verify whether `dev_registry.get(devname)` is synchronous and references round-5 Q2. §12 marks Q2 resolved, and live `DeviceRegistry.get(name: str) -> DeviceMgr` is synchronous by existing use (`device.act:77-81`, `_RFSTransaction.__init__` at `ttt.act:2071`). Remove the stale warning.

## Verification Of Round-5 Integration Items

Landed correctly:

- §3.7 adds the Tier-A batching invariant.
- `materialized_by_request_generation` exists in `RequestState`.
- §3.6 / §8.4 now correctly say actor construction happens before `Layer.load_from_db()`.
- Plain `ttt.Transform` rationale is still valid against live `_RFSTransaction.finalize` and `RFSFunction.init_dynstate`.
- `devname_from_swi_path` now checks the `/sw-rfs:rfs` parent namespace and shape.
- §7.3 has a concrete closure-shared factory skeleton.
- `transform_wrapper` returns `(gdata.Container(), None)`.
- Q1 and Q2 are marked resolved in §12.
- ADR duplicate `## Context` heading is gone.
- YANG stale `scp-port survives unbinding` comment is fixed.
- YANG request-history reference is corrected to §3.4.
- §15.5 adds runner-status, action-to-generation deviation, and no-pack-bound steady state.
- `ttt.Transform lower=` dead-code observation is present in §14.1.

Partially landed / needs correction:

- Action-ref stash: concept landed, but §3.6 and §7.3 show conflicting installation ownership.
- Runner-status guard: period pinned and "global config seen" defined, but timer start semantics contradict themselves and the YANG.
- No-pack-bound steady state is documented, but it also highlights that `starting` has two meanings: boot in progress and no configured work. That is acceptable, but tests and operator docs should use the "empty request list" disambiguator consistently.

## Recommendation

Do a small v5.2 doc/skeleton cleanup before Phase 4 implementation:

1. Make stash callback installation single-owner and factory-installed.
2. Rewrite the `missing-global-config` watchdog as precise pseudocode and update YANG text.
3. Add the actual `request-generation` reconciliation pseudocode in §4.1.
4. Clarify the restore callback barrier so local config cannot race ahead of queued restored dynstate.
5. Clean the stale v5 / round-5 tail text.

After those fixes, I would green-light Phase 4. No new platform prerequisite is needed.
