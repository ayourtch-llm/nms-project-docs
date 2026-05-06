# Round-3 Codex Design Review

Reviewer: Codex  
Date: 2026-05-07  
Scope: fresh read of `00-orientation.md`, `01-software-install-logic.md`, `02-sw-install-design.md` v3, `docs/adr/cli-driver.md`, `stratoweave/sw-install/src/sw_install/yang.act`, then sanity-check against live StratoWeave `ttt.act`, `device.act`, `adapters/adapter.act`, and sorespo's per-device transform pattern.

## Executive summary

v3 is materially better than v2 in the areas the prior reviews pushed on: state is no longer split across memory/dynstate, the lease downgrade is now honest, cancel and enabled semantics are operator-visible, file inspection and transfer are separated, and the CLI design has moved into an ADR instead of overloading the Phase 4 design.

However, I do **not** think this is ready for Phase 4 implementation as written. The main blockers are not step semantics; they are substrate mismatches between the design and current `ttt`/RFS mechanics:

1. `dynstate` is restored into the transform transaction, but the actor created by `act(...)` is not passed that restored dynstate.
2. The "empty downward output, all work in actor" observer shape does not work with `RFSTransform` as-is because `_RFSTransaction.finalize()` suppresses `on_conf` when output is empty.
3. `Layer.declare_subscriptions` reads the **lower** layer, not sibling config in the current host layer, so the proposed global `/software-install` subscription only works if that subtree is deliberately present in the lower layer.
4. The YANG and design disagree about where the model attaches (`/devices/device` vs `/sw-rfs:rfs`), and the current YANG does not include any `sw:transform` / `sw:rfs-transform` hook.

These need a short design correction before coding. The rest of the findings are lower-risk but should be cleaned up now because they affect operability and testability.

## Findings

### HIGH 1 — Dynstate-only ownership is not implementable without an actor restore path

`02-sw-install-design.md` §3 and §8.4 make dynstate the runner's sole source of truth and say the runner is constructed with restored dynstate. Current `ttt.act` does not do that.

Relevant code:
- `_TransformTransaction.restore()` and `_RFSTransaction.restore()` decode `ATTR_DYNSTATE` into `self.dynstate`.
- `TransformActorParams` contains `path`, `update_dynstate`, `update_oper`, `dev`, and `lower`; it does **not** contain `dynstate`.
- `TransformFunction.on_conf(conf, memory)` and `RFSFunction.on_conf(conf, memory)` pass `memory`, not `dynstate`, to the actor callback.

Sorespo's `BBInterfaceTransform` initializes its actor-local `_dynstate` fresh, then calls `update_dynstate()` as telemetry arrives. That pattern is acceptable for a counter-like observer, but sw-install's `DeviceRunner` needs restored request state before it can safely decide whether a request was `processing`, `cancelling`, `paused`, waiting for backoff, etc.

This is a Phase 4 blocker. Pick one:

- Extend `TransformActorParams` / `init_dynstate` / `on_conf` so the actor receives restored dynstate.
- Move the state-machine reconciliation back into `transform_wrapper` and make the actor a side-effect executor, with a clear message protocol carrying decoded dynstate.
- Use `memory` intentionally as the actor restore path, which contradicts v3's core correction and should be avoided unless the platform API is changed.

Without this, restart semantics in §8.4 are aspirational.

### HIGH 2 — Empty observer output conflicts with `RFSTransform` callback delivery

The design repeatedly uses the sorespo RFS pattern as the anchor for Option B and says sw-install produces no downward config:

```acton
return (gdata.Container(), memory)
```

Current `_RFSTransaction.finalize()` only calls `function.on_conf(...)` when `self.output` is non-empty:

```acton
if self.running:
    output = self.output
    if output is not None:
        if not (isinstance(output, gdata.Container) and not output.presence and len(output.children) == 0):
            self.function.on_conf(self.get(), self.memory)
```

So an `RFSTransform` whose `transform_wrapper()` returns an empty container will not notify the actor on config changes. That breaks the proposed "all work happens in `DeviceRunner.on_local_config`" flow.

The design needs to state exactly which transform primitive Phase 4 will use:

- If it is `ttt.Transform`, it can produce empty output and still gets `on_conf`, but it will not receive `params.dev`; the factory must parse the device name from the path and call `dev_registry.get(...)`.
- If it is `ttt.RFSTransform`, either it must produce a harmless non-empty output, or `RFSTransform.finalize()` must be changed for observer transforms, or a new observer-RFS transform primitive is needed.

Do not leave this as "matches sorespo"; sorespo's observed transform still produces device config, so it does not prove this observer-only shape.

### HIGH 3 — Global config subscription path is underspecified and may observe the wrong tree

§7.2 says each per-device transform reads `/software-install/...` via:

```acton
params.lower.declare_subscriptions(...)
```

But `Layer.declare_subscriptions()` samples `get_data()` on the layer object it is called on. `params.lower` is the layer **below** the transform's host layer; it is not a sibling/root view of the current layer. If `/software-install` is only present in the host layer's northbound config, a lower-layer subscription will not see it unless the host layer also passes that subtree downward.

This matters because §2 tells apps to "add the YANG model to their lowest config layer" and wire the per-device transform there. It does not say `/software-install` is transformed or copied into the lower layer. A fresh implementer could follow the doc and get per-device local config callbacks but no global config.

The design needs a concrete topology:

- Which layer owns `/software-install` global config?
- Is that subtree in the same layer as `/sw-rfs:rfs/software-pack`, above it, or below it?
- If it is a sibling in the same layer, what API gives the transform a root/sibling view?
- If it is in the lower layer, what transform passes it there?

If the platform lacks a "subscribe to current layer root" API, this may need either a small platform change or a top-level coordinator/observer after all.

### HIGH 4 — YANG attachment point and transform hook are inconsistent with the design

The design and orientation repeatedly describe `/devices/device/<name>/software-pack` and direct `/devices/device/scp-port`. The actual v3 YANG augments:

```yang
augment "/sw-rfs:rfs" {
  leaf scp-port { ... }
  container software-pack { ... }
}
```

Current StratoWeave has both `/sw-rfs:device` and `/sw-rfs:rfs` lists. Sorespo's service transforms attach under `/sw-rfs:rfs`, while device meta config lives under `/sw-rfs:device`. The current YANG places `scp-port` under `/rfs`, not directly under the device meta list, despite §4.8 and §9.6 saying it is a direct device augmentation.

Also, `yang.act` imports `stratoweave-rfs` but not `stratoweave`, and it does not mark `software-pack` with `sw:transform` or `sw:rfs-transform`. If wiring is manual, say that explicitly and show the manual `ttt.Container(...)` shape. If wiring is generated, the extension is missing.

This needs cleanup before implementation:

- Decide whether sw-install lives under `/sw-rfs:rfs[name]/software-pack` or `/sw-rfs:device[name]/software-pack`.
- Put `scp-port` where the design says it belongs, or revise the design to say it is an RFS-entry setting.
- Add the transform extension if generated wiring is expected.
- Update all docs away from the ambiguous `/devices/device` shorthand, or define that shorthand precisely.

### HIGH 5 — Cancel callback guard appears to prevent the `cancelling -> cancelled` transition

§8.5 says cancellation bumps `generation_token` so in-flight callbacks no-op. It also says "when it returns from the device, `_step_completion` sees status==cancelling and transitions to cancelled."

Those two statements conflict if `_step_callback_guard(token, then)` wraps the completion callback and drops stale tokens before invoking `then()`. After cancellation, the in-flight callback necessarily has the old token, so it is stale and never reaches `_step_completion`.

The design needs a two-lane callback rule:

- stale normal completions must not mutate plan/state;
- stale cancelled completions must still notify the runner that the in-flight operation drained, so status can become `cancelled`.

This is also where timeout handling belongs. If a device RPC never calls back, `cancelling` can become permanent unless there is a watchdog or adapter-level timeout contract.

### MEDIUM 1 — Phase 4 pre-staged-image story conflicts with `CheckFiles`

§9.4 says Phase 4 has `NoopFileTransfer`, but SROS can still verify pre-staged images via `RemoteFileInspector` and skip `CopyImage`. The YANG `filename` description also says NETCONF-only Phase 4 expects files already present on the device at the given path.

The logic spec and design step list still say every OS starts with `CheckFiles`, and `CheckFiles` uses `LocalFileInspector` to verify controller-side files. If the operator only pre-stages the image on the device, `CheckFiles` fails before `RemoteFileInspector` can prove the copy is unnecessary.

Resolve this explicitly. Options:

- Phase 4 still requires local files even when device files are pre-staged; then update §9.4 and the YANG description.
- Phase 4 changes `CheckFiles` semantics when `FileTransfer.caps().put == false`; then document this as a conscious deviation from Python.
- Add separate `local-filename` and `remote-path`/URL modeling instead of overloading `filename`.

Right now a fresh reader cannot tell how to run an SROS Phase 4 install without a transfer implementation.

### MEDIUM 2 — Coalesced dynstate writes endanger critical restart data

§3 says dynstate is the source of truth, but §8.3 says run-log entries, install op-id polling progress, and similar tick-level updates mutate in-memory dynstate and are only periodically flushed.

That is reasonable for lossy telemetry, but not for state that makes steps idempotent after a crash. Examples:

- IOS-XR `op_id_add` / `op_id_prepare` / `op_id_activate` / `op_id_commit`
- SROS/Junos `rebooted` and `boot_time`
- `destination_paths` after a partial copy
- generation high-water marks before side effects begin

The design should classify dynstate fields into "must persist before side effect", "persist at step boundary", and "best-effort telemetry". Otherwise "dynstate-only" sounds durable while the implementation may still lose the exact data needed to resume safely.

### MEDIUM 3 — `waiting-for-device` needs a concrete wakeup mechanism

§8.7 says the runner pauses until DMC + adapter are available, and "`on_local_config` re-checks readiness on each call; a one-shot subscription on the device's status (or polling) drives re-evaluation."

That is not yet a design. Current `DeviceMgr.set_dmc()` can switch from `NoAdapter` to a real adapter and `on_connect()` can update modset, but a transform actor under `/rfs/.../software-pack` will not necessarily get a local config callback just because DMC changed. If this uses polling, specify the interval and persistence behavior. If this uses a subscription, specify the exact tree/provider and whether `NoAdapter.declare_subscriptions()` immediately returning `NotConnectedError` is enough to reschedule.

Also note that `DeviceMgr.get_dmc()` and `DeviceMgr.get_adapter()` already exist in the current code, so the design should stop listing `get_dmc()` as an open platform ask.

### MEDIUM 4 — `DeviceMgr.get_dmc()` platform ask is stale

`02-sw-install-design.md` §9.5 and §14 present `DeviceMgr.get_dmc()` as a requested platform addition. In the live code, `device.act` already has:

```acton
def get_dmc():
    return dmc
```

This is documentation drift, but it matters because §12 still asks reviewers whether the platform team will accept it. Replace Q2 with the real question: is returning full `DeviceMetaConfig` from `DeviceMgr` acceptable as an API contract for sw-install/file transfer, and are there redaction/logging constraints for credentials?

### MEDIUM 5 — Backoff type/rounding is not specified

The design says `backoff = (error_count.backoff or 10) * factor`, where `factor` is decimal. The YANG request oper leaf `error-count/backoff` is `uint32`.

If factor mode uses `1.2`, the sequence is fractional after the second retry (`14.4`, `17.28`, ...). The Python scheduler may tolerate floats internally, but the YANG surface cannot. Specify rounding/truncation/ceiling and use the same rule for `next_wake_at`. If matching Python exactly matters, verify what Python writes to the original uint32 leaf and document that.

### MEDIUM 6 — Trigger target errors should not be silent

§4.6 says target-id scoping "errors silently if no such id exists in current+history." Silent failure is poor for automation, especially because generation counters are edge triggers and the operator may think a cancel/start/clear happened.

Prefer an oper-visible `last-control-result` or per-trigger diagnostic with `{generation, target-id, status, error, at}`. Even if actions are gone, automation needs feedback for invalid target ids and stale target ids pruned from history.

### MEDIUM 7 — Request history retention and idempotency baseline are overcomplicated and slightly unclear

§3.3 retains latest non-terminal, latest of each terminal status, plus older entries up to 50, and says never drop the entry holding the most recent `last_pack_snapshot`. But `SwInstallDynstate` already has `last_pack_snapshot` as a top-level field, so idempotency does not need to retain a request entry for pack comparison.

If the top-level snapshot is authoritative, simplify the retention rule. If the retained request is authoritative, remove the redundant top-level snapshot or define how they are kept consistent. Two baselines invite drift in restart/pruning code.

### MEDIUM 8 — Confirmations as config need pruning semantics

`control/confirmation[]` is config. The runner observes it and stamps oper `confirmed`. The design does not say when stale confirmations are removed or ignored.

Questions to settle:

- If request 12 is pruned from history, are confirmations for request 12 still accepted in config forever?
- If an operator pre-creates confirmations for a future request id, are they ignored until that id exists, rejected, or consumed later?
- Does `confirm-all-generation` create persistent entries under `control/confirmation[]`, or only internal confirmed state in dynstate?

Leaving stale config behind is manageable, but the runner must have deterministic matching and cleanup behavior.

### LOW 1 — `control/confirmation/by-user` should probably be mandatory

The YANG says the runner stamps `confirmed.{by-user, when}` from matching confirmation entries, but config `control/confirmation/by-user` is optional. If absent, the oper projection either has empty attribution or invents one. Make it mandatory, or specify the fallback.

### LOW 2 — Diagnostic projections are good, but Junos projection is not represented in YANG

§3.4 lists per-Juniper RE diagnostics: `version`, `boot-time`, `rebooted`. The current YANG has component-level `boot-time` and `rebooted`, but no per-RE list under component. That loses the structure needed for dual-RE Junos.

If Junos is Phase 6, it is acceptable to defer, but then §3.4 should say the current v3 YANG only models common/SROS/IOS-XR diagnostics and Junos per-RE diagnostics will extend it in Phase 6.

### LOW 3 — `request-generation` target id is confusing

The YANG includes `request-target-id` but its description says it is generally absent because request-generation creates a new request. That companion leaf does not seem useful and can mislead users into thinking "create a new request based on request id X" is supported.

Consider dropping `request-target-id`, or define an actual behavior. Target scoping is valuable for start/cancel/confirm/clear; for create it appears to be conceptual noise.

### LOW 4 — ADR boundary is mostly right, but Phase 4 should not commit to CLI stubs per method unless needed

The CLI ADR is appropriately separated and does not overcommit Phase 4 to TextFSMPlus implementation details. The one thing I would soften is "Ship per-OS `Ops` modules with NETCONF strategy real and CLI strategy stubs raising `NotImplementedError`."

For Phase 4, it is enough that `DeviceOps` signatures and factories can accept a future `CliSession`. Adding dead per-method CLI branches now increases test surface without delivering behavior. A single `cli_session` field plus no CLI selection until Phase 5 is a cleaner boundary.

## YANG-specific review

The v3 YANG is readable and the control subtree is understandable, but I would not treat it as settled until the attachment decision is resolved.

Specific notes:

- The model says `scp-port` is direct device-level, but it augments `/sw-rfs:rfs`, not `/sw-rfs:device`.
- No transform extension appears in the model. If generated TTT wiring is expected, add `import stratoweave { prefix sw; }` and the appropriate `sw:transform` / `sw:rfs-transform`.
- `software-file/filename` description currently changes semantics by phase and transport. That is risky because existing spec says it is a controller-local path. Better model local path vs remote/device URL explicitly if the phase behavior differs.
- `run-log` key `(when, seq)` is a good improvement.
- `paused`, `cancelling`, and `waiting-for-device` are useful request states.
- Dropping `internal-state` is the right direction, but the current diagnostic leaves are incomplete for Junos dual-RE.
- `error-count/backoff` should not be `uint32` unless rounding is specified.

## Onboarding-doc review

`00-orientation.md` mostly stands on its own and does a good job getting a fresh reader into StratoWeave vocabulary. The biggest issue is that it overstates the observer-transform pattern:

- It says `update_dynstate` "persists actor-private state via lmdb, restored on platform startup. The runner's source of truth." That is true for storage, but not enough for actor restore. A fresh reader will assume the actor receives restored dynstate because the doc says "actor-private." Current `ttt` does not pass it to `act`.
- It uses `/devices/device/...` language throughout, while the current platform YANG has `/sw-rfs:device` and `/sw-rfs:rfs`. For this project the distinction matters.
- It presents `DeviceMgr.get_dmc()` as not available later in the design, but the live code now has it. The onboarding could mention that modules should generally use `DeviceMgr`'s public surface, including `get_dmc()` when credential-bearing helpers need it.

`01-software-install-logic.md` is strong. It captures the Python behavior with enough detail to implement from. The few places that deserve extra emphasis for the Acton port are:

- `CheckFiles` is local-controller filesystem validation in the Python package.
- Python cancellation kills the worker process; Acton cancellation is cooperative and must have an explicit drain/timeout contract.
- `internal-state` is not just diagnostic; it is the restart/idempotency record.

## Recommendation before Phase 4

Do one short v3.1 pass before coding. It should answer only the substrate questions:

1. Exact mount point: `/sw-rfs:rfs` vs `/sw-rfs:device`.
2. Exact transform primitive: `Transform`, `RFSTransform`, or a new observer-RFS variant.
3. Exact restored-dynstate path into `DeviceRunner`.
4. Exact global-config observation path.
5. Exact cancellation drain callback rule.

Once those are corrected, the step-level architecture and Phase 4 SROS scope look implementable.
