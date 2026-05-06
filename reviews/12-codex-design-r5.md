# Round-5 Codex Design Review

Reviewer: Codex
Date: 2026-05-07

Scope read in order:

1. `docs/00-orientation.md`
2. `docs/01-software-install-logic.md`
3. `docs/02-sw-install-design.md` v5
4. `docs/adr/cli-driver.md`
5. `stratoweave/sw-install/src/sw_install/yang.act` v5

Then sanity-checked against:

- `stratoweave/stratoweave/src/stratoweave/ttt.act`
- `stratoweave/stratoweave/src/stratoweave/device.act`
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act`
- `stratoweave/sorespo/src/sorespo/`

## Summary

v5 is materially better than v4. The switch to plain `ttt.Transform`, nested `scp-port`, `proc` callback types, `get_modules()` tuple shape, and the removal of Phase 4 platform prerequisites are directionally right.

I am not ready to call the design implementation-ready without a v5.1 doc pass. The highest-risk remaining issue is §3.7: the new "idempotent-on-re-fire" framing is correct as the only implementable shape on today's fire-and-forget `update_dynstate`, but the current text overclaims idempotency for `request-generation` and maybe for start-side effects. There are also two lifecycle/topology details that are close but still misleading enough to send an implementer down the wrong path.

## Findings

### HIGH 1 - §3.7 overclaims Tier A idempotency, especially for `request-generation`

`docs/02-sw-install-design.md` §3.7 says:

> `last_<trigger>_generation` values - publish before the trigger's work begins. Re-fire after crash is safe because the work itself is idempotent: re-creating a request with the same pack returns the same id (§4.1); re-starting an unprocessed request is a no-op (§4.2).

That is not true for every trigger covered by `last_<trigger>_generation`.

The v5 YANG says `control/request-generation` explicitly "force[s] a new request even if the pack assignment is unchanged" and is required for cancelled reactivation. That makes it non-idempotent by design. A crash after the runner materializes request N but before the generation consumption is durably visible can re-fire the same `request-generation` and materialize request N+1. The §4.1 "same pack returns same id" rule only applies to the Python-style implicit create-request idempotency path, not the explicit v5 generation-forces-new-request path.

This is not just a wording nit. It is the exact failure mode §3.7 is trying to classify. If the design wants Phase 4 to ship without an ack-capable `update_dynstate`, it needs a concrete re-fire rule for `request-generation`, such as:

- treat a consumed request-generation as part of the same dynstate mutation that creates `current`/moves prior current to history, then on reconcile recognize "current already corresponds to this generation";
- store a per-request `materialized_by_request_generation` anchor and suppress duplicates;
- accept duplicate requests as a conscious deviation and say so in §15.5; or
- move the ack variant of `update_dynstate` back into Phase 4 prerequisites.

The same paragraph should also be tightened for `start-generation`. "Re-starting an unprocessed request is a no-op" is only safe before a side-effecting step has been issued. If the crash window is "device side effect issued, dynstate write not durable", re-fire safety depends on the step's own device-side pre-check/idempotency. That may be true for many steps, but it should be stated as a Step contract requirement, not implied by the generation counter alone.

### HIGH 2 - §3.6/§8.4 describe the restore callback order incorrectly

The stash mechanism itself is plausible, but §8.4's startup sequence is wrong:

1. Platform restores transform dynstate from lmdb.
2. Forced post-restore recompute fires `transform_wrapper(...)`.
3. The `act` callback fires; runner constructed with default-empty dynstate.
4. First `on_local_config(...)` reads the stash.

Live `ttt.act` does not work in that order. `ttt.Transform(...)` constructs `_TransformTransaction`, constructs `self.function = function(log_handler)`, then immediately calls `transform.init_dynstate(node, act)` during rootgen if `act` is present. `_TransformTransaction.init_dynstate(...)` calls `self.function.init_dynstate(...)`, and `TransformFunction.init_dynstate(...)` calls the supplied `act(...)`. This all happens during `Layer(...)` construction, before `Layer.load_from_db()`.

The real order under `app.StartupBootstrap` is:

1. `Layer(...)` rootgen constructs the `TransformFunction` and calls `act(...)`; the runner is constructed before restore.
2. `Layer.load_from_db()` restores `_TransformTransaction.dynstate`.
3. `cfs.newsession().recompute(..., force=True)` calls `transform_wrapper(..., dynstate)` and stashes the restored dynstate.
4. `_TransformTransaction.finalize(...)` calls `function.on_conf(self.get(), self.memory)` if `self.running` is non-empty.
5. The already-constructed runner's first `on_local_config(...)` reads `transform_fn.stashed_dynstate`.

This distinction matters for implementation. The actor is not constructed after the stash exists; it must tolerate being alive with empty in-memory dynstate until the first post-restore `on_local_config`. Also, the design should explicitly say the stash path relies on the app-level `StartupBootstrap` recompute after `load_from_db()`, not on `Layer.load_from_db()` alone. `Layer.load_from_db()` only restores records; it does not itself recompute.

### HIGH 3 - The `runner-status` guard can still silently pass or false-alarm unless "global config seen" is defined precisely

The v5 guard is the right idea, but §7.2 is underspecified in a way that matters. `Layer.declare_subscriptions(...)` immediately samples and calls the owner callback. If the lower layer lacks `/software-install`, `get_data(filter)` can return `None` and the callback still fires. If implementation treats "callback fired" as "global config arrived", the footgun remains silent.

Conversely, `/software-install/enabled` has a YANG default of `false`, but gdata does not automatically materialize schema defaults in an absent subtree. A valid deployment that has no explicit `/software-install` edits yet may be indistinguishable from the wrong topology unless the app always creates/passes a sentinel container.

v5 should define the guard in data terms, not callback terms:

- What exact shape counts as "global config seen"? Non-`None` `/software-install` container? A container containing at least one configured `software-pack`? Any explicit global leaf?
- Does an absent global subtree mean topology error, or just "feature not configured yet"?
- Is the app integration required to materialize/pass through an empty `/software-install` container so the guard has a sentinel?

Without that, the 5s guard is not reliably fail-loud. It might mark `ok` on an empty callback, or mark `missing-global-config` on a valid default-only deployment.

### MEDIUM 1 - The concrete devname helper is still too weak and likely wrong for future path shapes

§7.1 now provides:

```acton
p = path.parent
if isinstance(p, ttt.PathKey):
    return p.name
```

For today's single-key `/sw-rfs:rfs[name]` list, `PathKey.name` is probably the list key string, so this likely works in the narrow case. But the integration note in `docs/reviews/11-integration-r4.md` asked for walking/verifying the ancestor whose parent is `/sw-rfs:rfs`; the v5 helper does not verify that. It will return the nearest parent list key even if the transform is later nested under another keyed list below `software-pack`.

The safer helper should mirror `ttt.devname_from_rfs_path(...)`: walk ancestors until a `PathKey` whose parent is a `PathElem` named `rfs` in the stratoweave-rfs namespace is found. Then return that key's `name` or, even better, read the `name` key leaf from `PathKey.keyvals` by the `sw-rfs:name` id. That avoids accidental dependence on encoded composite-key strings.

Also: the doc says `params.path` is a `PathContainer`; live `ttt.act` only has `PathRoot`, `PathElem`, and `PathKey`. Use `PathElem("software-pack")` terminology.

### MEDIUM 2 - `runner-status` needs precedence rules

The YANG enum combines lifecycle states with fault states:

- `missing-global-config`
- `restore-inconsistent`
- `paused-by-enabled`
- `waiting-for-device`
- `ok`
- `starting`

The design should define precedence. Example: after startup, global config is present but `enabled=false` and the device is not ready. Should the published status be `paused-by-enabled` or `waiting-for-device`? If dynstate is restore-inconsistent and global config is also missing, which wins? I would expect `restore-inconsistent` > `missing-global-config` > `waiting-for-device` > `paused-by-enabled` > `ok`, but the doc should specify this to avoid flicker and test ambiguity.

### MEDIUM 3 - §15.5 is missing at least one conscious deviation: generation triggers are not Python actions

§15.5 includes "Action-style return values replaced by per-device last-create-result and single-slot last-trigger-result", which covers the northbound surface partially. It should also explicitly call out the behavioral deviation that Python actions are per-call RPCs with immediate output, while v5 uses durable config generation counters sampled by reconciliation.

That difference drives several semantics that are not obvious to operators:

- repeated writes of the same generation are ignored;
- concurrent clients can race by writing the same or different generation values;
- `*-target-id` is sampled at observation time;
- backup/restore can make counters move backward relative to dynstate;
- `request-generation` is not equivalent to Python's idempotent `create-request` action because v5 says it forces a new request.

Some of this appears elsewhere, but §15.5 is the right consolidated place for it.

### MEDIUM 4 - YANG still contains stale `scp-port` prose

`stratoweave/sw-install/src/sw_install/yang.act` correctly places `scp-port` inside `software-pack`, but the augment comment above it still says:

> scp-port is a direct device-level setting (transport-related; survives unbinding the software-pack).

That contradicts the v5 decision and the leaf's own description, which says removing the pack also removes `scp-port`. Fix the comment before implementation starts; this is exactly the kind of stale design scar that causes future re-litigation.

### LOW 1 - The ADR has a duplicated `## Context` heading

`docs/adr/cli-driver.md` has two consecutive `## Context` sections. Not harmful, but clean it while touching docs.

### LOW 2 - Some section cross-references are stale

The YANG `request` description says "history retention policy (see design doc 02 §3.3)", but v5 retention is §3.4. Low severity, but this doc set is being treated as implementation input, so stale references are worth removing.

## Answers To The Specific Round-5 Questions

### Q1 - Does `transform_wrapper` stash work with current lifecycle?

Mostly yes, with the ordering correction above.

The live path is:

- `_TransformTransaction.restore()` loads `ATTR_DYNSTATE` into `self.dynstate`.
- `app.StartupBootstrap` calls `cfs.newsession().recompute(..., force=True)` after `load_from_db()`.
- `_TransformTransaction.compute()` calls `self.function.transform_wrapper(merged, linked, self.memory, self.dynstate)`, so the restored dynstate is visible to the function during recompute.
- `_TransformTransaction.finalize()` then calls `function.on_conf(...)` for non-empty running config, which can deliver the first local config callback to the runner.

Caveat: actor construction does not happen after the stash; the runner already exists. The design should describe that accurately. Also, if there is persisted dynstate but no persisted running config for the transform, `finalize()` will not call `on_conf` because `_TransformTransaction.finalize()` gates on `self.running`. That may be acceptable if absence of the `software-pack` presence container means no active runner, but it should be an explicit invariant.

### Q2 - Is `dev_registry.get(devname)` synchronous?

Yes. Live `DeviceRegistry.get(name: str) -> DeviceMgr` synchronously creates/returns the `DeviceMgr` actor reference. No async future/callback chain is needed for Phase 4.

### §7.1 - Deliberate departure + devname helper

The deliberate departure from `RFSTransform` is justified. Live `_RFSTransaction.finalize()` suppresses `on_conf` for empty `gdata.Container`, and `RFSFunction.init_dynstate()` passes `dev` but not `lower`. Plain `ttt.Transform` is the only current substrate that gives sw-install both `params.lower` and `on_conf` for observer-shaped empty output.

The devname helper needs the stronger ancestor check described above.

### §7.2 - Runner-status guard sufficiency

The guard is directionally sufficient only if v5 defines what counts as "global config seen" and what status wins when multiple non-ok states apply. As currently written, an implementation can accidentally count the immediate subscription callback with `None` as success, or false-alarm on a valid default-only global config.

### §15.5 - Consolidated deviations

The section is useful and mostly complete, but add the generation-counter-vs-action semantics explicitly, and consider adding the accepted duplicate-risk if §3.7 keeps `request-generation` without a durable idempotency anchor.

## Recommendation

Do a v5.1 doc-only pass before Phase 4 implementation:

1. Fix §3.7's Tier A claims and add a concrete idempotency anchor for `request-generation`.
2. Correct §3.6/§8.4 restore ordering and name the app-level post-restore recompute dependency.
3. Define `runner-status` guard data semantics and precedence.
4. Strengthen the devname helper.
5. Clean stale YANG/ADR references.

After those changes, I think Phase 4 can start without platform changes.
