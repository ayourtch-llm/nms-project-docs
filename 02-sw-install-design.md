# 02 — Stratoweave `software-install` Module Design (v3)

> **Status: v3 — incorporates round-2 review feedback from `docs/reviews/03-codex-design-r2.md` and `docs/reviews/04-claude-design-r2.md`, integrated in `docs/reviews/05-integration-r2.md`. Pending round-3 review before Phase 4 implementation begins.**

Read `00-orientation.md` (project context + stratoweave concepts) and `01-software-install-logic.md` (language-agnostic spec extracted from the Python source) first. Where v3 differs structurally from v2, the change is anchored to a review item like **A1**, **CR3**, **CL_R2_2** etc. — see `docs/reviews/05-integration-r2.md`.

Companion documents:
- `docs/adr/cli-driver.md` — CLI driver via TextFSMPlus templates (Phase 5 implementation detail; lifted out of this doc per round-2 reviewer guidance).

Markers used in this doc:
- **❓DECISION** — open question requiring user/team input
- **⚠️ASSUMPTION** — choice made absent input; should be sanity-checked
- **🆕** — added or substantially changed in v3

---

## 1. High-level shape

A new sibling repo under `stratoweave/`:

```
stratoweave/
└── sw-install/
    ├── Build.act                              # deps: stratoweave, netconf, yang
    ├── Makefile, README.md, REUSE.toml, LICENSES/, .gitignore
    ├── gen_adata/
    │   ├── Build.act
    │   └── src/gen_adata.act                  # YANG → typed Acton classes
    └── src/
        └── sw_install/
            ├── sw_install.act                 # public API
            ├── yang.act                       # raw YANG strings (input to gen_adata)
            ├── model.act                      # GENERATED — typed accessors
            ├── pack.act                       # SoftwarePack / Component value types
            ├── plan.act                       # ComponentPlan + StepStatus
            ├── state.act                      # State / StateSros / ...
            ├── step.act                       # Step protocol + StepResult
            ├── transform.act                  # the per-device Transform
            ├── device_runner.act              # actor DeviceRunner (per-device)
            ├── runlog.act                     # bounded run-log helpers + filter
            ├── local_file.act                 # 🆕 LocalFileInspector
            ├── remote_file.act                # 🆕 RemoteFileInspector
            ├── file_transfer.act              # FileTransfer interface
            ├── ops.act                        # 🆕 DeviceOps facade (NETCONF + CLI strategy)
            ├── platform_sros.act              # Phase 4 — SROS step impls
            ├── ops_sros.act                   # SROS DeviceOps impl (NETCONF strategy real; CLI strategy stubs)
            ├── platform_iosxr.act             # Phase 6
            ├── ops_iosxr.act                  # Phase 6
            ├── platform_junos.act             # Phase 6
            ├── ops_junos.act                  # Phase 6
            └── test_*.act                     # tests
```

The module is a **library**: apps opt in by adding the YANG to their layer stack and wiring the transform.

**Public API** (`sw_install.act`) — 🆕 simplified per round-2 (no top-level coordinator actor):

```acton
def make_sw_install_transform(
    dev_registry: swdev.DeviceRegistry,
    local_file_inspector: ?LocalFileInspector = None,         # default: real Acton-stdlib impl
    remote_file_inspector_factory: ?proc(...) -> RemoteFileInspector = None,  # default: per-OS NETCONF impl
    file_transfer_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer = None,
    cli_session_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> CliSession = None,
    log_handler: ?logging.Handler = None,
) -> proc(ttt.Path, ?ttt.Layer) -> ttt.Node

SOFTWARE_INSTALL_YANG: yang.Module
```

This factory is used to build a per-device transform (see §7).

---

## 2. Module boundary — what apps integrate against

Apps:

1. **Add the YANG model** to their lowest config layer (the layer where `/devices/device` lives).
2. **Wire one `Transform` per device entry** — sw-install attaches inside the device list, so each device gets its own transform instance with its own dynstate slice (Option B wiring; see §7).
3. **Optionally provide factories** for `FileTransfer`, `CliSession`, and the `RemoteFileInspector`. Phase 4 ships sane defaults: real `RemoteFileInspector` over NETCONF for SROS; `NoopFileTransfer` (refuses byte transfers); no CLI session.

App side:

```acton
import sw_install

def get_layers(dev_registry, log_handler, db):
    swi_factory = sw_install.make_sw_install_transform(
        dev_registry,
        log_handler=log_handler,
    )
    # The factory produces one transform per device entry; the host layer
    # composition wires it under each device's software-pack subtree
    # (sorespo's per-list-entry pattern — see §7).
    layer0 = ttt.Layer('0', my_t_0_with_swi(swi_factory), layer1, db)
```

---

## 3. Where state lives — corrected

Round-2 reviews showed the v2 split between transform `memory` and `dynstate` is incompatible with the platform: the runner has no path to write `memory`; only `transform_wrapper`'s return value updates it (`ttt.act:1942–1962`). v3 collapses to a single ownership rule.

### 3.1 The ownership rule (🆕 A1)

| Surface | Contents | Mutability |
|---------|----------|------------|
| **Config (gdata)** | desired pack assignment, control triggers (generations + confirmations + per-request overrides) | external read/write |
| **Dynstate (gdata persisted via `update_dynstate`)** | **all** runner-owned operational state: last-observed pack-snapshot, last-observed generation counters, request-id counter, plan, per-component State, error counters, `next_wake_at`, run-log buffer, request history (bounded) | runner-internal |
| **Oper (gdata published via `update_oper`)** | pure projection of dynstate plus computed view (status, plan, run-log, last-create-result, diagnostic projections of internal state — destination_volume, op_id_*, etc.) | external read |
| **Transform `memory`** | **unused.** `transform_wrapper` returns `(empty, memory)` unchanged. | n/a |

**Why**: split lifecycle state across `memory` + `dynstate` + actor-private fields creates consistency bugs at restart (the platform restores them at different times via different paths) and surfaces no useful invariants. Putting everything in dynstate gives one source of truth, one restore path, one consistency surface.

### 3.2 Per-device dynstate schema

```acton
class SwInstallDynstate(value):
    # Idempotency anchors
    last_pack_snapshot: ?SoftwarePack          # last materialized pack-data
    last_request_generation: u64               # last consumed control trigger
    last_start_generation: u64
    last_cancel_generation: u64
    last_confirm_all_generation: u64
    last_clear_run_log_generation: u64

    # Request lifecycle
    next_request_id: u32                       # always >= 1; CL_R2_11
    current: ?RequestState                     # at most one active request
    history: list[RequestState]                # bounded — see §3.3
```

`RequestState` (also dynstate-resident, hashable for dict use):

```acton
class RequestState(value):
    request_id: u32
    pack: SoftwarePack
    confirm_steps: bool                        # snapshot of effective policy at materialization (per-request override resolved here)
    plan: ComponentPlan
    states: dict[str, State]                   # per-component
    status: RequestStatus
    run_id_count: u64
    run_log: list[RunLogEntry]                 # bounded ring; see §6.6
    run_log_dropped: u64                       # total entries dropped from the ring
    error_count: ErrorCount                    # consecutive transient/other; CR3 also surfaces in oper
    next_wake_at: ?datetime                    # CR3 also surfaces in oper
    generation_token: u64                      # bumped on each new run / cancel; stale-callback guard
    obsolete: bool
```

### 3.3 Request history retention (🆕 CR1)

`history` retains:

- the latest non-terminal request unconditionally (drives idempotency comparisons against pack-data);
- the latest of each terminal status (`done`, `cancelled`, `failed-other`, `failed-transient`, `obsolete`) — gives operators a "what happened most recently" view per outcome;
- additional older entries up to a bound of 50 total (configurable in v2.0+).

When pruning, never drop the entry holding the most recent `last_pack_snapshot` (its pack data is the idempotency baseline). If pruning would conflict with idempotency, retain the entry and prune older ones first.

### 3.4 Diagnostic projection of dynstate into oper (🆕 CL_R2_2)

The Python `internal-state` leaf is dropped from the v3 YANG. To recover the operability — operators debugging via NETCONF/RESTCONF still need to see what's happening — selected dynstate fields project into the oper subtree as read-only diagnostic leaves:

- per request `error-count/{transient, other, backoff}` (CR3)
- per request `next-wake-at` (CR3)
- per component `destination-volume`, `destination-paths` (small map, key=local-file, value=remote-path)
- per component `boot-time`
- per component `op-id-add`, `op-id-prepare`, `op-id-activate`, `op-id-commit` (IOS-XR)
- per component `rebooted` (SROS, Junos)
- per Junos RE: `version`, `boot-time`, `rebooted`

These are documented diagnostic leaves, not opaque blobs. Spec §15.5 (conscious deviations) lists this as a deliberate change from the Python `internal-state` JSON.

### 3.5 Generation counter restore semantics (🆕 CL_R2_6)

If config is restored from backup, the user-facing `<trigger>-generation` counter goes backward relative to the runner's persisted `last_<trigger>_generation`. The runner's `current > last_observed` check then evaluates false until the user manually bumps the counter past the persisted value.

**Documented behavior** (in the YANG description and a §15.5 entry): generation-counter triggers are not designed to survive a config-only restore that doesn't restore dynstate alongside it. Recommended operator practice: restore both config and dynstate together (lmdb backup). If only config is restored, bump each generation counter to a value larger than the previous high-water mark visible in oper before relying on the trigger semantics.

⚠️ASSUMPTION: a future platform-side "config restore" event hook would let us reset all `last_observed_*` counters automatically. Out of scope for v1; flag as v2.0 platform prerequisite.

---

## 4. Control surface — reactive triggers replacing NSO actions

The Python package exposes five NSO actions: `create-request`, `execute-request`, `cancel-request`, `confirm-step`, `clear-run-log`. v3 keeps the v2 generation-counter pattern, with two additions per round-2: per-request scoping and `cancelling` enum.

### 4.1 `create-request` ↔ pack-data change OR `request-generation` increment

**Trigger:** `(pack-data ≠ dynstate.last_pack_snapshot) OR (request-generation > dynstate.last_request_generation)`.

**Behavior:**
- Materialize a new request with id = `dynstate.next_request_id` (CL_R2_11; starts at 1).
- Bump `dynstate.next_request_id`.
- Mark `dynstate.current` (if any) as obsolete; move to `dynstate.history`.
- Snapshot pack-data into `RequestState.pack`.
- Update `dynstate.last_pack_snapshot` and `dynstate.last_request_generation` **before** any work begins (A4 invariant: durable trigger consumption).
- Publish via `update_oper`, including `last-create-result = {request-id, status: "new-request"|"existing-request", at: <now>}`.

**Cancelled-reactivation** (preserves Python's "cancelled forces new" rule): pack-data unchanged + last request was `cancelled` → user must increment `request-generation` to force a fresh request. Documented; covers the case where an operator wants to retry a cancelled install without changing the pack.

### 4.2 `execute-request` ↔ `start-generation` (preserves stage-and-review)

`unprocessed → processing` requires either:
- `start-generation > dynstate.last_start_generation`, **OR**
- `auto-execute-after-confirm = true` AND all required confirmations are in place.

Default `auto-execute-after-confirm = false` matches the Python explicit-start posture.

### 4.3 `confirm-step` ↔ writeable confirmations under `control/`

Per-step confirmations live in config:

```yang
container control {
    list confirmation {
        key "request-id component step";
        leaf request-id { type uint32; }
        leaf component { type string; }
        leaf step { type string; }
        leaf by-user { type string; }
    }
    leaf confirm-all-generation { type uint64; default 0; }
    // ... see §4.7 for the full list
}
```

When the runner reaches a step in `waiting-confirmation` and a matching `confirmation` exists in config, the step proceeds. The runner stamps `confirmed.{by-user, when}` in the oper projection.

`confirm-all-generation` increment expands internally to confirmations for every step of the targeted request (see §4.6 scoping).

### 4.4 `cancel-request` — state machine with `cancelling` enum (🆕 A5)

**Trigger:** `cancel-generation > dynstate.last_cancel_generation`.

**State transitions:**

```
processing  ──(cancel-generation +1)──▶  cancelling
                                            │  (in-flight RPC drains;
                                            │   generation_token bumped;
                                            │   subsequent step callbacks no-op)
                                            ▼
                                        cancelled
```

The user-observable contract is honest: `cancelling` means "we've received the signal, an RPC may still be in flight, status will resolve to `cancelled` when it drains." For long-poll steps (IOS-XR `_monitor_operation_log` with 600s timeout) this transition can take that long; operators see exactly what's happening.

A subsequent `request-generation` increment after `cancelled` reactivates per §4.1 cancelled-reactivation.

### 4.5 `clear-run-log` ↔ `clear-run-log-generation` (🆕 CL4)

**Trigger:** `clear-run-log-generation > dynstate.last_clear_run_log_generation`.

**Behavior:** drop all `run-log[]` entries for the targeted request (see §4.6) and reset `run_log_dropped` to 0.

Complements (does not replace) the bounded ring buffer (§6.6).

### 4.6 Per-request scoping for triggers (🆕 generation-counter scoping per CR/§4)

Each generation counter has an optional companion `target-request-id` leaf. If set, the trigger applies to that specific request id only (errors silently if no such id exists in current+history). If unset, the trigger applies to the device's latest request at observation time.

```yang
container control {
    leaf request-generation { type uint64; default 0; }
    leaf request-target-id { type uint32; }   // optional; absent = "latest"

    leaf start-generation { type uint64; default 0; }
    leaf start-target-id { type uint32; }

    leaf cancel-generation { type uint64; default 0; }
    leaf cancel-target-id { type uint32; }

    leaf confirm-all-generation { type uint64; default 0; }
    leaf confirm-all-target-id { type uint32; }

    leaf clear-run-log-generation { type uint64; default 0; }
    leaf clear-run-log-target-id { type uint32; }

    list confirmation { ... }                 // per §4.3
    container request-options {
        leaf confirm-steps { type boolean; }  // per-request override; CR5
    }
}
```

**Why optional companion leaves rather than mandatory**: the common case is "operate on the current request"; mandatory scoping would burden interactive use. For automation safety, automation tooling should always set `*-target-id` to remove the read-modify-write race window.

### 4.7 `enabled` semantics — state transition table (🆕 A5/C1)

`/software-install/enabled` is the master switch. Effects:

| `enabled` transition | Current request status | New request status | Action |
|----------------------|------------------------|--------------------|--------|
| `true → false` | `unprocessed` | `unprocessed` | nothing in flight; new requests still materialize, but won't start |
| `true → false` | `processing` | `paused` | `generation_token` bumps (stales out in-flight callbacks); current step's RPC may still complete (best-effort cooperative pause); no further steps execute |
| `true → false` | `failed-transient` (mid-backoff, `next_wake_at` set) | `paused` | `next_wake_at` retained; backoff doesn't fire while disabled |
| `true → false` | `waiting-confirmation` / `cancelling` / terminal | unchanged | nothing to pause |
| `false → true` | `paused` | `processing` (if `start-generation` was already consumed for this request) OR `unprocessed` (if not) | resume per-policy |
| `false → true` | other | unchanged | normal triggers apply |

**Disabled-then-cancel:** cancel still works while `enabled = false`. `cancelling → cancelled` once any in-flight RPC drains.

### 4.8 YANG diff vs Python original

| Item | v3 plan |
|------|---------|
| `software-pack` list (global) | unchanged |
| `software-install-matrix` | dropped from YANG; not modelled in Acton; re-add only if a real use case surfaces (CL_R2_12) |
| `confirm-steps`, `auto-execute-after-confirm`, `error-handling/...` | unchanged |
| `request-status` enum | **add `paused`, `cancelling`, `waiting-for-device`** (4.4, 4.7, CL_R2_10) |
| `request[]` and below | **all `config false`** (oper-only); confirmations live in `control/confirmation` |
| 🆕 `/devices/device/scp-port` | **direct device augmentation** (CL_R2_5) — not under `software-pack/` — survives unbinding the pack |
| 🆕 per-device `/devices/device/software-pack/control/` subtree | **new** — generation counters + target-ids + confirmations + per-request options |
| 🆕 per-device `/devices/device/software-pack/last-create-result` | oper-only feedback for create-request |
| 🆕 per-component `internal-state` leaf | **dropped** (CL14) — replaced by typed diagnostic projections (§3.4) |
| Action nodes | **all dropped** — covered by reactive triggers + control surface |
| `vrp` enum value | **kept** (A9) — `ValidatePlatform` fails cleanly on unsupported |
| 🆕 `run-log` key | `(when, seq)` — sequence number breaks microsecond-collision ties (CR10) |

---

## 5. Typed data model

Two layers — same as v2:

1. **`gen_adata`-generated typed accessors** for the YANG (`model.act` after build).
2. **Internal value types** in `pack.act` / `state.act` tuned for state-machine use.

### 5.1 Pack types (`pack.act`)

```acton
class SoftwarePackOs(value):
    IOSXR = "ios-xr"
    JUNOS = "junos"
    SROS = "sros"
    VRP = "vrp"           # kept; ValidatePlatform fails clean

class ComponentKind(value):
    BASE = 0
    PATCH = 1

class SoftwarePackComponent(value):
    kind: ComponentKind
    version: str
    filenames: list[str]
    os: SoftwarePackOs
    def name(self) -> str: ...

class SoftwarePack(value):
    name: str
    os: SoftwarePackOs
    base: ?SoftwarePackComponent
    patches: list[SoftwarePackComponent]
    def components(self) -> list[SoftwarePackComponent]: ...
```

### 5.2 State types (`state.act`)

Per-OS State subclasses each implement `reset()` per spec §5.1 (CL9):

```acton
class State(value):
    device: str
    component: SoftwarePackComponent
    virtual_router: bool
    done: bool
    def reset(self) -> State: ...

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
    # restart_prepare_clean / restart_prepare / restart_activate per spec §5.3

class RouteEngine(value):
    base: GenericDevice
    version: ?str
    rebooted: bool

class StateJunos(value):
    head: State
    dual_re: ?bool                     # cross-run invariant — see §6.3
    switch: bool
    failover_config: ?value
    route_engine: dict[str, RouteEngine]
    route_engine_priority: list[int]
    def reset(self) -> StateJunos: ...
```

`reset()` returns a new value (Acton value-typed semantics) rather than mutating; callers replace the State binding.

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
    re_id: ?str        # None for non-Junos and for trailing Done

class Step(object):
    key: StepKey
    proc def pre_check(self, state, ops: DeviceOps, lfi: LocalFileInspector,
                       rfi: RemoteFileInspector, ft: FileTransfer,
                       cb: action(StepResult, NewState, ?Exception) -> None) -> None: ...
    proc def execute(self, state, ops: DeviceOps, lfi, rfi, ft,
                     cb: action(StepResult, NewState, ?Exception) -> None) -> None: ...
    def next_step(self, state) -> ?StepKey: ...
    def supports_pre_check(self) -> bool: ...
```

🆕 Step methods receive **four** abstractions:
- `ops: DeviceOps` — per-OS facade for NETCONF/CLI device operations (§9.7)
- `lfi: LocalFileInspector` — controller-side filesystem checks (§9.2)
- `rfi: RemoteFileInspector` — device-side file metadata via NETCONF (§9.3)
- `ft: FileTransfer` — byte-mover (Phase 5; `NoopFileTransfer` in Phase 4)

### 6.2 Step contract invariants (🆕 CL_R2_8 / CL_R2_7)

- **Callback mailbox.** Every `cb` passed to a step must dispatch on the **per-device DeviceRunner mailbox**, not on the step's own actor (if any). This is what makes the §8 generation-token check effective.
- **`next_step` jump target validation.** If `next_step(state)` returns a `StepKey` not present in the current plan, the runner emits a clear log entry and returns `FAILURE` for the current step. (Mirrors Python `refresh_steps`'s regression guard.)
- **Failure isolation.** A step's exception surfaces as `(FAILURE, NewState, exc)` from the callback. The runner logs and proceeds with FAILURE handling — does not let the exception propagate up the actor.

### 6.3 `ComponentPlan` invariants (unchanged from v2)

- **Refresh discipline (A8):** `refresh_steps` runs **after every step's `_execute_step_action`**. Enables IOS-XR FPDs to be discovered mid-run.
- **Monotonicity (A8):** the refresh may add steps, must not remove prior components or steps. A removal indicates a logic bug; the runner raises.
- **Flush ordering (CL8):**
  1. Step's `pre_check` or `execute` returns `(StepResult, NewState)`.
  2. If `result != FAILURE`: persist NewState into the per-component State store (dynstate write).
  3. If `result == FAILURE`: discard NewState; mark step `failed`; mark all subsequent steps in the component back to `not-reached` (or `waiting-confirmation` if confirm-mode).
  4. Refresh the plan.
  5. Persist the plan (dynstate write).

A FAILURE result must NOT persist the half-mutated NewState.

### 6.4 Per-OS step lists

- **SROS** (`platform_sros.act`): step list per logic doc §6.1. **No `Cleanup` step** (CL18).
- **IOS-XR** (`platform_iosxr.act`, Phase 6): step list per logic doc §6.2. **`Cleanup` is IOS-XR only.** `CheckVersions` requires controller-side archive parsing (`.iso` via `get_version_from_iso`, `.tar` via `get_file_packages`); Phase 6 dependency on iso/tar parsers in Acton.
- **Junos** (`platform_junos.act`, Phase 6): per logic doc §6.3. **`StepKey(re_id=None)` for the trailing unparameterized `Done`** (CL12). **Cross-run invariant (CL10):** if `state.dual_re` changes between runs of the same request, `ValidatePlatform` returns FAILURE and invalidates the request.
- **VRP**: enum kept; `ValidatePlatform` fails cleanly with "unsupported platform".

### 6.5 Status mapping at run end (unchanged from v2 — A4)

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

Counters are **consecutive**; FAILURE resets `transient`, WAIT resets `other`.

### 6.6 Run-log filter and bounded ring (🆕 A10/CR9/CR10)

Only log records bearing the `swi_component` attribute are persisted (matches the Python `OperCdbLoggingHandler._log_filter`). The runner installs a per-step logging context that adds `swi_request_path / swi_component / swi_step / swi_run_id` to records emitted within the package's logger. Records flowing past from other modules are dropped at the persistence boundary.

Bounded ring buffer:
- Default cap: **1000 entries per request**. Configurable in v2.0+.
- When full, drops oldest; increments `run_log_dropped`.
- 🆕 Run-log key is `(when, seq)` where `when` is `yang:date-and-time` and `seq` is a per-request monotonic `u64`. `seq` resets to 0 on `clear-run-log-generation` increment.
- Surface `run_log_dropped` in oper so operators investigating a failure know they may have missed entries.

Explicit `clear-run-log-generation` increment empties the ring (§4.5).

### 6.7 Retry budget (unchanged from v2 — A4)

Per-class budget:
- `error_count.transient > config.max_retries` after a WAIT → terminate as `FAILED_TRANSIENT`.
- `error_count.other > config.max_retries` after a FAILURE → terminate as `FAILED_OTHER`.

🆕 **Backoff formula match** (CR2): `backoff = (error_count.backoff or 10) * factor` (factor mode) — exactly as the Python spec, not `factor * max(...)`.

---

## 7. The Transform substrate — Option B (per-device transform + global subscription)

🆕 Decision per round-2 (A3): wire **one transform per device entry** in the host layer, not a single top-level multi-root observer. Matches sorespo's grain (`t_2.act:18` shows `ttt.List(ttt.RFSTransform(BBInterface, ...), [<key>])` — one transform per list entry, each with its own `update_dynstate` / `update_oper`).

### 7.1 Wiring topology

```
Host layer:
    /devices/device[name]/
        +-- ... config from RFS layer above ...
        +-- /software-pack/                       ◀── per-device transform attaches here
                ttt.Container({
                    q("name"): yang.gdata.Leaf,
                    q("scp-port"): yang.gdata.Leaf,
                    q("control"): ttt.Container({...}),
                    // request[] and oper projections published via update_oper
                })
```

Each per-device transform's `act`-spawned actor is the `DeviceRunner` for that device. No `SwInstallRunner` singleton.

### 7.2 Global config subscription

Per-device transforms also need to read `/software-install/...` (pack library, `enabled`, `error-handling`, etc.). This is **not** in their input `cfg`; they obtain it via the existing platform mechanism:

```acton
proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
    runner = DeviceRunner(
        params.path, params.update_oper, params.update_dynstate,
        dev_registry.get(devname_from_path(params.path)),
        ...
    )
    if params.lower is not None:
        # Subscribe to /software-install/... in the lower layer
        params.lower.declare_subscriptions(
            owner_id="sw_install:" + devname_from_path(params.path),
            cb=runner.on_global_config,
            want={SubscriptionSpec(filt=SOFTWARE_INSTALL_FILTER, period=...)},
        )
    return lambda cfg, mem: runner.on_local_config(cfg, mem)
```

`Layer.declare_subscriptions` already exists (`ttt.act:735`) and is exactly designed for this. No SwInstallRunner coordinator needed.

### 7.3 Transform body

The transform itself is essentially passive — sw-install does not produce downward config:

```acton
class SwInstallTransform(ttt.TransformFunction):
    def transform_wrapper(self, cfg, linked, memory, dynstate):
        # No downward output. Memory unchanged (we don't use it — A1).
        return (gdata.Container(), memory)
```

All work happens in the `DeviceRunner` actor invoked through the `act` callback's `on_local_config` and `on_global_config` paths.

### 7.4 Why not the §7-fallback "top-level actor + TreeProvider"?

The fallback is parked. Option B is achievable using only existing platform mechanisms (`Transform`, `act`, `update_oper`, `update_dynstate`, `Layer.declare_subscriptions`). If implementation surfaces a substrate-level incompatibility (e.g., subscription delivery semantics conflict with the per-device transform actor lifecycle), we revisit; otherwise, no need.

⚠️ASSUMPTION: `Layer.declare_subscriptions` callback is delivered to the per-device transform's `act`-spawned actor without re-entering through `transform_wrapper`. Verify in Phase 4 implementation.

---

## 8. DeviceRunner architecture (per-device, per-transform)

### 8.1 Lease scope — honest downgrade (🆕 A2)

The Python `MaapiLocker.lock_partial(/devices/device[name=X])` was a **system-wide** mutex over the device subtree: every other writer blocked while the install was in flight. The Acton `DeviceRunner` does **not** provide an equivalent guarantee. While a sw-install run is active:

- An RFS transform may push config via `DeviceMgr.configure(...)`.
- A monitoring transform may issue `rpc_xml` against the same adapter.
- Subscriptions continue to deliver oper updates.

**This is a real safety gap for OS upgrades.** The Acton `DeviceMgr` does not currently expose an exclusive-operation API; adding `DeviceMgr.acquire_exclusive(owner_id, timeout)` is a platform-side change outside sw-install's scope.

**v1 contract:** sw-install serializes its **own** runs per device. Operators must ensure no RFS layer is actively reconciling against a device under upgrade — typically by gating upstream config or by understanding that the install will likely race with normal reconciliation.

This is documented prominently in:
- The README ("Important safety note")
- §15.5 conscious deviations
- The runtime log at request start ("warning: sw-install does not preempt other DeviceMgr writers")

**Platform prerequisite for v2.0:** see §14.

### 8.2 The DeviceRunner actor

```acton
actor DeviceRunner(
    path: ttt.Path,
    update_oper: action(?gdata.Node) -> None,
    update_dynstate: action(?gdata.Node) -> None,
    dev: swdev.DeviceMgr,
    local_fi: LocalFileInspector,
    remote_fi_factory: proc(swdev.DeviceMgr) -> RemoteFileInspector,
    file_transfer: ?FileTransfer,
    ops_factory: proc(swdev.DeviceMgr, ?CliSession) -> DeviceOps,
    cli_session_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> CliSession,
    log_handler: ?logging.Handler,
):
    var dynstate: SwInstallDynstate = ...      # restored from the platform on startup
    var global_config_cache: ?GlobalSwInstallConfig = None  # latest from subscription

    action def on_local_config(cfg: ?gdata.Node, mem: ?gdata.Node):
        # Pure idempotent reconciliation — A4 invariant 1.
        # Computes: should there be an active request? what triggers fired?
        # Mutates dynstate accordingly.
        ...

    action def on_global_config(g: ?gdata.Node, err: ?Exception):
        # Update global_config_cache; re-run reconciliation against current cfg.
        ...

    action def _step_callback_guard(token: u64, then: action() -> None):
        # CL7: stale-callback no-op via generation_token.
        if dynstate.current is not None and token == dynstate.current.generation_token:
            then()
        # else: silently drop.

    action def _persist_dynstate():
        # Coalesced flush — A4 invariant 3. Called at state-class transitions, not on every poll.
        update_dynstate(dynstate.to_gdata())

    action def _publish_oper():
        update_oper(self._build_oper_view())
```

### 8.3 Three re-entrancy invariants (🆕 A4)

1. **`on_local_config` is a pure idempotent reconciliation function.** Given `(cfg, dynstate)` it computes the desired action set; it does not start work as a side effect of being called. If a piece of work is already in flight, the function observes that (via dynstate) and does nothing. This means re-entry from `update_dynstate`'s `Session.recompute()` is safe: same input → same observation → no double-action.

2. **Generation observations are durably persisted in dynstate before any work begins for that generation.** The runner reads `cfg.start_generation`; if `> dynstate.last_start_generation`, it **first** sets `dynstate.last_start_generation` and persists, **then** initiates the run. A crash between persist and run leaves the trigger consumed; the next on_local_config sees no trigger and waits for a new one. Operator can re-trigger by incrementing the counter again.

3. **High-frequency state writes coalesce.** Run-log entries, install-op-id polling progress, and similar tick-level updates do **not** call `_persist_dynstate` per event. They mutate in-memory dynstate; a periodic flush actor (or a state-class transition like step-completed) calls `_persist_dynstate`. Keeps LMDB churn O(state-transitions), not O(polls).

### 8.4 Restart story (🆕 CL_R2_9)

On platform startup:
1. Platform restores the transform's `dynstate` from lmdb via `_TransformTransaction.restore(...)` → `swdb.decode_node_bytes(...)`.
2. The `act` callback fires for the per-device transform; runner is constructed with restored dynstate injected.
3. Runner inspects `dynstate.current`:
   - If status was `processing` at crash time → set status to `failed-transient` and bump `error_count.transient += 1`. The scheduler's normal retry loop will re-run the request; the per-step `pre_check` (which is idempotent — the spec requires this) handles "file already copied" / "op_id_* already in flight" cases.
   - If status was `cancelling` at crash time → since no live RPC remains, transition directly to `cancelled`.
   - If status was a terminal state (`done`/`cancelled`/`failed-other`/etc.) → no action.
   - If status was `paused` → remains `paused` (waits for `enabled` to flip back).
   - If status was `waiting-confirmation` → remains so; awaits config-side confirmation.
4. On the first `on_local_config` call after restore, reconciliation runs with restored dynstate as authoritative.

### 8.5 Cancel implementation (post-A5)

```
on cancel-generation increment for current request:
    persist dynstate.last_cancel_generation := new_value
    if dynstate.current.status == processing:
        dynstate.current.status = cancelling
        dynstate.current.generation_token += 1     # invalidates in-flight callbacks
        log "cancellation requested at <step>"
        _publish_oper()
        # In-flight RPC's callback will re-enter via _step_callback_guard, no-op.
        # When it returns from the device, _step_completion sees status==cancelling
        # and transitions to cancelled.
    elif dynstate.current.status in (waiting-confirmation, paused, failed-transient, unprocessed):
        dynstate.current.status = cancelled        # nothing in flight, instant
        _persist_dynstate(); _publish_oper()
```

### 8.6 Backoff (🆕 CL5 + CR2 + CR3)

Per-device, per-request:

```
on FAILURE / WAIT terminal of a run:
    error_count.<class> += 1
    error_count.<other-class> = 0
    if error_count.<class> > config.max_retries:
        status = FAILED_<class>; gave_up = True
        publish & persist; done
    backoff = (error_count.backoff or 10) * factor   # CR2 — exact formula
    error_count.backoff = backoff
    next_wake_at = now() + backoff
    _persist_dynstate()                              # PERSIST BEFORE scheduling
    after backoff: _start_run()
```

`error_count.backoff` and `next_wake_at` are projected into oper (CR3) so operators see the retry schedule.

On restart with `next_wake_at` in the future, schedule a fresh `after max(0, next_wake_at - now): _start_run()`.

### 8.7 Device-not-yet-ready case (🆕 CL_R2_10)

If `dev_registry.get(name)` returns a `DeviceMgr` whose adapter is `NoAdapter` (no DMC, no real adapter) or `DeviceMgr.set_dmc` hasn't been called yet, the runner cannot do useful work. v3 introduces:

- `request-status: waiting-for-device` — the runner pauses with this status until DMC + adapter become available; `on_local_config` re-checks readiness on each call; a one-shot subscription on the device's status (or polling) drives re-evaluation.

Documented in §15.5 conscious deviations as a v3 addition (no Python equivalent — Python ran inside NSO and assumed a usable maagic context).

---

## 9. Transport scope: NETCONF + tiered file abstractions

The Python `software_install_script.py` mixes NETCONF, CLI, and SCP. v3 splits these into clean abstractions and ships only what Phase 4 needs.

### 9.1 Op coverage matrix

| Op | SROS | IOS-XR | Junos |
|----|------|--------|-------|
| version, redundancy, boot time, free space | NETCONF state datastore | NETCONF live-status oper | NETCONF RPCs |
| install/activate/commit | NETCONF actions | NETCONF actions | `request package add` via NETCONF |
| BOF reconfiguration | NETCONF edit-config (`bof-running`) | n/a | n/a |
| Device file inspection (stat/list) | `RemoteFileInspector` over NETCONF (Phase 4) | Phase 6 | Phase 6 |
| Image upload (byte transfer) | `FileTransfer` (Phase 5) | Phase 5 | Phase 5 (also cross-RE `file copy`) |
| `prepare_format_standby` (SROS) | NotImplementedError → SKIP_STEP | n/a | n/a |
| `prepare_save_rollback` (SROS) | NotImplementedError → SKIP_STEP | n/a | n/a |
| Archive parsing (IOS-XR `.iso`/`.tar`) | n/a | Phase 6 — controller-side iso/tar libs | n/a |
| Interactive CLI (prompts, log scraping) | Phase 5 (TextFSMPlus templates — see ADR) | Phase 5 | Phase 5 |

### 9.2 `LocalFileInspector` — controller-side filesystem

```acton
class LocalFileInspector(object):
    proc def is_file(self, path: str, cb: action(bool, ?Exception) -> None) -> None: ...
    proc def get_size(self, path: str, cb: action(?u64, ?Exception) -> None) -> None: ...
    proc def hash(self, path: str, algo: str, cb: action(?str, ?Exception) -> None) -> None: ...
```

Used by `CheckFiles` (every per-OS step list starts with this — verify the controller has the file before doing anything else). Default Phase 4 impl uses Acton stdlib filesystem primitives.

### 9.3 `RemoteFileInspector` — device-side metadata via NETCONF (🆕 A6)

```acton
class RemoteFileInfo(value):
    path: str
    size: ?u64
    checksum: ?str
    checksum_algorithm: ?str
    mtime: ?str

class RemoteFileInspector(object):
    proc def stat(self, path: str, cb: action(?RemoteFileInfo, ?Exception) -> None) -> None: ...
    proc def list(self, dir: str, cb: action(list[RemoteFileInfo], ?Exception) -> None) -> None: ...
```

Per-OS implementations:
- **SROS (`ops_sros.act`, Phase 4):** `oper-file` get / `state state` queries via NETCONF — the same paths the Python `NokiaSrosNetconfStrategy.copy_file` / `is_bof_configured` use.
- **IOS-XR / Junos:** Phase 6, via per-OS NETCONF state queries.

Used by `CopyImage.pre_check`: stat each filename → compare size/checksum → `SKIP_STEP` if all present and match, `FAILURE` if missing (and no FileTransfer available), else `SUCCESS` (proceed to execute / byte transfer).

### 9.4 `FileTransfer` — Phase 5 byte-mover

```acton
class FileTransferCaps(value):
    put: bool
    delete: bool
    device_pull: bool       # device fetches from URL (preferred where supported)

class FileTransfer(object):
    proc def caps(self) -> FileTransferCaps: ...
    proc def put(self, local_path: str, remote_path: str,
                 cb: action(?Exception) -> None) -> None: ...
    proc def delete(self, path: str, cb: action(?Exception) -> None) -> None: ...
```

Phase 4 ships **`NoopFileTransfer`** with `caps()` returning all-false and `put`/`delete` returning `NotImplementedError`. **Crucially**, since `RemoteFileInspector` is a **separate** abstraction (§9.3), Phase 4 SROS *can* still verify pre-staged images via NETCONF and SKIP_STEP `CopyImage` cleanly. The v2 incoherence is gone (A6).

`CopyImage.execute` in Phase 4: returns FAILURE with clear "no FileTransfer configured — pre-stage the image" if files missing per RemoteFileInspector. Phase 5 fills in the real `FileTransfer` implementation.

### 9.5 Credential reuse — `DeviceMgr.get_dmc()` (🆕 CL_R2_1)

The current code reality: `DeviceMetaConfig.credentials` is owned by `DeviceMgr` (`device.act:288 var dmc: DeviceMetaConfig = DeviceMetaConfig(...)`) and injected into adapters via `set_dmc(...)`. DMC is mutable (`set_dmc` is called repeatedly on reconfiguration).

**Resolution:** plumb a one-line getter on the platform side:

```acton
# In stratoweave/stratoweave/src/stratoweave/device.act, on actor DeviceMgr:
def get_dmc() -> DeviceMetaConfig:
    return dmc
```

This **does not increase the credential blast radius** — DMC is owned by `DeviceMgr`, not the adapter, so exposing it via a getter doesn't leak through any abstraction the adapter establishes.

**File transfer factory signature:**

```acton
file_transfer_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer = None
```

**Read DMC at use time, not construction time** — DMC is mutable. The factory builds a `FileTransfer` instance bound to the `DeviceMgr`; the instance calls `dev.get_dmc()` per transfer to get fresh credentials.

The earlier (a)/(c) framing (extending `DeviceAdapter.get_credentials()` / piggybacking on netconf SSH transport) is **dropped**. The piggyback option in particular was structurally infeasible: `netconf/src/ssh_client.act:39–116` wraps an OpenSSH subprocess (one connection ≡ one process); channel multiplexing would require ControlMaster/ControlPath plumbing or an in-process SSH library, neither of which exist.

⚠️ASSUMPTION: the platform team will accept the `DeviceMgr.get_dmc()` addition. **Round-3 question.**

### 9.6 `scp-port` placement (🆕 CL_R2_5)

The original Python YANG had `scp-port` directly on the per-device augment (`/devices/device/scp-port?`). v2 moved it under `software-pack/` — that means removing the pack assignment also removes the scp-port, which is wrong (scp-port is really an SSH/SCP transport setting, orthogonal to sw-install).

**v3:** `scp-port` lives at `/devices/device/scp-port?` (direct device augmentation). Survives unbinding the pack. Future non-sw-install consumers (config backup, device support tooling) can share the leaf.

### 9.7 `DeviceOps` facade — CLI strategy boundary

The Python original mixes NETCONF and CLI per-OS — `NokiaSrosCliStrategy` vs `NokiaSrosNetconfStrategy` both implement `NokiaSrosOperationsProto`, with `NokiaSrosOperations` delegating based on device capabilities. v3 preserves this strategy pattern.

```acton
# ops.act
class DeviceOps(object):
    """Per-OS operations facade. Hides NETCONF/CLI strategy choice from steps."""
    proc def get_version(self, cb: action(?str, ?Exception) -> None) -> None: ...
    proc def get_boot_time(self, cb: action(?datetime, ?Exception) -> None) -> None: ...
    proc def get_redundancy_info(self, cb: action(?(str, str), ?Exception) -> None) -> None: ...
    proc def reboot(self, cb: action(?bool, ?Exception) -> None) -> None: ...
    # ... per-OS device interaction primitives
```

`SrosOps` (et al) implements the facade; internally selects NETCONF or CLI strategy at construction based on `dev.get_capabilities()` / `dev.get_modules()` snapshot. **Capability snapshot is per-run** (not per-step) — fail clean on incompatible drift between runs (CR6).

Phase 4: `SrosOps` is constructed with `cli_session = None`. NETCONF strategy methods are real; CLI strategy methods exist as stubs that raise `NotImplementedError`. SROS Phase 4 only invokes NETCONF paths.

**TextFSMPlus implementation details (templates, Send/Preset/Done semantics, aycalc-equivalent expression eval, prompt synchronization, terminal width, command echo, secrets handling, the acton-utils textfsm extension dependency) live in `docs/adr/cli-driver.md`.** That ADR is a Phase 5 design artifact; this design doc commits only to the `DeviceOps` boundary in §9.7.

---

## 10. Testing strategy

Following stratoweave's existing test conventions:

- `test_pack.act` — value tests; equality/hashing/from_gdata round-trips.
- `test_plan.act` — **plan refresh monotonicity, after-every-step trigger, next-step jump targets (incl. "target not in plan" → FAILURE), flush-only-if-not-FAILURE invariant, status mapping (consecutive counters), retry exhaustion per-class**.
- `test_state.act` — per-OS `reset()` semantics; Junos `dual_re` cross-run invariant.
- `test_runlog.act` — bounded retention, dropped-count, `(when, seq)` keying, `swi_component` filter, `clear-run-log-generation` semantics.
- `test_dynstate.act` — restore-after-crash scenarios: `processing → failed-transient`, `cancelling → cancelled`, generation token consumption, backoff resume.
- `test_step_runner.act` — actor integration with `MockAdapter`; scripted SROS install end-to-end; **stale-callback no-op via generation token**, `on_local_config` re-entrancy idempotency (call twice, observe single action).
- `test_sros.act` — per-step tests against a mock NETCONF server.
- `test_remote_file_sros.act` — `RemoteFileInspector` SROS impl tests.
- `test_local_file.act` — `LocalFileInspector` tests with temp files.

---

## 11. Implementation phasing (within Phase 4)

Once round-3 review converges:

1. **Skeleton + revised YANG** (the v3 yang.act covering control subtree, per-device augment, `scp-port` direct device augment, generation+target leaves, `paused`/`cancelling`/`waiting-for-device` enums, `(when, seq)` run-log key).
2. **Pure types** (`pack.act`, `state.act`, `plan.act`, `step.act`, `local_file.act`, `remote_file.act`, `file_transfer.act`, `ops.act`) — value types with full unit tests.
3. **Transform + DeviceRunner skeleton** (`transform.act`, `device_runner.act`) — wires up Option B subscription, on_local_config + on_global_config + reconciliation, no device interaction yet. Tests verify generation-counter consumption and `on_local_config` re-entrancy.
4. **First step: `CheckFiles`** — uses `LocalFileInspector` only, no device interaction.
5. **First device step: `CheckVersions` for SROS** — uses real `SrosOps` NETCONF strategy. Mock test against `MockAdapter`.
6. **All SROS steps** in spec order, each with mock tests.
7. **Mock-driven full-flow integration test:** scripted SROS upgrade end-to-end against a `MockAdapter` that scripts NETCONF + RemoteFileInspector responses.
8. **Restart test:** kill mid-step, verify re-spawn from dynstate continues correctly.

Phase 4 done when: SROS Phase 4 test_step_runner end-to-end test passes against a MockAdapter, including a forced cancellation and a forced restart-mid-step scenario.

---

## 12. Open decisions (round-3 questions)

| # | Question | My current lean |
|---|----------|-----------------|
| **❓Q1** | Is `Layer.declare_subscriptions` (called from a per-device transform's `act`-spawned actor) safe to use the way §7.2 sketches? Need platform-owner sanity check. | Yes (it's an existing platform mechanism; sorespo doesn't use it this way but the API supports it). Verify in implementation. |
| **❓Q2** | Will the platform team accept `DeviceMgr.get_dmc()` as a one-line addition (§9.5)? | Yes — DMC is already DeviceMgr-owned, exposing a getter doesn't leak through any abstraction. |
| **❓Q3** | Is honest device-lease-downgrade acceptable, or is `DeviceMgr.acquire_exclusive(...)` a Phase 4 prerequisite? | Downgrade for v1; lease API is the v2.0 platform task (§14). |
| **❓Q4** | Are diagnostic-projection oper leaves (§3.4) the right operability story for the dropped `internal-state`? | Yes — typed leaves > opaque JSON blob. |
| **❓Q5** | Run-log default bound (1000)? Cap retained as configurable in v2.0+. | Keep 1000 default. |
| **❓Q6** | Phase split: Phase 5 = TextFSMPlus + DeviceOps CLI strategy + FileTransfer byte-mover; Phase 6 = IOS-XR + Junos + polish. Aligned with user direction. | Yes. |

---

## 13. Implementation details deferred to first-cut coding

These resolve naturally:

- Exact lmdb key layout for runner dynstate (follows `_TransformTransactionBase.db_ops` patterns).
- `update_oper` snapshot frequency (every state-class transition; coalesce per-tick polls).
- Acton stdlib logging-handler glue for `swi_component` attribute filtering.
- Acton-stdlib filesystem primitives used by `LocalFileInspector` (likely `file.FileCap` + `file.ReadFile` / stat helpers).

---

## 14. Platform prerequisites for v2.0 (🆕)

These are sw-install-adjacent platform changes that v1 is willing to live without, documented here so they don't get lost.

1. **`DeviceMgr.acquire_exclusive(owner_id, timeout, cb)`** + matching `release_exclusive` — gates `configure`, `rpc_xml`, `fetch_config`, `declare_subscriptions` paths under exclusive ownership. Closes the §8.1 lease gap.
2. **Config-restore event hook on `Layer`** — fires when config is restored from backup, lets sw-install reset all `last_observed_*` counters automatically (§3.5).
3. **`DeviceMgr.get_dmc()`** — already required for v1 §9.5, but listed here in case the platform team decides the path is `DeviceAdapter.get_credentials()` instead.

---

## 15. Deferred features

Things deliberately **not** in this design, captured here:

- **`software-install-matrix`** (Python YANG): unused by Python core logic. Dropped from YANG; not modelled in Acton; re-add only if a real use case surfaces.
- **CLI / textfsm parsing**: Phase 5 dependency on the acton-utils textfsm extension (Send/Preset/Done line actions + aycalc-equivalent expression eval). Reference impl `/Users/ayourtch/rust/ayclic/aytextfsmplus`. See `docs/adr/cli-driver.md`.
- **IOS-XR archive parsing** (`get_version_from_iso` / `get_file_packages`): controller-side iso/tar reading. Phase 6 dependency.
- **VRP step module**: enum value preserved, `ValidatePlatform` fails clean. Implementation deferred indefinitely.
- **Snabb / ONS-TL1 / HGW**: dropped from the Acton port (logic doc §6.4). Dropped, not deferred.
- **Approval-required / multi-stage commit hooks**: out of scope for sw-install — that's the platform's job.
- **`DeviceMgr.acquire_exclusive(...)` lease API**: v2.0 platform prerequisite (§14).

## 15.5 Conscious deviations from the Python spec (🆕 §D from claude-r2)

A consolidated list of intentional fidelity-vs-operability tradeoffs the v3 design has made. A reader going from `01-software-install-logic.md` straight into this design might miss them otherwise.

1. **`internal-state` is no longer NETCONF/RESTCONF-visible.** v3 §3.3. Replaced by typed diagnostic projections (§3.4) — better for typed inspection, worse for "show me everything via one CLI command".
2. **Run-log is bounded at 1000 entries/request** with `dropped-count` surfaced. Python was unbounded.
3. **Cancel takes effect at "next step boundary or RPC return," with explicit `cancelling` enum.** v3 §4.4. Python `cancel-request` SIGINT'd the worker; this is more honest about timing.
4. **Per-device install lease is sw-install-internal only.** v3 §8.1. Python `MaapiLocker` was system-wide. Operators must avoid concurrent RFS reconciliation against a device under upgrade. v2.0 platform task.
5. **Generation counters can go backward on backup-restore unless dynstate is restored alongside config.** v3 §3.5. Python had no equivalent generation concept; restore semantics didn't apply.
6. **`software-install-matrix` dropped from YANG.** v3 §15. Python had it but unused.
7. **`vrp` enum kept; no `vrp` step module.** v3 §15. ValidatePlatform fails clean.
8. **Snabb/ONS-TL1/HGW dropped.** v3 §15.
9. **Action-style return values replaced by per-device `last-create-result` (single-slot, last writer wins).** v3 §4.1. CR8 — concurrent automation should use per-call correlation if needed (future enhancement).
10. **CLI strategy methods exist as stubs in Phase 4.** v3 §9.7. Real impl in Phase 5.
11. **`waiting-for-device` request status (no Python equivalent)** — handles DMC-not-yet-set / NoAdapter cases (§8.7). Python ran inside NSO, assumed maagic context always available.
12. **`paused` request status (no Python equivalent)** — explicit "operator disabled the system mid-flight" state (§4.7).
13. **Run-log key change: `(when, seq)` not just `when`.** v3 §6.6. Avoids microsecond-collision ties under multiple log emitters.
14. **`scp-port` placement: `/devices/device/scp-port?` (direct augment), not nested in `software-pack/`.** v3 §9.6. Survives pack unbinding.

---

## 16. Round-3 review

This v3 design integrates all round-2 review feedback. Both reviewers' six convergent HIGH-priority items (memory/dynstate confusion, device lease scope, wiring topology, update_dynstate re-entrancy, cancel/enabled semantics, NoopFileTransfer/CopyImage incoherence) are addressed. All MEDIUM and LOW items are folded in unless explicitly listed as deferred (§15) or platform-side (§14).

**Stop here for round-3 review.** Both reviewers will be re-briefed against the full revised doc set (`00-orientation.md`, `01-software-install-logic.md`, `02-sw-install-design.md` v3, `docs/adr/cli-driver.md`) with no carry-over context.
