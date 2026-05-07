# Round-7 Codex Design Review

Reviewer: Codex
Date: 2026-05-07

Scope read in order:

1. `docs/00-orientation.md`
2. `docs/01-software-install-logic.md`
3. `docs/02-sw-install-design.md` v5.2
4. `docs/adr/cli-driver.md`
5. `stratoweave/sw-install/src/sw_install/yang.act`

Then sanity-checked against live Stratoweave:

- `stratoweave/stratoweave/src/stratoweave/ttt.act`
- `stratoweave/stratoweave/src/stratoweave/app.act`
- `stratoweave/stratoweave/src/stratoweave/device.act`
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act`
- `stratoweave/sorespo/src/sorespo/rfs.act`
- `stratoweave/sorespo/src/sorespo/sysspec.act`

I formed the assessment before reading prior reviews, then checked the round-6 findings in `docs/reviews/15-codex-design-r6.md`, `docs/reviews/16-claude-design-r6.md`, and the integration notes in `docs/reviews/17-integration-r6.md`.

## Verdict

Not quite lock-in as written. v5.2 lands the round-6 architectural fixes correctly in the main sections: factory-owned `stash_cb`, empty-restore signal via `_stash_dynstate(None)`, FIFO mailbox commitment, concrete missing-global-config watchdog, required topology-misconfigured test, corrected IOS-XR projection prose, and updated YANG runner-status text.

However, the new §4.1 reconciliation pseudocode has a real ordering bug: the `materialized_by_request_generation` short-circuit runs before pack-change detection, so a previously consumed explicit generation can suppress a later pack-content change. That is a correctness issue in the lock-in algorithm, not just wording. Fixing it is local: move/narrow the idempotency check so pack-change still fires when `target_pack != last_pack_snapshot`.

After that one algorithm fix and a small stale-snippet cleanup, I would green-light Phase 4.

## Findings

### HIGH 1 - §4.1 idempotency short-circuit suppresses pack-change requests

`docs/02-sw-install-design.md` §4.1 now starts reconciliation with:

```acton
if (self.dynstate.current is not None and
    cfg_request_gen > 0 and
    self.dynstate.current.materialized_by_request_generation == cfg_request_gen):
    return
```

This check runs before Trigger B (`target_pack != self.dynstate.last_pack_snapshot`). That means a normal sequence can lose an intended request:

1. Operator sets `request-generation=42`; runner materializes request A with `materialized_by_request_generation=42` and `last_request_generation=42`.
2. Later, the software-pack definition or selected pack changes to target pack B, with `request-generation` still 42.
3. Reconciliation sees `current.materialized_by_request_generation == cfg_request_gen` and returns before comparing `target_pack` with `last_pack_snapshot`.
4. The pack-change request is never materialized.

This conflicts with the stated three-trigger model and with the original Python behavior where pack-data changes create a new request even without an explicit action/generation bump.

Recommendation: do not use `materialized_by_request_generation` as a blanket top-level return. Make the algorithm:

- Trigger A first: if `cfg_request_gen > last_request_generation`, materialize for that generation.
- Trigger B next: if `target_pack != last_pack_snapshot`, materialize for pack-change, even when `cfg_request_gen` equals the current request's materialized generation.
- Use the materialized-by anchor only to suppress duplicate handling of the explicit-generation path, and only when the target pack snapshot already matches the current request / last snapshot.

Given the §3.7 Tier-A batching invariant, the post-commit crash case already restores `last_request_generation=42` together with `current.materialized_by_request_generation=42`, so Trigger A will not refire. The top-level short-circuit is stronger than needed and blocks a valid Trigger B.

### MEDIUM 1 - §7.2 still shows the obsolete `transform_fn` DeviceRunner constructor shape

The main v5.2 factory skeleton in §7.3 is correct: `DeviceRunner` is constructed without a transform-function back-reference, and the `act` callback installs `fn.stash_cb = runner._stash_dynstate` from the transaction-owned side.

But §7.2 still has an older snippet:

```acton
runner = DeviceRunner(
    params.path, params.update_oper, params.update_dynstate,
    dev_registry.get(devname),
    transform_fn,             # v5.1: for stash_cb installation
    ...
)
transform_fn.stash_cb = runner._stash_dynstate
```

This contradicts §3.6, §7.3, and §8.2, which all correctly say the runner has no `transform_fn` parameter. It is probably stale illustrative text, but it is in the "Global config subscription + runner-status guard" section, so implementers may copy it.

Recommendation: replace the §7.2 snippet with the §7.3 shape or delete it and point to §7.3 as the single source of truth for construction.

### LOW 1 - §2 still says the topology watchdog is 5 seconds

`docs/02-sw-install-design.md` §2 says wrong topology yields `missing-global-config` "after a 5-second startup window." §7.2 and `yang.act` now correctly define `MISSING_GLOBAL_CONFIG_GRACE = 15.0` and describe a 15s construction-time watchdog.

Recommendation: update §2 to "15-second default grace" or just refer to `MISSING_GLOBAL_CONFIG_GRACE`.

### LOW 2 - `_signal_no_restore` comments are stale after the v5.2 empty-restore fix

§3.6's code comments say `_stash_dynstate(None)` may be called "via `_signal_no_restore`" and `on_local_config` says the empty-restore path is handled via `_signal_no_restore (§3.6.1)`. But §3.6.1 says the opposite: there is no separate signal; `transform_wrapper` calls `_stash_dynstate(stashed=None)` during forced recompute when no dynstate was restored.

Recommendation: remove `_signal_no_restore` from the comments. The intended empty-restore mechanism is now simpler and should stay that way.

## Round-6 Fix Verification

Landed correctly:

- Factory-owned `stash_cb` install is the canonical shape in §3.6 / §7.3 / §8.2; no runner back-reference is needed.
- Empty restore now sends `_stash_dynstate(None)`, and the design explicitly relies on FIFO actor mailbox ordering before `on_local_config`.
- §3.7 preserves the Tier-A batching invariant and correctly moves IOS-XR `op_id_*` to step-boundary persistence.
- §7.2 missing-global-config watchdog is now one concrete algorithm: schedule at runner construction, fire unconditionally after 15s, and flip back to `ok` if `/software-install/` appears later.
- YANG `runner-status` mirrors the 15s watchdog and precedence rules.
- §10 requires `test_topology_misconfigured.act`.
- §3.3 IOS-XR diagnostic projection prose now matches YANG for `op-id-*`.
- §7.1 cites `devname_from_rfs_path`, which is the closer live helper.
- §8.7 Q2 stale warning was removed; `DeviceRegistry.get()` is synchronous by live code.
- Orientation and revision framing are updated to v5.2 / round 7.

Still needs correction:

- §4.1's top-level idempotency return must not run before pack-change detection.
- §7.2's construction snippet still carries the pre-v5.2 `transform_fn` parameter.
- Two stale timing/signal comments remain (`5-second` and `_signal_no_restore`).

## Platform Sanity Check

The core substrate assumptions still hold against live code:

- `StartupBootstrap._run` loads LMDB before forced recompute (`app.act`), so `transform_wrapper(..., dynstate)` can see restored `_TransformTransaction.dynstate`.
- Plain `ttt.Transform` remains the right substrate. `_RFSTransaction.finalize` suppresses `on_conf` for empty output, and `RFSFunction.init_dynstate` does not pass `lower`; sw-install needs empty downward output plus `params.lower`.
- `Layer._owner_publish` may not invoke the callback at all when the merged subscription tree is `None`, so the construction-time watchdog is the right fix for "missing forever."
- `DeviceRegistry.get(name) -> DeviceMgr` is synchronous.
- `SubscriptionSpec(period=5.0)` is valid: `gdata.SubscriptionSpec` normalizes float seconds to nanoseconds before `ttt.layer_subscription_delay` divides back to seconds.

## Recommendation

Do a v5.2.1 doc-only patch before Phase 4:

1. Fix §4.1 reconciliation ordering so pack-change is not masked by `materialized_by_request_generation`.
2. Remove the stale `transform_fn` constructor snippet from §7.2.
3. Clean the `5-second` and `_signal_no_restore` stale comments.

No new platform prerequisite is needed.
