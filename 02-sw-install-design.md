# 02 — Stratoweave `software-install` Module Design

> **Status: v2 — incorporates round-1 review feedback from `docs/reviews/01-codex-design.md` and `docs/reviews/02-claude-design.md`, with decisions consolidated in `docs/reviews/03-integration.md`. Pending round-2 review before Phase 4 implementation continues.**

Read `00-orientation.md` (project context + stratoweave concepts) and `01-software-install-logic.md` (language-agnostic spec extracted from the Python source) first. Decisions made in `03-integration.md` are referenced inline as **D1**, **D2**, etc.

Markers used in this doc:
- **❓DECISION** — explicit user-facing question still open
- **⚠️ASSUMPTION** — choice I'm making absent input that should be sanity-checked
- **🆕** — added in v2 from review feedback

---

## 1. High-level shape

A **new sibling repo** under `stratoweave/`, mirroring the layout of `sorespo`, `actxcrypt`, etc.:

```
stratoweave/
└── sw-install/                           # ← new repo (v2 confirms sibling-repo)
    ├── Build.act                         # name=sw_install; deps: stratoweave, netconf, yang
    ├── Makefile, README.md, REUSE.toml, LICENSES/, .gitignore
    ├── gen_adata/
    │   ├── Build.act
    │   └── src/gen_adata.act             # YANG → typed Acton classes
    └── src/
        └── sw_install/
            ├── sw_install.act            # public API: make_sw_install_transform(...)
            ├── yang.act                  # raw YANG strings (input to gen_adata)
            ├── model.act                 # GENERATED — typed accessors
            ├── pack.act                  # SoftwarePack / SoftwarePackComponent value types
            ├── plan.act                  # ComponentPlan + StepStatus state machine
            ├── state.act                 # State / StateSros / StateIosXr / StateJunos
            ├── step.act                  # Step protocol + StepResult enum
            ├── transform.act             # the stratoweave Transform integrating with a layer
            ├── device_runner.act         # 🆕 D1: actor DeviceRunner (one per device)
            ├── runlog.act                # bounded run-log helpers + filter
            ├── file_transfer.act         # FileTransfer interface + NoopFileTransfer
            ├── platform_sros.act         # Phase 4 — SROS step impls
            ├── platform_iosxr.act        # Phase 5
            ├── platform_junos.act        # Phase 5
            ├── ops_sros.act              # SROS DeviceOps (netconf + cli strategy)
            ├── ops_iosxr.act             # Phase 5
            ├── ops_junos.act             # Phase 5
            ├── cli_session.act           # 🆕 CLI session driver via TextFSMPlus
            └── test_*.act                # tests
```

The module is a **library**: apps opt in by adding the YANG to their layer stack and wiring the transform.

**Public API** (`sw_install.act`):

```acton
def make_sw_install_transform(
    dev_registry: swdev.DeviceRegistry,
    file_transfer_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer = None,
    cli_session_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> CliSession = None,    # 🆕 CLI
    log_handler: ?logging.Handler = None,
) -> proc(ttt.Path, ?ttt.Layer) -> ttt.Node

# YANG model exports — the per-app gen_adata pulls these in
SOFTWARE_INSTALL_YANG: yang.Module
```

---

## 2. Module boundary — what apps integrate against

Apps:

1. **Add the YANG model** to their lowest config layer (the same layer where `/devices/device` lives).
2. **Wire a `Transform`** at that layer attached at the per-device path.
3. **Optionally provide a `FileTransfer` factory** for image upload (Phase 5+; Phase 4 ships only `NoopFileTransfer`).

The integration looks like this on the app side:

```acton
import sw_install

def get_layers(dev_registry, log_handler, db):
    swi_transform = sw_install.make_sw_install_transform(
        dev_registry,
        file_transfer_factory=None,   # Phase 4: no upload
        log_handler=log_handler,
    )
    layer0 = ttt.Layer('0', my_t_0_with_swi(swi_transform), layer1, db)
    # ...
```

---

## 3. Where state lives — the central design decision (revised)

Stratoweave gives us three places to put data: gdata config (writeable by users), gdata oper data (readable, produced by transforms via `update_oper`), and actor-private state (persisted via `update_dynstate`). The clean split — corrected after round-1 feedback (**A5**) — is:

| Data | Home | Visibility |
|------|------|------------|
| `/software-install/software-pack[]` (pack library) | gdata config | read/write |
| `/software-install/{enabled, confirm-steps, auto-execute-after-confirm, error-handling}` (global policy) | gdata config | read/write |
| `/devices/device/software-pack/name` (pack assignment) | gdata config | read/write |
| 🆕 **`/devices/device/software-pack/control/{request,start,cancel,confirm-all,clear-run-log}-generation`** (operator-driven triggers) | gdata config | read/write |
| 🆕 **`/devices/device/software-pack/control/confirmation[]`** (per-step confirmation inputs) | gdata config | read/write |
| `/devices/device/software-pack/request[]` (materialized requests, immutable snapshots) | gdata oper, via `update_oper` | read-only |
| `request/component[]/step[]` (plan + step status) | gdata oper, via `update_oper` | read-only |
| `request/run-log[]` (bounded log, with `swi_component` filter — A10) | gdata oper, via `update_oper` | read-only |
| 🆕 **`/devices/device/software-pack/last-create-result`** (action-return-values regression fix — CL1) | gdata oper, via `update_oper` | read-only |
| `request_id` counters, plan, per-component State, error counters, current generation tokens, **next_wake_at**, **error_count.backoff** | actor-private, persisted via `update_dynstate` (CL5, CL15) | internal |

**Crucial v2 change**: user inputs no longer live under `request[]`. The `request[]` list and everything below it is operational projection; user inputs go under a `control/` config sibling. This avoids mixing config and oper axes within the same subtree (codex Q1 / claude item 5 — A5).

### 3.1 Generation-counter pattern (🆕 A2)

For all operator-driven triggers we use **`uint64` increment-to-trigger** leaves under `control/`:

- `request-generation` — increment to force a fresh request even if pack-data is unchanged (covers cancelled-request reactivation — A1)
- `start-generation` — increment to start/resume the latest request (preserves stage-and-review — CL2)
- `cancel-generation` — increment to cancel the active request
- `confirm-all-generation` — increment to confirm all pending steps (covers `case all` semantics — CL4)
- `clear-run-log-generation` — increment to clear the run-log for forensic purposes (CL4)

**Why generation counters and not edge-trigger booleans:** they are idempotent under config replay (rebooting the controller and re-applying config doesn't re-fire); they don't require the runner to clear config behind the user's back; they survive backup/restore; they give the runner an unambiguous "did this trigger fire since the last time I observed?" question to answer.

The runner persists each "last-observed generation" in dynstate. A trigger fires when the config value exceeds the persisted value.

### 3.2 Transform memory and create-request idempotency (🆕 A6)

The transform's `memory` holds a snapshot per device:

```
memory[device_name] = {
    last_pack_data: ?SoftwarePack,           # what was last materialized
    last_request_generation: u64,
    last_start_generation: u64,
    last_cancel_generation: u64,
    last_confirm_all_generation: u64,
    last_clear_run_log_generation: u64,
}
```

The runner explicitly updates memory whenever it materializes a new request (or transitions one), via the transform's `update_dynstate` path. On restart, memory restores from lmdb, and the runner can correctly answer "is this pack-data the same as the last materialized request?" — preserving the spec §3.1 idempotency rule across reboots.

### 3.3 What about `internal-state`? (🆕 CL14)

The Python package serializes the per-component `State` object as `jsonpickle` JSON in `internal-state`. The Acton port does **not** model this in YANG. Per-OS State subclasses are statically typed; their persistence is the responsibility of the runner via `update_dynstate`. The original `internal-state` leaf is **dropped from the v2 YANG**; debugging surfaces (if needed later) can be added explicitly without touching the spec mapping.

---

## 4. The control surface — surfacing actions reactively

The Python package exposes five NSO actions: `create-request`, `execute-request`, `cancel-request`, `confirm-step`, `clear-run-log`. Stratoweave has no first-class action node. We replace each with a **config-driven trigger** plus, where needed, an **oper feedback channel**.

### 4.1 `create-request` ↔ pack-data change OR `request-generation` increment

**Trigger:** the runner observes `(pack-data ≠ memory.last_pack_data) OR (request-generation > memory.last_request_generation)`.

**Behavior:** materialize a new request (next sequential id, starting at 1 — CL13). Mark prior requests `obsolete=true`. Snapshot pack-data into the new request. Update memory.

**🆕 Feedback (CL1):** the runner stamps `last-create-result = {request-id, status: "new-request"|"existing-request", at: <timestamp>}` in oper data. Callers writing pack assignment can read this to learn the assigned id without scanning the request list.

**Cancelled-reactivation case (A1):** if the last request had status `cancelled`, the next config write of an unchanged pack-data does not, by itself, trigger a new request (memory comparison says "same"). The user increments `request-generation` to force one. This preserves the Python spec's "cancelled forces new request" rule explicitly.

### 4.2 `execute-request` ↔ `start-generation` increment (preserves stage-and-review)

**🆕 Decision (CL2):** transition `unprocessed → processing` requires either:

- `start-generation` to advance past the last-observed value, **OR**
- `auto-execute-after-confirm = true` AND all required confirmations are in place.

This preserves the operator workflow: pack assignment changes → new request materialised in `unprocessed` → operator inspects plan and per-step confirmation gates → operator increments `start-generation` to kick off. **Default value of `auto-execute-after-confirm` remains `false`** to match the Python spec's explicit-start posture.

### 4.3 `confirm-step` ↔ writeable confirmations under `control/`

Per step confirmations live in `control/confirmation[]` (config), keyed by `(request-id, component, step)`:

```yang
list confirmation {
    key "request-id component step";
    leaf request-id { type uint32; }
    leaf component { type string; }
    leaf step { type string; }
    leaf by-user { type string; }
}
```

When the runner's plan loop reaches a step in `waiting-confirmation` status and a matching `confirmation` exists in config, the step proceeds. The runner stamps `confirmed.{by-user, when}` in the oper projection of that step.

**🆕 `confirm-all-generation` (CL4):** incrementing this leaf is equivalent to "create confirmations for every step of every component in the latest request". Implementation-wise the runner expands it internally — the user does not write the per-step entries. This restores the Python `case all { leaf all empty; }` ergonomics.

### 4.4 `cancel-request` ↔ `cancel-generation` increment

**Trigger:** `cancel-generation > memory.last_cancel_generation`.

**🆕 Cancel semantics (CL6):** cancellation in Acton is cooperative. It takes effect at the **next step boundary or completion of the current outstanding RPC, whichever is sooner**. For long-polling steps (e.g. IOS-XR's `_monitor_operation_log` with 600s timeout), cancel can take up to that long to land. The runner sets request status to `cancelled` immediately and marks all subsequent callbacks for the cancelled request to no-op via the generation token (see §8). The user observes `request.status = cancelled` once the in-flight RPC returns.

### 4.5 `clear-run-log` ↔ `clear-run-log-generation` increment (🆕 CL4)

**Trigger:** `clear-run-log-generation > memory.last_clear_run_log_generation`.

**Behavior:** drop all `run-log[]` entries for the request and reset the bounded-buffer cursor.

This complements (not replaces) the bounded retention: by default the log is capped at 1000 entries per request (configurable later), AND the user can explicitly clear between runs for forensic capture.

### 4.6 `enabled` semantics (🆕 C1)

`/software-install/enabled` precise semantics:

- **Pack-data changes still materialize requests** (status = `unprocessed`). User-visible "this should change" state is preserved.
- **No step execution starts** while `enabled = false`.
- **In-flight runner reaches a cooperative stop point** (current step or current RPC completes), then pauses with `request.status = waiting-confirmation` — wait, actually let's pick a distinct status. 🆕 Add `paused` to the request-status enum, to distinguish "operator disabled the system" from "waiting on user confirm".
- **Re-enable** resumes per `start-generation` if explicit-start mode, or auto-resume if `auto-execute-after-confirm`.

### 4.7 YANG diff vs the Python original

| Item | v2 plan |
|------|---------|
| `software-pack` list (global) | unchanged |
| `software-install-matrix` | drop, log in §15 deferred-features |
| `confirm-steps`, `auto-execute-after-confirm`, `error-handling/...` | unchanged |
| `request-status` enum | **add `paused`** (4.6) |
| `request[]` and below | **all `config false`** (oper-only); `confirmed` no longer writeable here |
| 🆕 `/devices/device/software-pack/control/*` subtree | **new**: generation counters + confirmations |
| 🆕 `/devices/device/software-pack/last-create-result` | **new**: oper-only feedback for create-request |
| 🆕 `scp-port` (per-device) | **kept** (C4) — under per-device sw-install subtree |
| Action nodes | **all dropped** (A1, A2) |
| `internal-state` leaf | **dropped** (CL14) |
| `vrp` enum value | **kept** (A9) — `ValidatePlatform` fails cleanly on unsupported |

---

## 5. Typed data model (gen_adata + internal value types)

Two layers — same as v1:

1. **`gen_adata`-generated typed accessors** for the YANG (`model.act` after build).
2. **Internal value types in `pack.act`** tuned for state-machine use (hashable as map keys, deep equality):

```acton
class SoftwarePackOs(value):
    IOSXR = "ios-xr"
    JUNOS = "junos"
    SROS = "sros"
    VRP = "vrp"           # 🆕 kept per A9; ValidatePlatform fails cleanly

class ComponentKind(value):
    BASE = 0
    PATCH = 1

class SoftwarePackComponent(value):
    kind: ComponentKind
    version: str
    filenames: list[str]
    os: SoftwarePackOs
    def name(self) -> str: ...        # base-<v> or patch-<v>

class SoftwarePack(value):
    name: str
    os: SoftwarePackOs
    base: ?SoftwarePackComponent
    patches: list[SoftwarePackComponent]
    def components(self) -> list[SoftwarePackComponent]: ...
```

State types (`state.act`) — **🆕 each per-OS subclass implements `reset()`** (CL9):

```acton
class State(value):
    device: str
    component: SoftwarePackComponent
    virtual_router: bool
    done: bool
    def reset(self) -> State: ...      # default: clear virtual_router

class GenericDevice(value):
    destination_volume: ?str
    destination_paths: dict[str, str]
    boot_time: ?datetime
    done: bool
    def reset(self) -> GenericDevice: ...

class StateSros(value):
    base: GenericDevice
    head: State
    version: ?str
    rebooted: bool
    def reset(self) -> StateSros: ...

class StateIosXr(value):
    base: GenericDevice
    head: State
    op_id_add: ?int
    op_id_prepare: ?int
    op_id_activate: ?int
    op_id_commit: ?int
    packages: list[str]
    reload_required: bool
    def reset(self) -> StateIosXr: ...
    # restart_prepare_clean / restart_prepare / restart_activate helpers per spec §5.3

class RouteEngine(value):
    base: GenericDevice
    version: ?str
    rebooted: bool

class StateJunos(value):
    head: State
    dual_re: ?bool                     # CL10: cross-run invariant — see §6
    switch: bool
    failover_config: ?value
    route_engine: dict[str, RouteEngine]
    route_engine_priority: list[int]
    def reset(self) -> StateJunos: ...
```

`reset()` returns a new value (Acton value-typed semantics) rather than mutating. Callers replace the State binding with the returned one.

---

## 6. Plan + step semantics

### 6.1 `Step` protocol

```acton
class StepResult(value):
    SUCCESS = 0
    FAILURE = 1
    SKIP_STEP = 2
    SKIP_COMPONENT = 3
    WAIT = 4

class StepKey(value):
    name: str
    re_id: ?str        # None for non-Junos and for trailing Done — CL12

class Step(object):
    key: StepKey
    proc def pre_check(self, state, ops: DeviceOps, ft: FileTransfer, cb: action(StepResult, NewState, ?Exception) -> None) -> None: ...
    proc def execute(self, state, ops: DeviceOps, ft: FileTransfer, cb: action(StepResult, NewState, ?Exception) -> None) -> None: ...
    def next_step(self, state) -> ?StepKey: ...
    def supports_pre_check(self) -> bool: ...
```

State is value-typed and threaded through the callback (`NewState`). The runner persists the new state only when `result != FAILURE` (CL8).

**🆕 `DeviceOps` is a per-OS facade** that hides whether a given operation goes through NETCONF or CLI. Steps call `ops.get_version(cb)` without caring which transport answers. See §9.7.

### 6.2 `ComponentPlan` invariants

**🆕 Refresh discipline (A8):** `refresh_steps` is called **after every step's `_execute_step_action`**, not only at run start. This is what enables IOS-XR FPDs to be discovered mid-run.

**🆕 Monotonicity (A8):** the refresh may **add** new steps but **must not remove** prior components or steps. A removal indicates a logic bug; the runner raises.

**🆕 Flush ordering (CL8):**
1. Step's `pre_check` or `execute` returns (StepResult, NewState).
2. If `result != FAILURE`: persist NewState into the per-component State store.
3. If `result == FAILURE`: discard NewState. Mark step `failed`. Mark all subsequent steps in this component back to `not-reached` (or `waiting-confirmation` if confirm-mode).
4. Refresh the plan.
5. Persist the plan.

A FAILURE result must NOT persist the half-mutated NewState.

### 6.3 Per-OS step lists — fidelity notes

- **SROS** (`platform_sros.act`): step list per logic doc §6.1. **No `Cleanup` step** (CL18).
- **IOS-XR** (`platform_iosxr.act`): step list per logic doc §6.2. **`Cleanup` is IOS-XR only.** `CheckVersions` requires controller-side archive parsing (`.iso` via `get_version_from_iso`, `.tar` via `get_file_packages`); 🆕 this is acknowledged as a Phase 5 dependency (CL11) — needs an iso/tar parser on the controller.
- **Junos** (`platform_junos.act`): per logic doc §6.3. **`StepKey(re_id=None)` for the trailing unparameterized `Done`** (CL12). 🆕 **Cross-run invariant (CL10):** if `state.dual_re` changes between runs of the same request, `ValidatePlatform` returns FAILURE and invalidates the request.
- **VRP**: enum kept (A9). `ValidatePlatform` fails cleanly with "unsupported platform". No step module wired.

### 6.4 Status mapping at run end (🆕 A4)

The Python `_write_request_status` rules exactly:

```
if request.obsolete:                             status = OBSOLETE
elif all components.completed:                   status = DONE; clear error_count
elif needs_confirmation:                         status = WAITING_CONFIRMATION
elif last_result == WAIT:                        status = FAILED_TRANSIENT
                                                 error_count.transient += 1
                                                 error_count.other = 0
elif last_result == FAILURE (or other non-success):
                                                 status = FAILED_OTHER
                                                 error_count.other += 1
                                                 error_count.transient = 0
```

Counters are **consecutive**, not cumulative — FAILURE resets `transient`, WAIT resets `other`.

### 6.5 Retry exhaustion (🆕 A4)

The retry-budget check is per-class:

- WAIT exhausted (`error_count.transient > config.max_retries` after a WAIT) → terminate as `FAILED_TRANSIENT`.
- FAILURE exhausted (`error_count.other > config.max_retries` after a FAILURE) → terminate as `FAILED_OTHER`.

Do **not** check `transient + other > max_retries` — that drifts from spec.

### 6.6 Run-log filtering (🆕 A10)

Only log records bearing the `swi_component` attribute are persisted. The runner installs a per-step logging handler that adds `swi_request_path / swi_component / swi_step / swi_run_id` to records emitted within the package's logger context. Records flowing past from other modules are dropped at the persistence boundary. Bounded ring buffer at 1000 entries/request by default, plus explicit `clear-run-log-generation` (§4.5).

---

## 7. The Transform (or alternative)

### 7.1 First plan: `Transform` substrate (preferred)

```acton
class SwInstallTransform(ttt.TransformFunction):
    def transform_wrapper(self, cfg, linked, memory, dynstate):
        # 🆕 Crucial: compute new memory from cfg so create-request idempotency
        # survives transform restart (A6).
        new_memory = compute_memory_snapshot(cfg, memory)
        return (gdata.Container(), new_memory)


def make_sw_install_transform(dev_registry, file_transfer_factory=None, log_handler=None):
    proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
        sw_runner = SwInstallRunner(
            params.path,
            params.update_oper,
            params.update_dynstate,
            dev_registry,
            file_transfer_factory,
            log_handler,
        )
        return lambda cfg, mem: sw_runner.on_conf(cfg, mem)
    return ttt.Transform(SwInstallTransform, act=act, log_handler=log_handler)
```

**Pros:** free `update_oper` / `update_dynstate` plumbing, gen_adata wires the YANG types in, integrates with `ttt.Layer.get_data(...)` naturally.

**Cons:** we don't produce downward config (returning empty `Container` from `transform_wrapper`). This pattern is unusual — sw-install is a *parallel control loop*, not part of the data path. There is no precedent in stratoweave for "transform that's a sink/observer."

### 7.2 Fallback: top-level actor wired at app-spec time (🆕 D2)

If the `Transform` substrate proves awkward (gen_adata rejects the empty-output shape, or the platform owners object), the alternative is:

- `SwInstallApp` actor instantiated by the app's wiring code at startup, alongside `DeviceRegistry`.
- It calls `layer.declare_subscriptions(...)` (an existing platform mechanism — see `ttt.act:735`) to observe the relevant config subtree.
- It exposes a `gdata.TreeProvider` (existing pattern — see `device.act:207` `DeviceTreeProvider`) so its operational state is queryable like any other layer's data.
- It owns persistence directly via lmdb without going through transform-level dynstate.

**❓DECISION (round-2 question):** does the platform owner / reviewer prefer 7.1 or 7.2? 7.1 is faster to build if it works; 7.2 is more honest about the architecture but more glue code. **I'm starting with 7.1 and will switch if `gen_adata` integration shows real friction.**

---

## 8. DeviceRunner architecture (🆕 D1 — flipped from per-request to per-device)

### 8.1 The problem this solves

Per-request runners cannot guarantee per-device exclusivity (A3). `DeviceMgr` actor-mailbox serialization gives only message-level mutex; an outstanding `rpc_xml` can interleave with other RFS configures or unrelated rpc_xml. The Python `MaapiLocker.lock_partial(/ncs:devices/ncs:device[name=X])` held across the entire run.

### 8.2 The shape

**One `DeviceRunner` actor per device.** Internally it owns at most one active `RequestState` value:

```acton
class RequestState(value):
    request_id: u32                              # starts at 1 — CL13
    pack: SoftwarePack
    confirm_steps: bool                          # may override global
    plan: ComponentPlan
    states: dict[str, State]                     # per-component
    status: RequestStatus
    run_id_count: u64
    run_log: list[RunLogEntry]                   # bounded
    error_count: ErrorCount                      # consecutive — A4
    next_wake_at: ?datetime                      # 🆕 CL5: persistent backoff
    generation_token: u64                        # 🆕 CL7: stale-callback no-op
    last_observed_generations: GenerationLastSeen


actor DeviceRunner(
    device_name: str,
    dev: swdev.DeviceMgr,
    file_transfer: ?FileTransfer,
    publish: action(DeviceOper) -> None,
    log_handler: ?logging.Handler,
):
    var current: ?RequestState = None
    var history: list[RequestState] = []         # for the request[] oper projection (bounded)

    action def on_config_change(cfg: SwInstallDeviceConfig):
        # All trigger-detection and request lifecycle is in here.
        ...

    action def _start_run():
        # Bumps run_id_count; runs the step loop with a bound generation_token.
        ...

    action def _step_callback_guard(token: u64, then: action() -> None):
        # 🆕 CL7: every step callback re-enters via this guard.
        # If token != current.generation_token, the callback is stale; no-op.
        ...
```

**Why one per device, not per request:**
- The actor's *existence* IS the install lease for the device.
- Per-request races (CL7) cannot occur — only one request is active per device at a time; new pack-data arriving mid-run preempts via generation token.
- Cancellation is a state transition on the runner, not actor lifecycle management.

### 8.3 Generation tokens for stale-callback discipline (🆕 CL7)

Every `RequestState` carries a `generation_token: u64`. Every step callback closes over the token at scheduling time and checks it on entry against `current.generation_token`. If they differ — because a newer pack-data arrived, or a cancel happened, or a confirm fired and the runner re-derived state — the callback no-ops.

This handles:
- Two pack-data updates in quick succession: the first runner is preempted; its outstanding callbacks no-op when they return.
- Cancel during a long-poll: status updates immediately; callback no-ops on completion.
- Restart: tokens reset; re-spawned runner carries a fresh token; old persisted callbacks (there shouldn't be any, but defensive) no-op.

### 8.4 Restart story (🆕 CL15)

`DeviceRunner` is a child actor of the transform's `act`-spawned runner. It is **not** a transform itself, so `update_dynstate` is plumbed *up* through the transform.

The lifecycle:

1. **At runtime:** every state-bearing change (request transition, plan update, error counter change, `next_wake_at` set) emits a serialized snapshot via `update_dynstate(...)` to the parent transform.
2. **At restart:** the platform restores the transform's dynstate from lmdb. The transform's `act` callback inspects the restored state on first invocation and **respawns DeviceRunners** with their persisted `RequestState` injected at construction time.
3. **Backoff resume (🆕 CL5):** if a restored `RequestState.next_wake_at` is in the future, the runner schedules a fresh `after max(0, next_wake_at - now): _start_run()`. If it's in the past, it kicks off immediately.

This satisfies spec §10 invariant 2 ("step_executor is restartable") concretely. **⚠️ASSUMPTION:** the dynstate-bubbling-from-actor-to-transform pattern is feasible — I have not yet verified `update_dynstate` accepts updates from arbitrary actor depths. **Round-2 question for the platform reviewer.**

### 8.5 Cancel semantics (🆕 CL6)

On `cancel-generation` advance:
1. Set `current.status = CANCELLED` immediately and publish via `update_oper`.
2. Bump `current.generation_token` so all in-flight callbacks no-op on return.
3. Log "cancellation requested at <step>" to the run-log.

The user-observable sequence: status flips to `CANCELLED` immediately, but the most recent log entry from the in-flight RPC may continue to land for up to the RPC's natural timeout (e.g. 600s for IOS-XR). This is documented behavior; explicit, not surprising.

### 8.6 Backoff (🆕 CL5)

Internal to `DeviceRunner`. After a `failed-other` / `failed-transient` run end:

```
if error_count.<class> > config.max_retries:    # per-class budget — A4
    publish terminal status, gave_up = True
    return
backoff = config.backoff.factor * max(error_count.backoff, 10)   # OR config.backoff.constant
error_count.backoff = backoff
next_wake_at = now() + backoff
update_dynstate(...)                            # PERSIST before scheduling
after backoff: _start_run()
```

Persisted before scheduling, so a controller restart respects the backoff.

### 8.7 What about parallel runs across different devices?

Trivially supported: one DeviceRunner per device, all live in parallel under the same `SwInstallRunner`. The `SwInstallRunner` actor (transform's `act`-spawned root) is a coordinator that owns `dict[device_name, DeviceRunner]` and dispatches incoming config changes per device.

---

## 9. Transport scope: NETCONF first, file transfer pluggable

### 9.1 Op coverage matrix

| Op | SROS | IOS-XR | Junos |
|----|------|--------|-------|
| version, redundancy, boot time, free space | NETCONF state datastore | NETCONF live-status oper | NETCONF RPCs |
| install/activate/commit | NETCONF actions | NETCONF actions | `request package add` via NETCONF |
| BOF reconfiguration | NETCONF edit-config (`bof-running`) | n/a | n/a |
| **Image upload** | Phase 5 (FileTransfer) | Phase 5 | Phase 5 (also cross-RE `file copy`) |
| `prepare_format_standby` (SROS) | NotImplementedError → SKIP_STEP | n/a | n/a |
| `prepare_save_rollback` (SROS) | NotImplementedError → SKIP_STEP | n/a | n/a |
| **Archive parsing (IOS-XR)** | n/a | 🆕 Phase 5: controller-side iso/tar libs (CL11) | n/a |
| Interactive CLI (prompts, confirmations, reboot, log scraping) | Phase 5 (TextFSMPlus templates — §9.7) | Phase 5 (TextFSMPlus) | Phase 5 (TextFSMPlus) |

### 9.2 The `FileTransfer` interface (🆕 typed; C2 / C3)

```acton
class RemoteFileInfo(value):
    path: str
    size: ?u64
    checksum: ?str
    checksum_algorithm: ?str       # e.g. "md5", "sha256"
    mtime: ?str                    # yang:date-and-time

class FileTransferCaps(value):
    put: bool
    delete: bool
    checksum: bool
    device_pull: bool              # device fetches from a URL (preferred where supported)

class FileTransfer(object):
    proc def caps(self) -> FileTransferCaps: ...
    proc def stat(self, path: str, cb: action(?RemoteFileInfo, ?Exception) -> None) -> None: ...
    proc def list(self, dir: str, cb: action(list[RemoteFileInfo], ?Exception) -> None) -> None: ...
    proc def put(self, local_path: str, remote_path: str, cb: action(?Exception) -> None) -> None: ...
    proc def delete(self, path: str, cb: action(?Exception) -> None) -> None: ...
```

Phase 4 ships **`NoopFileTransfer`**: every method except `caps()` returns `NotImplementedError`; `caps()` returns all-false. `CopyImage.pre_check` returns `SKIP_STEP` if all files are already present (verified via `stat`/`list`), or `FAILURE` with a clear "no FileTransfer configured" message if files are missing — **🆕 fixing the SUCCESS/SKIP_STEP slip in v1 (CL16).**

### 9.3 Steps that consume `FileTransfer`

- **`CopyImage` (SROS, IOS-XR, Junos)**: `pre_check` uses `stat` + size/checksum match; `execute` uses `put`. Verifies via `stat` after `put`.
- **`Cleanup` (IOS-XR only — CL18)**: `delete` for each `state.destination_paths` entry. **SROS does not have a Cleanup step.** Junos's per-RE cleanup is an open question for Phase 5.

### 9.4 Credential reuse — three options, decision (🆕 CL3)

The current code reality: `DeviceMetaConfig.credentials` is owned by `DeviceMgr` and injected into the *adapter* via `set_dmc(...)`. Neither `DeviceMgr` nor `DeviceAdapter` exposes credentials publicly. A `FileTransfer` constructed with only `swdev.DeviceMgr` cannot obtain credentials.

Three options:

(a) **Extend `DeviceAdapter` ABI with `get_credentials()`** — a small, focused public API addition. Pro: minimal surface change, easy to mock. Con: leaks credentials out of the adapter; tests can read them.

(b) **Pass `DeviceMetaConfig` at the factory boundary** (`file_transfer_factory(dev, dmc)`). Pro: no DeviceAdapter ABI change. Con: requires the transform to obtain DMC, which is not currently visible to non-RFS transforms; needs platform-side plumbing.

(c) **Piggyback file transfer on the netconf adapter's existing SSH transport.** The transport is already authenticated; spawn an SCP/SFTP channel on the same SSH connection. Pro: no credential exposure, no DMC plumbing, no key/password duplication, single point of truth for auth. Con: requires `netconf/src/ssh_client.act` to expose a "spawn channel" affordance (it currently doesn't — see Claude review item 8-11).

**Decision:** prefer **(c) where supported**, fall back to **(b)** with explicit DMC plumbing through the transform. (a) is rejected — credential leakage out of the adapter is a long-term smell.

The Phase 4 stub (`NoopFileTransfer`) sidesteps this entirely. The Phase 5 implementation will require **either** a one-line `netconf` package extension (option c) **or** transform-side DMC visibility (option b). Both will go to a platform-owner conversation; the design path is clear.

### 9.5 `scp-port` preservation (🆕 C4)

The original per-device augmentation has `scp-port? inet:port-number`. The Python code uses it as the SCP server port (defaulting to the SSH port). We **preserve it on the per-device sw-install subtree** as a hint to the FileTransfer impl (option (c) reads it; option (b) reads it). YANG:

```yang
container software-pack {
    presence "...";
    leaf name { ... }
    leaf scp-port { type inet:port-number; }
    container control { ... }
    ...
}
```

### 9.6 YANG filename interpretation (🆕 CL17)

The `filename` leaf-list path interpretation is **defined by the configured `FileTransfer` implementation**. NETCONF-only Phase 4 (NoopFileTransfer) requires the file to already be on the device at the given path. SCP-based Phase 5 expects controller-local paths. Device-pull expects a device-reachable URL. The YANG description states this explicitly so users know what to write.

### 9.7 CLI driver — `DeviceOps` facade and TextFSMPlus templates (🆕)

The Python `software_install_script.py` mixes NETCONF and CLI per-OS — `NokiaSrosCliStrategy` vs `NokiaSrosNetconfStrategy` both implement `NokiaSrosOperationsProto`, and `NokiaSrosOperations` delegates to whichever strategy the device's capabilities select. We preserve this strategy pattern in the Acton port; the difference is that we anchor the CLI side on a **TextFSMPlus-style template engine** rather than ad-hoc match-and-respond loops.

#### Why TextFSMPlus templates for CLI driving?

Standard TextFSM is parse-only — extract structured values from `show <foo>` output. The Python original uses ad-hoc string matching for the *interactive* parts (prompts like `(y/n)`, `[confirm]`, `Are you sure?`). That's a recurring pattern across every CLI-driving step on every OS.

TextFSMPlus (a TextFSM superset, original implementation in `/Users/ayourtch/rust/ayclic/aytextfsmplus`) adds three line actions on top of standard TextFSM that turn it into an Expect-style language:

- **`Send "..."`** — send text to the stream (with variable interpolation via `${...}` expressions).
- **`Preset`** — caller-supplied values fed into the engine before it runs (commands, parameters, credentials).
- **`Done`** — terminal state signaling successful completion of an interactive sequence.

The same engine drives both **parsing** (existing TextFSM templates from ntc-templates work unmodified — battle-tested at 1790/1818 in the reference Rust implementation) and **interactive sessions** (new templates with `Send`/`Preset`/`Done` to handle prompts).

Concrete example — driving SROS `admin reboot now`:

```
Value Preset Confirm (yes|no)

Start
  ^.*Are you sure.* -> Send ${Confirm} WaitGoodbye

WaitGoodbye
  ^.*[Cc]onnection.*closed -> Done
```

Same engine, same template shape. No separate Expect library, no ad-hoc match loops in step bodies.

#### `DeviceOps` facade

```acton
# ops_sros.act
class DeviceOps(object):
    """Per-OS operations facade. Hides NETCONF/CLI strategy choice from steps."""
    proc def get_version(self, cb: action(?str, ?Exception) -> None) -> None: ...
    proc def get_boot_time(self, cb: action(?datetime, ?Exception) -> None) -> None: ...
    proc def get_redundancy_info(self, cb: action(?(str, str), ?Exception) -> None) -> None: ...
    proc def reboot(self, cb: action(?bool, ?Exception) -> None) -> None: ...
    # ... all per-OS device interaction primitives

actor SrosOps(
    dev: swdev.DeviceMgr,
    cli_session: ?CliSession,         # None means NETCONF-only mode
    log_handler: ?logging.Handler,
):
    var prefer_cli: bool = False     # set by capability-based strategy selection at construction

    proc def get_version(self, cb):
        if prefer_cli and cli_session is not None:
            cli_session.run_template(SROS_SHOW_VERSION_TEMPLATE, ..., cb_parse_version(cb))
        else:
            dev.rpc_xml(..., cb_parse_netconf_version(cb))

    # ... and so on
```

`SrosOps` decides per-method (or per-device, at construction) which strategy to use based on `dev.get_capabilities()`. The Python `_NOKIA_NS_MAP` dispatch logic moves into Acton as a small constructor-time decision.

For Phase 4 (NETCONF-only on SROS): `cli_session` is `None` and every method goes through NETCONF. The CLI strategy code paths are typed and present, but `cli_session is None` short-circuits them.

#### `CliSession` — the TextFSMPlus driver

```acton
# cli_session.act
class TextFsmPlusTemplate(value):
    """Compiled TextFSMPlus template — Send/Preset/Done extensions over standard TextFSM."""
    states: dict[str, StateRules]
    values: dict[str, ValueDef]    # incl. Preset declarations

class CliSession(object):
    """Wraps an SSH/Telnet stream; runs TextFSMPlus templates over it.

    Parse mode: feed bytes, extract DataRecords. Interactive mode: feed bytes,
    on Send action emit text downstream, on Done complete successfully.
    """
    proc def run_template(self,
                          tmpl: TextFsmPlusTemplate,
                          presets: dict[str, str],
                          cb: action(?TextFsmPlusResult, ?Exception) -> None
                         ) -> None: ...
    proc def close(self) -> None: ...
```

Backed by the existing `netcli` / `netclics` packages for the underlying SSH/Telnet stream. Template execution is async — Send actions write to the stream, then the engine waits for the next match.

#### Templates as data, shipped with per-OS modules

Each `ops_<os>.act` ships templates as multi-line string constants — same pattern as the ayclic `templates.rs`. Templates are reusable across orchestration tools (the format is broadly compatible with the rapidly-growing TextFSM ecosystem; ntc-templates output parsing slots in unmodified).

#### Phase 4 vs Phase 5

| Item | Phase 4 | Phase 5 |
|------|---------|---------|
| `DeviceOps` facade in code | ✅ shipped | unchanged |
| `SrosOps` NETCONF strategy | ✅ shipped | unchanged |
| `SrosOps` CLI strategy (methods exist, raise `NotImplementedError`) | ✅ stub | ⚠️ implemented |
| `CliSession` interface | ✅ defined | unchanged |
| `CliSession` real impl over netcli/netclics | ❌ | ⚠️ implemented |
| Acton TextFSMPlus extensions in `acton-utils textfsm` | ❌ — needs port | ⚠️ **dependency**: extend the `acton-utils textfsm` package with `Send`/`Preset`/`Done` line actions. Reference impl: `/Users/ayourtch/rust/ayclic/aytextfsmplus`. |
| `IosXrOps` / `JunosOps` | ❌ | ⚠️ implemented |

#### Why design CLI in now (rather than defer wholesale)

Two reasons: (1) the architecture impact of CLI on `DeviceOps` and `Step` signatures is small — but only if we know it's coming. Designing CLI in later would force step signatures to refactor. (2) Phase 4 is already in design-review iteration; the marginal review cost of "review with CLI included" is small compared to "do a separate Phase 5 review with the same iteration cycle." See `docs/reviews/03-integration.md` for the discussion that led to this call.

#### Acton TextFSMPlus port — design dependency

The acton-utils ecosystem already has standard TextFSM. Phase 5 CLI work depends on extending it with the three TextFSMPlus extensions (`Send`/`Preset`/`Done`) plus an aycalc-equivalent for `${...}` expression evaluation in `Send` actions. This is a **separate work item parallel to sw-install Phase 5** — owned by the platform side, not by sw-install itself.

The reference Rust implementation is mature (1790/1818 ntc-templates compatibility) and small enough to port directly. **❓DECISION (round-2 question for platform reviewer):** does the user / platform owner want this port to happen as part of sw-install Phase 5, or as a separate workstream that sw-install consumes?

Captured in §15 (deferred features).

---

## 10. Testing strategy

Unchanged from v1 in shape. **🆕 Pure tests called out by both reviewers (A8 / CL8 / CL10 / etc.):**

- `test_pack.act` — value tests; equality/hashing/`from_gdata` round-trips.
- `test_plan.act` — **plan refresh monotonicity, after-every-step trigger, next-step jump targets, flush-only-if-not-FAILURE invariant, status mapping (consecutive counters)**.
- `test_state.act` — per-OS `reset()` semantics, Junos `dual_re` cross-run invariant.
- `test_runlog.act` — bounded retention, `swi_component` filter, `clear-run-log-generation` semantics.
- `test_step_runner.act` — actor integration test with `MockAdapter`; scripted SROS install end-to-end; **stale-callback no-op via generation token**, restart resume of in-flight backoff.
- `test_sros.act` — per-step tests using a mock NETCONF server.

---

## 11. Implementation phasing (Phase 4)

1. **Skeleton + revised YANG** (this iteration): scaffolding, `Build.act`, `Makefile`, `gen_adata` flow, REUSE compliance, `yang.act` complete with control subtree + per-device augment.
2. **Types**: `pack.act`, `state.act`, `plan.act`, `step.act`, `file_transfer.act` — pure Acton, fully unit-tested.
3. **Transform shell**: `transform.act` + `device_runner.act` skeleton wires into a layer; on_conf events bookkeep generation tokens; oper publication round-trips. End-to-end "config in, no-op out" without device interaction.
4. **CheckFiles + CheckVersions for SROS** — first device-touching steps; mock-tested.
5. **All SROS steps** in order; mock integration test for full run.
6. **Restart test**: kill mid-step, verify re-spawn from dynstate continues correctly.
7. **(Phase 5)** IOS-XR + Junos + FileTransfer (option c or b) + polish.

---

## 12. Open decisions (round-2 questions)

| # | Question | My current lean |
|---|----------|-----------------|
| **❓Q1** | `Transform` substrate (§7.1) vs top-level actor + `TreeProvider` (§7.2)? Platform owner input would help. | Try `Transform` first. |
| **❓Q2** | Is `update_dynstate` from an actor deeper than the transform's `act`-spawned root (i.e., a child `DeviceRunner`) supported? Verifying §8.4. | Assuming yes; will verify. |
| **❓Q3** | Default value of `auto-execute-after-confirm`. Python defaults to `false` (operator-explicit start). | Keep `false`. |
| **❓Q4** | (a) `DeviceAdapter.get_credentials()`, (b) DMC at factory, (c) piggyback on netconf SSH? See §9.4. | (c) preferred, (b) fallback. |
| **❓Q5** | Run-log default bound (1000?). | 1000 entries/request. |
| **❓Q6** | acton-utils textfsm extension (Send/Preset/Done) — sw-install Phase 5 owns it, or separate workstream? See §9.7. | Separate workstream; sw-install consumes. |

Resolved from round-1:

- ❓1 (sibling vs in-tree): **sibling**. Confirmed.
- ❓2 (drop YANG actions): **dropped**, replaced with control subtree + reactive triggers (§4).
- ❓3 (cancel-requested leaf): **subsumed** by `cancel-generation` (§4.4).
- ❓4 (clear-run-log): **both** — bounded retention + explicit `clear-run-log-generation` (§4.5).
- ❓5 (step return shape): **value-typed `(StepResult, NewState)`** (§6.1).
- ❓6 (NETCONF-only / FileTransfer abstraction): **resolved** via §9.
- ❓7 (snapshot tests): no.
- ❓8 (high-level `attach_to(layer)` helper): no — just the factory.

---

## 13. What I am still not deciding

These resolve naturally during implementation:

- Exact lmdb key layout for runner dynstate (§8.4 has the *what*; the *layout* follows `_TransformTransactionBase.db_ops` patterns).
- `update_oper` snapshot frequency (every state change vs. coalesced per-tick).
- Logger handler implementation choices (Acton stdlib's logging may want a thin adapter for the `swi_component` filter — see §6.6).

---

## 14. (deleted from v2 — was the previous "Ready for review" section)

---

## 15. Deferred features (🆕 collected per reviewer requests)

Things deliberately **not** in this design, captured here so future contributors find them:

- **`software-install-matrix`** (Python YANG): unused by the Python core logic. Not modelled in the Acton port. Re-add if a real use case surfaces.
- **CLI driver via TextFSMPlus templates** (§9.7): `DeviceOps` facade and `CliSession` interface ship in Phase 4 as stubs. Real impl in Phase 5, depending on the **acton-utils textfsm extension** (Send/Preset/Done line actions + aycalc-equivalent expression evaluation) — itself a parallel workstream sourced from `/Users/ayourtch/rust/ayclic/aytextfsmplus` as the reference.
- **IOS-XR archive parsing** (`get_version_from_iso` / `get_file_packages`): controller-side iso/tar reading. Phase 5 IOS-XR support requires this dependency; design notes call out the cost.
- **Image upload (FileTransfer)**: Phase 5; `NoopFileTransfer` ships in Phase 4 to allow exercising the rest of the plumbing against pre-staged images.
- **VRP step module**: enum value preserved, `ValidatePlatform` fails clean. Implementation deferred indefinitely.
- **Snabb / ONS-TL1 / HGW**: dropped from the Acton port (logic doc §6.4). Dropped, not deferred.
- **Approval-required / multi-stage commit hooks**: out of scope for sw-install — that's the platform's job (`DeviceMetaConfig.approval_required`).

---

## 16. Ready for round-2 review

This v2 design is consistent with `01-software-install-logic.md` and incorporates round-1 review feedback per `03-integration.md`. Open architectural questions are listed in §12. The implementation order in §11 is unchanged from v1 in spirit; the work is the same, the abstractions are tightened.

**Stop here for round-2 review before resuming Phase 4 implementation.**
