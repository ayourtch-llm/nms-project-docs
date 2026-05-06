# 03 — Codex Review of `sw-install` Phase 4 Design

Review date: 2026-05-06

Scope read:

- `docs/00-orientation.md`
- `docs/01-software-install-logic.md`
- `docs/02-sw-install-design.md`
- `stratoweave/stratoweave/src/stratoweave/ttt.act`
- `stratoweave/stratoweave/src/stratoweave/device.act`
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act`
- current scaffold under `stratoweave/sw-install/`

Overall: the proposed Acton/Stratoweave direction is sound. The design should stay reactive and actor-driven rather than preserving NSO's action callbacks and multi-process worker machinery. The main changes I recommend before implementation are:

- keep a small writeable control surface so dropping actions does not also drop important operator semantics;
- add a per-device install execution guard, because `DeviceMgr` serializes actor work but does not by itself provide a semantic install lease;
- make `FileTransfer` credential-aware without reaching through concrete adapters;
- tighten persistence/dynstate and YANG config/oper boundaries before coding the runner.

## 1. Dropping YANG actions

Question: `02-sw-install-design.md` §4 proposes dropping `create-request`, `execute-request`, `confirm-step`, `cancel-request`, and `clear-run-log`, replacing them with config-driven reactive triggers and writeable leaves.

Verdict: mostly yes, this is the right Stratoweave call.

Stratoweave does not appear to have a native action-as-RPC path comparable to NSO actions. The platform pattern is that transforms receive config, maintain memory/dynstate, and project operational data via `update_oper`. That is explicit in `ttt.TransformActorParams`, which carries:

- `path`
- `update_dynstate`
- `update_oper`
- optional `dev`
- optional `lower`

See `stratoweave/stratoweave/src/stratoweave/ttt.act` around `TransformActorParams` and `TransformFunction.init_dynstate`.

Operational data projection is also already supported in `get_data`: transform data is merged with `self.oper` before filtering. That means a runner-owned request/plan/run-log view exposed through `update_oper` is not an odd side channel; it is the existing mechanism.

So I agree with dropping the action nodes as the primary mechanism.

The pushback: do not accidentally drop the action semantics.

The Python spec has two semantics that a pure "pack data changed" trigger does not preserve:

1. A cancelled last request forces a new request even if the pack is unchanged.
2. `execute-request` gives an operator an explicit start/enqueue operation distinct from creating the request.

The design currently says a user-visible "force a new request" knob is not strictly needed and that changing the pack does it. I disagree. The cancelled-request reactivation rule is load-bearing in `01-software-install-logic.md` §3.1:

```text
A cancelled last request always forces a new request, even if the pack is unchanged.
```

If you remove the explicit action and do not replace it with a control leaf, the operator has no clean way to say "retry this same pack after cancellation" without making a fake pack change.

Recommended shape:

- Drop NSO-style action nodes.
- Add a small config control subtree, separate from oper request state.
- Use generation/counter leaves rather than boolean "edge" leaves where possible.

For example:

```yang
container software-pack {
  presence "Software-pack present";

  leaf name { ... }

  container control {
    leaf request-generation {
      type uint64;
      default 0;
      description
        "Increment to force a new request even if the selected pack is unchanged.";
    }

    leaf start-generation {
      type uint64;
      default 0;
      description
        "Increment to explicitly start or resume the newest request.";
    }

    leaf cancel-generation {
      type uint64;
      default 0;
      description
        "Increment to cancel the active request.";
    }

    leaf clear-run-log-generation {
      type uint64;
      default 0;
      description
        "Increment to clear the current request run log.";
    }
  }
}
```

This is more robust than write-once booleans because it survives replay and avoids the runner needing to clear config leaves behind the user's back.

For confirmation, I would also avoid making `request/component/step/confirmed` directly writeable inside an otherwise operational request subtree. In the original model, `request` is `config false`; the current design wants to repurpose one child under it as config. That tends to be awkward for generated config/oper schemas.

Better:

- keep request/component/step/status/etc. as operational projection;
- add a config sibling for confirmations keyed by request/component/step;
- project confirmed status back into oper for readability.

Example:

```yang
container control {
  list confirmation {
    key "request-id component step";
    leaf request-id { type uint32; }
    leaf component { type string; }
    leaf step { type string; }
    leaf by-user { type string; }
  }
}
```

The runner can stamp `when` in oper when it observes the config. If you need "confirm all", either support a control generation leaf or let the client create entries for all known steps.

For `enabled`, define exact semantics now. The scaffold YANG says:

```text
When false, requests are not advanced even if otherwise ready.
```

That is a reasonable interpretation. I would make it mean:

- pack assignment can still materialize/obsolete requests;
- no step execution starts while disabled;
- in-flight runner reaches a cooperative stop point and pauses;
- re-enable resumes according to `start-generation` / auto policy.

That is cleaner than making disabled suppress request creation, because operators can still see desired-vs-pending state.

## 2. Simplifying the scheduler

Question: `02-sw-install-design.md` §8 simplifies the Python multi-process scheduler to one `RequestRunner` actor per active request, relying on `DeviceMgr` being one actor per device for serialization.

Verdict: yes to removing the Python scheduler; add a per-device install guard.

The Python scheduler exists largely because NSO executes work in external processes and needs queues, worker PID tracking, cancellation via SIGINT, and per-device coordination. Stratoweave already gives you actors and async callbacks, so carrying over that machinery would be unnecessary.

`DeviceMgr` is indeed per-device and serializes its configuration queue. Its docstring says device configuration interaction is serial: either idling or with configuration in transit, with new target configs queued. The implementation backs that with `current_tids`, `confq`, `_send_config`, and callbacks.

That supports the design's claim that you do not need a global multi-process worker pool.

The caveat: `DeviceMgr` serializes actor message handling and config reconciliation; it does not by itself provide a semantic "software install owns the device" lease.

Important details:

- `DeviceMgr.configure(...)` queues intended config and sends it through adapter `configure`.
- `DeviceMgr.rpc_xml(...)` simply forwards to `adapter.rpc_xml(...)`.
- `DeviceMgr.fetch_config(...)` forwards to `adapter.fetch_config(...)`.
- Resync and normal reconciliation can still happen while a software install step is waiting on a callback.

So if the install runner performs install RPCs, reboots, BOF edits, or long polling through `rpc_xml`, you still need to prevent two software-install flows, or a stale cancelled flow, from publishing or acting concurrently for the same device.

Recommended shape:

- Have exactly one device-scoped software install owner at a time.
- Prefer one `DeviceRunner` actor per device, with request generations inside it, rather than independent request actors racing on the same device.
- If you keep `RequestRunner` per active request, give each one a `(device, request_id, generation)` token and make all callbacks check it before mutating state or publishing oper.
- When a new request supersedes an old one, mark the old generation obsolete and make pending callbacks no-op.
- Treat cancellation as cooperative where possible; Acton callbacks cannot be killed like Python worker processes. Cancellation should set state immediately and cause future callbacks to check and stop.

Backoff can remain internal to the device/request runner using `after delay`. That is a good simplification.

One bug to avoid in the backoff status mapping: the Python spec distinguishes consecutive transient and other errors. `_write_request_status` maps WAIT to `failed-transient`, increments transient, and resets other; FAILURE maps to `failed-other`, increments other, and resets transient. Do not use `transient + other > max_retries` if the intended semantics are consecutive retries of the current class. The spec calls these consecutive counters.

Also be careful with final "give up" status:

- WAIT retries exhausted should end as `failed-transient`.
- FAILURE retries exhausted should end as `failed-other`.

Do not always publish `FAILED_OTHER` on max retries.

## 3. FileTransfer and credential reuse

Question: `02-sw-install-design.md` §9 introduces a `FileTransfer` interface so SCP/SFTP can be deferred to Phase 5 without locking the design.

Verdict: good boundary; make the factory richer and avoid concrete adapter coupling.

The proposed `FileTransfer` methods are a reasonable first cut:

- `put(local_path, remote_path)`
- `list_files(remote_dir)`
- `delete(remote_path)`
- `checksum(remote_path)`

This is enough to keep `CopyImage` and `Cleanup` from hard-coding SCP/SFTP. It also allows Phase 4 to fail clearly when a file is missing and no transfer implementation is configured.

I would change three things before coding it.

First, make metadata typed rather than `dict[str, int]`.

The steps need more than file sizes over time:

- remote path
- size
- checksum value
- checksum algorithm
- maybe modified time
- maybe existence without listing the whole directory

Suggested value:

```acton
class RemoteFileInfo(value):
    path: str
    size: ?u64
    checksum: ?str
    checksum_algorithm: ?str
    mtime: ?str
```

Then provide:

```acton
proc def stat(remote_path: str, cb: action(?RemoteFileInfo, ?Exception) -> None) -> None
proc def list(remote_dir: str, cb: action(list[RemoteFileInfo], ?Exception) -> None) -> None
```

Second, add explicit capabilities.

Not every backend can do every operation. Device-pull may support copy but not delete. Some platforms may provide checksum, others may not. The runner should know whether missing checksum means "cannot verify" or "verification failed".

Possible shape:

```acton
class FileTransferCaps(value):
    put: bool
    delete: bool
    checksum: bool
    device_pull: bool
```

Third, credential reuse should flow from device meta config, not from `NetconfAdapter`.

`DeviceMgr` owns device meta config (`DeviceMetaConfig`) and passes it into adapters. `NetconfAdapter` uses `new_dmc.credentials.username` and `new_dmc.credentials.password` when setting up NETCONF. A future SFTP/SCP implementation should reuse the same source data.

Do not reach through `DeviceMgr.get_adapter()` and special-case `NetconfAdapter`; that couples file transfer to one adapter implementation and makes mock/testing awkward.

Better options:

- expose a narrow `DeviceMgr.get_dmc()` or credential accessor if acceptable;
- pass `DeviceMetaConfig` into `FileTransferFactory`;
- let the app provide `file_transfer_factory(device_name, dmc, pcap)`.

The original YANG also has `scp-port` on each device. The logic spec calls it out as part of the per-device augmentation. Preserve it somewhere. It can live in sw-install device config or in Stratoweave device metadata, but do not drop it if SCP remains a future fallback.

Device-pull should remain the preferred future path where supported. It avoids running controller-side SCP/SFTP and reduces dependence on shell tools. But SCP/SFTP fallback is practical given the existing ecosystem already uses process capabilities and command-line SSH/sshpass for NETCONF transport.

## 4. Load-bearing logic and design gaps

Question: Anything in the Python spec §1-§11 missed or wrong?

Main misses and corrections:

### 4.1 Current scaffold YANG is missing the per-device surface

The current `stratoweave/sw-install/src/sw_install/yang.act` defines global `/software-install` and the pack library, but not the per-device association/request/control surface.

The original model has:

- `/devices/device/scp-port`
- `/devices/device/software-pack` presence container
- `/devices/device/software-pack/name`
- operational `request[]`
- request actions

The port does not need NSO's `tailf-ncs` augment, but it does need an equivalent Stratoweave device association. This is the integration point for the runner, so it is not just a later polish item.

### 4.2 Config and oper need a clean split

The design says actor state projects to oper, which is good. But it also proposes writeable leaves under request/step, where request was operational in the Python model.

Stratoweave generation already has distinct config-only and oper views. Mixing user control leaves into an otherwise operational subtree will make typing and RESTCONF behavior harder.

Recommended split:

- config:
  - pack library
  - device pack assignment
  - control generations
  - confirmation inputs
  - request-level overrides, if they are user inputs
- oper:
  - materialized requests
  - immutable pack snapshots
  - status
  - plan
  - current confirmed projection
  - run-log
  - error counters
  - run-id-count

### 4.3 Transform memory/dynstate is underspecified

The design's transform sample returns `(gdata.Container(), memory)`.

That does not update memory. If request materialization depends on comparing pack data against memory, the runner must cause the new snapshot to be stored somewhere persistent.

Options:

- `transform_wrapper` computes the new memory directly from config and returns it.
- Runner owns request state and calls `update_dynstate` whenever request metadata changes.
- Use both: transform memory for last input snapshot, dynstate for runner state.

Do not leave this implicit. Restarts will otherwise duplicate requests, lose request IDs, or fail to honor "existing request" semantics.

Also note: `update_dynstate` triggers a recompute in `ttt.act`. Make sure runner calls are idempotent and do not create an update loop.

### 4.4 `internal-state` is more than debug in the Python package

The design says Acton does not need `internal-state` as a source of truth because state is typed and can live in dynstate. That is acceptable, but only if dynstate really persists all fields needed to resume:

- destination paths
- destination volume
- boot time
- rebooted flags
- IOS-XR op IDs
- Junos RE state
- failover config snapshot
- current plan statuses
- run counters and error counters

If any of those are only in memory, reboot/restart behavior will diverge from Python.

It is fine to keep `internal-state` as a human-readable oper summary, but make dynstate the explicit source of truth and test restore.

### 4.5 Plan refresh monotonicity is load-bearing

The Python plan refresh may add steps but must not remove prior components/steps. It raises if old plan entries disappear.

That is important for IOS-XR and Junos dynamic cases, and also for preserving operator confirmations/status. The Acton `ComponentPlan.refresh_steps` should enforce this exactly, not just recompute a fresh plan.

### 4.6 Step cursor semantics need exact preservation

The spec's `next_step` behavior is important:

- `next_step` can jump forward.
- skipped intermediate steps are marked `skipped`.
- failures reset subsequent steps.
- WAIT ends the run as `failed-transient`; it is not an in-step retry.
- `pre_check` can skip execute but `next_step` still runs after step handling.

These details affect idempotency and resume behavior. They should be covered in pure `test_plan.act` before device step implementation.

### 4.7 Request status and retry counters

Make status mapping exact:

- obsolete -> `obsolete`
- all components completed -> `done`, clear errors
- needs confirmation -> `waiting-confirmation`
- last result WAIT -> `failed-transient`, transient++, other=0
- last result FAILURE -> `failed-other`, other++, transient=0

The design's backoff pseudocode using `transient + other` risks drifting from "consecutive" semantics.

### 4.8 Capability detection versus pack OS

The logic spec distinguishes:

- pack `os`: selects step module;
- device capability-derived `DeviceOs`: validates the actual platform.

The port should preserve that separation. Do not infer the step module solely from `DeviceMgr.get_modules()`, and do not skip `ValidatePlatform`.

Also note that the scaffold keeps `vrp` in the enum. The logic spec says VRP is recognized in the YANG enum but has no implementation. Decide whether:

- keep `vrp` but fail validation with a clear unsupported-platform error; or
- remove it from the ported YANG and document the compatibility break.

Keeping it but failing clearly is probably less surprising for a port.

### 4.9 Run-log retention

Bounded run-log is a good improvement over the Python package's unbounded log. Still, explicit clear is cheap and operationally useful.

I recommend:

- bounded retention, default e.g. 1000 entries per request;
- `clear-run-log-generation` control leaf;
- preserve the run-log shape: `{when, run-id, component, step, message}`.

### 4.10 Phase 4 NETCONF-only scope

NETCONF-only Phase 4 is reasonable, but be explicit about what SROS can and cannot complete without file transfer.

`CopyImage.pre_check` can pass only when files are already present and verifiable. If files are missing and no `FileTransfer` is configured, `CopyImage.execute` should fail with a clear permanent error, not spin as transient.

This lets the runner, plan, logging, confirmation, and NETCONF RPC plumbing be tested before SCP/SFTP exists.

## Recommended implementation adjustments

Before going deep into Phase 4, I would revise the design/scaffold as follows:

1. Add an explicit config control subtree for request generation, start/resume, cancel, clear log, and confirmations.
2. Keep request/plan/run-log as operational projection only.
3. Define the exact dynstate schema/source of truth for request IDs, request snapshots, plan statuses, per-component state, counters, and active generation tokens.
4. Use one device-scoped runner/lease, even if it owns per-request child actors internally.
5. Adjust retry exhaustion to preserve `failed-transient` vs `failed-other`.
6. Make `FileTransferFactory` take device metadata or a credential accessor, not a concrete adapter.
7. Preserve `scp-port` or an equivalent device file-transfer override.
8. Add pure tests for create-request idempotency, cancelled reactivation, plan refresh monotonicity, next-step jumps, WAIT/FAILURE status mapping, and stale callback no-ops.

Net: proceed with the reactive design. The Stratoweave primitives support it. The main thing is to replace the NSO actions with a small declarative control model, not with implicit pack-change-only behavior.
