# Round-4 Codex Design Review

Reviewer: Codex  
Date: 2026-05-07  
Scope: fresh read of `00-orientation.md`, `01-software-install-logic.md`, `02-sw-install-design.md` v4, `docs/adr/cli-driver.md`, `stratoweave/sw-install/src/sw_install/yang.act`, then sanity-check against live StratoWeave `ttt.act`, `device.act`, `adapters/adapter.act`, and the sorespo per-device transform pattern.

## Executive summary

v4 fixes several round-3 issues cleanly: switching sw-install to plain `ttt.Transform` is the right direction for an observer-only transform, `DeviceMgr.get_dmc()` is now framed as an existing API, the `/sw-rfs:rfs` placement narrative is much better, target-trigger feedback is no longer silent, and the control YANG is generally aligned with the design.

I still do **not** think this is ready for Phase 4 implementation. The remaining blockers are narrower than round 3, but they are load-bearing:

1. The proposed `TransformActorParams.dynstate` platform addition is not sufficient as described, because transform actors are initialized during layer construction, before `Layer.load_from_db()` restores LMDB state.
2. Tier A "persist before side effect" is not implementable with the current `update_dynstate(?gdata.Node) -> None` fire-and-forget callback.
3. Attaching the transform at `/sw-rfs:rfs[name]/software-pack` means the runner cannot see sibling `scp-port`, while the design expects file-transfer code to use it.
4. The global `/software-install` subscription topology is still too fragile for a fresh integrator; `params.lower.declare_subscriptions()` only samples the lower layer, not sibling/current-layer config.

I would resolve these before coding. The rest are mostly doc/YANG cleanup items.

## Findings

### HIGH 1 - `TransformActorParams.dynstate` does not solve restore-at-construction

`02-sw-install-design.md` §3.6 and §8.4 say the runner is constructed with `initial_dynstate = SwInstallDynstate.from_gdata(params.dynstate)`. Current startup order makes that impossible with only the proposed five-line field addition.

Live code:
- `Layer(...)` builds `root = rootgen(...)` immediately.
- `ttt.Transform(...)` calls `transform.init_dynstate(node, act)` while building that root.
- `StartupBootstrap._run()` calls `cfs.load_from_db()` later.
- `_TransformTransaction.restore()` decodes `ATTR_DYNSTATE` into `self.dynstate` during `load_from_db()`.
- The forced recompute happens only after restore.

So at actor construction time, `self.dynstate` is still `None`. Threading `self.dynstate` through `_TransformTransaction.init_dynstate()` only passes the pre-restore value. The v4 D3b fallback is actually closer to the current lifecycle, because `transform_wrapper(..., dynstate)` runs during the post-restore recompute and can see restored dynstate.

Fix options:
- Change platform lifecycle so actor initialization happens after restore, or add a post-restore actor callback such as `on_restored_dynstate(dynstate)`.
- Make `TransformActorParams.dynstate` a live getter/subscription rather than a constructor field.
- Prefer the D3b stash for Phase 4, but document it as the primary implementation unless the platform change also changes initialization order.

As written, §14 understates the prerequisite. This is not a five-line additive patch.

### HIGH 2 - Tier A persistence requires an acknowledgement API

§3.7 says Tier A fields "MUST persist before side effect" and that the runner blocks until the LMDB write completes. Current platform API does not give the runner any completion signal:

```acton
class TransformActorParams:
    update_dynstate: proc(?gdata.Node) -> None
```

`_TransformTransaction.update_dynstate()` mutates `self.dynstate` and calls `Session(...).recompute()` with no callback exposed to the transform actor. The actor cannot know whether the recompute committed, whether the LMDB write completed, or whether persistence failed.

This matters for exactly the fields §3.7 classifies as critical: generation high-water marks, `next_request_id`, retry scheduling, and future IOS-XR op IDs. "Persist before issuing RPC" is a good invariant, but it needs one of:

- `update_dynstate(dynstate, done)` / awaitable ack from the platform;
- a runner-owned persistence API separate from transform dynstate;
- weaker wording that says "publish dynstate update before side effect" and explicitly accepts crash windows.

If the design keeps Tier A semantics, add this to §14 as a Phase 4 platform prerequisite alongside the dynstate restore path.

### HIGH 3 - The transform cannot see sibling `scp-port`

v4 places `scp-port` at `/sw-rfs:rfs[name]/scp-port`, as a sibling of `/sw-rfs:rfs[name]/software-pack`, and wires the transform at `software-pack`:

```acton
ttt.Container({
    q("name"): ttt.Leaf(),
    q("scp-port"): ttt.Leaf(),
    q("software-pack"): swi_factory,
})
```

That means `on_local_config(cfg, mem)` receives only the `software-pack` container subtree. It does not receive the sibling `scp-port`. But §9.6 says `scp-port` is consumed by Phase 5+ `FileTransfer`, and the YANG description says the same.

This is a real topology bug. Pick one:

- Move `scp-port` inside `software-pack` if only sw-install uses it.
- Attach the transform at the `/sw-rfs:rfs[name]` container level so local config includes both `scp-port` and `software-pack`.
- Add an explicit sibling/current-layer read or subscription path and document it.

The current shape preserves `scp-port` across pack unbinding, but also hides it from the actor that needs it.

### HIGH 4 - Global config subscription is still underspecified

§7.2 correctly notes that `params.lower.declare_subscriptions()` reads from the lower layer. But the integration instructions still do not give a concrete working layer topology. §2 says apps add the YANG model to their lowest config layer and wire the transform there; a fresh reader can reasonably put `/software-install` and `/sw-rfs:rfs/software-pack` in the same host layer. In that case the per-device transform's `params.lower` will not see `/software-install`.

This needs a concrete example, not just a warning:

- Show which layer contains northbound `/software-install`.
- Show how that subtree is passed through to the lower layer that `params.lower` samples.
- Or make the transform subscribe/read from the current layer root via a platform addition.

Without this, the runner may start with local device assignment but no global pack library, no `enabled`, and no retry policy.

### MEDIUM 1 - `devname_from_path` cannot be copied from `devname_from_device_path`

§7 says the runner extracts the device name from `params.path`, "same pattern as `_DeviceTransaction.devname_from_device_path`." The live helper is:

```acton
def devname_from_device_path(path: Path):
    if isinstance(path, PathKey):
        return path.name
```

For a transform attached at `/sw-rfs:rfs[name]/software-pack`, `params.path` is a `PathElem` for `software-pack`, whose parent is the `PathKey` for the RFS entry. The helper cannot be reused directly. The design should specify a new helper that walks ancestors and verifies it found the `/sw-rfs:rfs` key, not any arbitrary list key.

### MEDIUM 2 - `waiting-for-device` polling uses the wrong `get_modules()` shape

§8.7 sketches:

```acton
modules = dev.get_modules()
if len(modules) > 0:
```

Live `DeviceMgr.get_modules()` returns `(dict[str, ModCap], ?str)`, and sorespo uses `dev.get_modules().0`. The design should use `modules = dev.get_modules().0`. Also, a real but disconnected adapter may still report an empty modset; readiness probably needs a clear rule based on `(modset, modset_id)` or capabilities, not just non-empty length.

### MEDIUM 3 - YANG is missing diagnostics promised by §3.3

§3.3 says IOS-XR diagnostic projections include `packages` and `reload-required`. The v4 YANG includes `op-id-add`, `op-id-prepare`, `op-id-activate`, and `op-id-commit`, but not `packages` or `reload-required`.

Either add those leaves with IOS-XR `when` constraints, or change §3.3 to say they are deferred to Phase 6 YANG changes.

### MEDIUM 4 - Junos diagnostic projection remains only described, not modeled

§3.3 lists Junos per-RE diagnostics: `version`, `boot-time`, `rebooted`. The YANG has only component-level common fields and no per-RE list. This is acceptable if Junos is truly Phase 6, but the doc should say the v4 YANG intentionally does not model Junos per-RE diagnostics yet.

### MEDIUM 5 - Conscious deviations list contradicts the ADR on CLI stubs

`02-sw-install-design.md` §9.7 and `docs/adr/cli-driver.md` now say Phase 4 should **not** ship per-method CLI stubs raising `NotImplementedError`. But §15.5 #10 still says:

> CLI strategy methods exist as stubs in Phase 4.

That stale deviation should be deleted or rewritten to "CLI strategy not selected in Phase 4; `DeviceOps` keeps a future `cli_session` boundary."

### MEDIUM 6 - `auto-execute-after-confirm` wording may accidentally bypass persisted start generation

§4.2 says `unprocessed -> processing` requires `start-generation > last_start_generation` OR `auto-execute-after-confirm = true` and confirmations are in place. §3.7 says `last_<trigger>_generation` values are Tier A and persist before trigger work begins.

For the auto-execute path there may be no `start-generation` to consume. The design should define the idempotency anchor for an auto-start caused by confirmation: e.g. a per-request `auto_started_after_confirm` flag or a bump to `generation_token` persisted before starting. Otherwise restart after confirmations could repeatedly auto-start.

### LOW 1 - Orientation overstates current platform support

`00-orientation.md` presents `TransformActorParams` as already carrying the right handles for sw-install, including dynstate restore-oriented behavior. For a fresh reader, this hides that v4 is conditional on a platform change and, per HIGH 1/HIGH 2, possibly more than one platform change. Add a short "current gap" note pointing to `02` §3.6/§14.

### LOW 2 - `01-software-install-logic.md` still has stale Phase 3 decision language

§12 still reads as "Things to confirm with the user (decision points for Phase 3)." Many are now resolved in v4: generation counters replace actions, dynstate/oper split is selected, SROS-first is selected, `software-install-matrix` is dropped, NETCONF-only Phase 4 is scoped. Add a note that §12 is historical extraction context and point to the resolved sections in `02`.

### LOW 3 - Request history comment in YANG references the wrong design section

The YANG `request` description says "see design doc 02 §3.3" for history retention. v4 retention is §3.4. Tiny, but easy to fix.

## Notes on areas that look good

- Plain `ttt.Transform` is the right substrate for empty observer output. `_TransformTransaction.finalize()` does call `on_conf` for empty output when `running` exists, unlike `_RFSTransaction.finalize()`.
- `last-trigger-result` is a good replacement for silent target-id failures.
- Mandatory `control/confirmation/by-user` is the right YANG choice.
- The file abstraction split is coherent, and the Phase 4 `CheckFiles` deviation is now explicit.
- `DeviceMgr.get_dmc()` exists as v4 says; the only remaining caveat is compiling the actor method call shape when the factory uses it.

## Recommendation

Do one more design revision before Phase 4. The next revision should focus on the platform contract and topology, not step semantics:

1. Replace the "five-line `params.dynstate`" prerequisite with a restore lifecycle that actually reaches the runner after LMDB load.
2. Add a persistence acknowledgement story for Tier A writes, or weaken the Tier A guarantee.
3. Decide whether sw-install attaches at `software-pack` or the whole RFS entry, accounting for `scp-port`.
4. Provide a concrete layer wiring example for global `/software-install` visibility.

Once those are nailed down, I would expect the remaining issues to be mechanical cleanup.
