# 01 — `software-install` Logic Specification

This document is a **language-agnostic specification** of the existing NSO/Python `software-install` package. Two engineers reading this should be able to implement compatible reimplementations without referring back to the Python code.

Source of truth: `activities/software-install/packages/sw-install/`. Everywhere this doc says "the spec" or "the package", it means that.

The doc is organized so you can read **§2 (data model) and §3 (state machine)** first and stop there if you only need a high-level understanding. §4–§6 add the per-step semantics, error handling, and per-OS specifics needed for full equivalence.

---

## 1. Vocabulary (recap from `00-orientation.md`)

| Term | Definition |
|------|------------|
| **software-pack** | Named profile = `{ name, os, base?, [patch...] }`. Lives in global config. |
| **component** | An entry of a software-pack — either the `base` image or a single `patch`. Each has `version` and one or more `filenames`. |
| **request** | A snapshot of a software-pack bound to a device. Created via the `create-request` action. Identified by `(device, request-id)`; `request-id` is monotonically increasing per device, key starts at `1`. |
| **plan** | The list `OrderedDict[component, OrderedDict[step, StepStatus]]` derived for a request. Components are 1:1 with the pack's components. Step list comes from the per-OS module's `get_steps(state)`. |
| **run** | One execution attempt of a request. Each run gets a fresh `run-id`, monotonically increasing per request. |
| **step** | One executable unit within a component plan. Per-OS class with `execute()`, optionally `pre_check()`, optionally `next_step()`. |

---

## 2. Data Model (YANG → conceptual)

The full model is in `src/yang/software-install.yang`. This section names every datum that participates in the logic.

### 2.1 Global config — `/software-install/`

| Path | Type | Default | Used by |
|------|------|---------|---------|
| `enabled` | bool | `false` | Worker activation |
| `confirm-steps` | bool | `true` | Plan default; can be overridden per-request |
| `auto-execute-after-confirm` | bool | `false` | `confirm-step` action: when true, calls `execute-request` after confirming |
| `error-handling/max-retries` | uint32 | `30` | Scheduler: gives up after this many retries |
| `error-handling/backoff/factor` | decimal64 (2dp) | `1.2` | Exponential backoff multiplier (choice case) |
| `error-handling/backoff/constant` | uint32 | `30` | Constant backoff in seconds (choice case, default case) |
| `software-pack[name]` | list | — | Library of pack profiles (see §2.2) |
| `software-install-matrix[running-pack]` | list | — | Defines `running-pack → target-pack` mapping (currently unused by core logic) |

### 2.2 `software-pack` (and `software-pack-data` snapshot)

Both the global pack definition and the per-request `software-pack-data` use the `software-pack-grouping`:

```
software-pack {
    name             # string
    os               # enum: ios-xr | vrp | junos | sros
    base?            # presence-container with software-file (version + filenames)
    [patch]          # ordered list, key=version (only when os = ios-xr or vrp)
}

software-file {
    version          # string, mandatory; must match what the device reports after install
    [filename]       # leaf-list, max 31, ordered-by user; absolute path on the NSO host
}
```

Two `software-pack` values compare by **deep equality** of `(name, os, base, patch)`. This drives the idempotency of `create-request` (see §3.1).

### 2.3 Per-device augmentation — `/devices/device/software-pack`

```
software-pack {
    presence "Software-pack present"   # absence = remove all sw-install state for device
    name?                               # leafref → /software-install/software-pack/name (the desired pack)
    [request]                           # config false; populated by the system
    create-request action               # output: request-id, status (new-request | existing-request)
}
```

Plus, on each device: `scp-port?` (port for the host-side SCP server fallback used in some Cisco flows).

### 2.4 `request` (operational data, populated by the system)

```
request {
    id                                 # uint32, key
    status                             # request-status enum (see §3)
    confirm-steps?                     # bool — overrides global if set
    software-pack-data {               # IMMUTABLE snapshot of the pack at request creation
        ... software-pack-grouping ...
    }
    error-count {
        transient                      # consecutive transient (WAIT/timeout/etc.) errors
        other                          # consecutive other (FAILURE) errors
        backoff (units seconds)        # current backoff, mutated by the scheduler
    }
    obsolete                           # bool, default false; set true when superseded
    [component] (ordered-by user) {
        name                           # base-<version> | patch-<version>
        [step] (ordered-by user) {
            name                       # step class name (or class[REn] for Junos)
            status                     # step-status enum (see §3)
            error?                     # error string for failed step
            when?                      # yang:date-and-time when status last changed (skipping "not-reached")
            confirmed? {               # presence container
                by-user                # mandatory string
                when?
            }
        }
        internal-state?                # JSON-encoded State object (jsonpickle); see §5
        completed                      # bool — true when last step's status == 'reached'
    }
    run-id-count                       # uint64; incremented at the start of every run
    [run-log] {                        # key=when (yang:date-and-time), so unique per us
        when, run-id, component, step, message
    }
    confirm-step action                # input: choice {all | [component {name, [step]}]}
    execute-request action             # output: status string
    cancel-request action              # output: status string
    clear-run-log action               # deletes ALL run-log entries; output: success bool
}
```

### 2.5 Enumerations

```
request-status:
    unprocessed | waiting-confirmation | processing | done
    | cancelled | obsolete | failed-other | failed-transient

step-status:
    not-reached | waiting-confirmation | processing | reached
    | failed | skipped | wait
```

Note: `request-status` and `step-status` are **distinct** enumerations even though they share several spellings.

### 2.6 OS detection and `DeviceOs` enum

`device_os.py` derives an internal `DeviceOs` enum from the device's NETCONF capabilities:

| Capability URI | DeviceOs |
|----------------|----------|
| `http://tail-f.com/ned/alu-sr` | `SROS_CLI` |
| `urn:nokia.com:sros:ns:yang:sr:conf` | `SROS_NC` |
| `http://cisco.com/ns/yang/cisco-xr-types` | `IOSXR` |
| `urn:juniper-rpc` | `JUNOS` |
| `http://tail-f.com/ned/huawei-vrp` | `VRP` |
| `snabb:softwire-v2` | `SNABB_v2` |
| `snabb:lwaftr` (only if no other set) | `SNABB_v1` |
| `http://terastream.net/ned/ons-tl1` | `ONS_TL1` |
| `http://terastrm.net/ns/yang/terastream-software` or `…-hack` | `HGW` |

Used only by per-OS `validate_platform()` to refuse to operate on the wrong device type. This is **separate** from the YANG `os` enum on the pack (`ios-xr | vrp | junos | sros`); the pack `os` selects which Python module's `get_steps()` to call.

---

## 3. Request Lifecycle (the master state machine)

### 3.1 `create-request` — idempotency rules

Pseudocode for `actions.py:CreateRequestAction.cb_action`:

```
target_sp := SoftwarePack.from_cdb(/software-install/software-pack[<dev's pack name>])
last_request := the request with the highest id on this device, if any

if no requests:
    update := True
    last_request_cancelled := False
else:
    request_sp := SoftwarePack.from_cdb(last_request.software_pack_data)
    update := target_sp != request_sp           # deep equality
    last_request_cancelled := last_request.status == 'cancelled'

if update or last_request_cancelled:
    for r in all existing requests:
        r.obsolete := True
    new_request := create request with id = (last_id + 1) or 1 if none
    new_request.software_pack_data := target_sp
    return (request_id = new_request.id, status = 'new-request')
else:
    return (request_id = last_request.id, status = 'existing-request')
```

**Key invariants:**
- `software-pack-data` is set once at creation and is never mutated.
- Re-running `create-request` with no change to the pack returns the same id.
- A *cancelled* last request always forces a new request, even if the pack is unchanged (this is how a user "reactivates" a cancelled install).
- All previous requests are marked `obsolete=true` whenever a new one is created.

### 3.2 `request.status` state diagram

```
                               obsolete=true forced by create-request
                          ┌────────────────────────────────────────────┐
                          │                                            ▼
   [create-request]     unprocessed ─execute-request─▶ processing   obsolete (terminal*)
                                                          │
                                                          │ step loop (§3.3)
                                                          │
                  ┌───────────────────┬───────────────────┼────────────────────┐
                  ▼                   ▼                   ▼                    ▼
            waiting-confirmation   failed-other     failed-transient          done
            (next step needs       (FAILURE,        (WAIT, retry             (terminal)
             confirmation)          retry per         per scheduler)
                  │                  scheduler)
       confirm-step action            │                    │
       + auto-execute-after-          │ scheduler retries  │ scheduler retries
       confirm? → execute             ▼                    ▼
                  │              processing            processing
                  ▼
            (back to processing
             when execute-request fires)

           cancelled (terminal* — actually overridable: a fresh create-request after
                       cancel always creates a NEW request)
```

`*terminal` = the **request** is not driven further by the worker, but a **new request** for the same device may begin from `unprocessed`.

The exact decisions live in `step_executor.py:_write_request_status`:

```
if obsolete:                        status = obsolete
elif all components.completed:      status = done; clear error_count
elif needs_confirmation:            status = waiting-confirmation
elif last result == WAIT:           status = failed-transient; ++error_count.transient; error_count.other = 0
else:                               status = failed-other;     ++error_count.other;     error_count.transient = 0
```

Note: a **WAIT** result terminates the run as `failed-transient` so the scheduler can retry — it is *not* an in-step retry.

### 3.3 Run loop (`step_executor.step_executor`)

For each call (one run):

1. Atomically increment `request.run_id_count`; remember new value as `run_id`.
2. Read the request. If `obsolete`, write status=`obsolete` and return.
3. Build (or refresh) the **plan** from `software-pack-data` and the per-OS `get_steps()` (see §4).
4. Write status=`processing`. Persist plan.
5. For each `component` in the plan, in order:
   - Reload `state` from `request/component[name]/internal-state` (or build a fresh one — see §5).
   - Call `_execute_component(plan, component, state)` (§3.4).
   - If component returns `(needs_confirmation=True, _)` or `result in {WAIT, FAILURE}`: stop iterating components.
6. Compute the new request status (rules above) and persist it.

### 3.4 Component step loop (`_execute_component`)

Iterates `plan.next_step(component)` which yields `(step_class, needs_confirmation)` pairs **starting from the current cursor** (i.e., the first step that is not yet `reached`/`skipped`/`failed`/`processing`/`wait`). Behavior per-yield:

```
if previous result was SKIP_COMPONENT, OR (we set next_step earlier and this isn't it):
    plan.skip(component, step); continue

if needs_confirmation:
    return (True, None)                      # bubble up "waiting-confirmation"

executor := step_class(context)

# Optional pre_check (only if step implements StepExecutorSupportsPreCheck)
if isinstance(executor, StepExecutorSupportsPreCheck):
    result := _execute_step_action(..., executor.pre_check, state)
    if result in {WAIT, FAILURE}: return (False, result)
    if result == SKIP_COMPONENT: continue
    # else: fall through. SKIP_STEP from pre_check means don't run execute().
    # SUCCESS from pre_check means continue to execute().

# execute() — only if pre_check didn't skip
if result is None or result == SUCCESS:
    result := _execute_step_action(..., executor.execute, state)
    if result in {WAIT, FAILURE}: return (False, result)
    if result == SKIP_COMPONENT: continue

# Always run next_step() after execute (even if step skipped).
# Failures in next_step() degrade the step to FAILURE.
try:
    next_step := executor.next_step(state)
except Exception as e:
    plan.fail(...); return (False, FAILURE)
# next_step is a step class to JUMP TO; intermediate steps are skipped.
```

Returns `(needs_confirmation: bool, last_result: StepResult?)`.

### 3.5 `_execute_step_action` — per-step bookkeeping

Wraps each invocation (`pre_check` or `execute`):

1. Set `context.step = step_class.name()`.
2. Mark step `processing` and persist the plan.
3. Invoke the executor method.
4. On exception → `result = FAILURE`, `plan.fail(component, step, error=...)`, log error + traceback.
5. On result → call `plan.complete / fail / wait / skip` per the table below.
6. If result ≠ FAILURE: **`state.flush(t_write)`** (persist `internal-state` JSON).
7. **Refresh the plan from the read-state** (`plan.refresh_steps`) — picks up any newly-discovered steps (e.g., XR FPDs after install).
8. Persist plan again.

| StepResult | plan state op |
|------------|---------------|
| `SUCCESS` | `complete` → status `reached` |
| `FAILURE` | `fail` → status `failed`, error field set, **all subsequent steps reset to default-unprocessed** |
| `SKIP_STEP` / `SKIP_COMPONENT` | `skip` → status `skipped` |
| `WAIT` | `wait` → status `wait` |

Always: `when` is set when status changes (and is not `not-reached`).

### 3.6 Plan refresh on the fly (`ComponentPlan.refresh_steps`)

Plans are **monotonic** in steps: a refresh may **add** new steps (e.g., IOS-XR FPD reload steps that only become known after `SoftwarePrepare`), but **must not remove** them. The refresh:

1. Builds a new ordered plan by walking the current pack's components and calling `_get_component_steps(component, state)` for each.
2. For each new step, attempts to inherit prior status: `(component, step) → StepStatus`. If absent, falls back to reading the on-disk request, else default `(unprocessed, confirm)`.
3. **Raises** if any prior component or step is missing from the refreshed plan (regression guard).

The persisted plan is reordered in CDB to match the authoritative order (`step.move`, `component.move`).

### 3.7 `confirm-step` action

Input is a `choice`:
- `case all` — `leaf all empty` — confirms every step in every component.
- `case component` — list of `{name, [step]}` — confirms specific steps within named components.

Effect:
- For each (component, step), create the `confirmed` presence container with `by-user = uinfo.username`, `when = now()`.
- After applying: if the global `auto-execute-after-confirm` is true, dispatch the request's `execute-request` action.

When a step's plan loop reads `step.status == 'waiting-confirmation'`, it sets `needs_confirmation = not confirmed.exists()`. The component loop in §3.4 returns early when it encounters a step with `needs_confirmation=True`.

### 3.8 `cancel-request` action

`RequestWorker.cancel_request(path)` enqueues `(path, _, RequestStatus.CANCELLED)` on the scheduler queue. The scheduler:
1. Looks up the worker PID for that path.
2. Sends `SIGINT` to the worker process (kills the in-flight run).
3. Removes from the workers map.
4. Sets `request.status = cancelled`.

Subsequently, if the user calls `create-request` again, the cancelled-status check in §3.1 forces a fresh request even if the pack hasn't changed.

### 3.9 `clear-run-log` action

Deletes the entire `run-log` list for the request. Returns `success=true`.

---

## 4. Plan structure

### 4.1 Components

`SoftwarePack.components()` yields, in order:

1. The `base` component (if present).
2. Each `patch` (only when `os ∈ {ios-xr, vrp}`; SROS and Junos do not have patches in this model).

Each component has:
- `version` — the version string the device should report after install
- `filenames` — tuple of file paths (immutable — `Iterable[str]` cast to `tuple` in `from_cdb`)
- `os` — `SoftwarePackOs` enum
- `name` — derived: `base-<version>` or `patch-<version>` (used as the YANG list key)

`SoftwarePackComponent` has subclasses `SoftwarePackComponentBase` and `SoftwarePackComponentPatch`. **Equality and hash are by deep value** (frozen dataclass); this is what makes them dict-keyable.

### 4.2 Steps — the contract

A step is a class implementing the `StepExecutor` ABC plus optional protocols:

```
class StepExecutor:                                       # mandatory
    name() -> str                                         # default: __qualname__ (Junos parametrizes this)
    __init__(context: Context)                            # optional (Junos overrides for re_id)
    execute(state) -> StepResult                          # mandatory
    next_step(state) -> Optional[Type[StepExecutor]]      # optional override; default returns None
    _add_oper_log(message)                                # convenience; logs at ERROR level

class StepExecutorSupportsPreCheck(Protocol):             # optional
    pre_check(state) -> StepResult                        # invoked BEFORE execute
```

`StepResult` enum:

| Value | Meaning |
|-------|---------|
| `SUCCESS` | Step completed; advance to next |
| `FAILURE` | Step failed; abort the component, abort the run, status = `failed-other` |
| `SKIP_STEP` | Step not needed; mark `skipped`, advance |
| `SKIP_COMPONENT` | Skip this and all remaining steps in this component |
| `WAIT` | Step partially done; device needs time. Run ends as `failed-transient`; scheduler retries |

Notes:
- A step's `next_step(state)` returning `X` causes the inner loop to **skip** all steps strictly between current and `X`. They are marked `skipped` (this is what enables jump-targets like "go to `Done` if already at target version" and "go to `Cleanup` if patches already committed").
- `next_step` runs even after `SKIP_STEP` results, but **not** after `SUCCESS` of `execute()` if `next_step()` itself raises (that becomes a `FAILURE`).

### 4.3 `name()` and Junos's parametrized steps

For Junos dual-RE devices, the same step class is instantiated for each RE. To preserve dict-keyability (steps are used as `OrderedDict` keys), `_partial_RE(cls, re_id)`:
- Generates a subclass with `__init__` partial-applied to `re_id`.
- Overrides `name()` to return `f"{cls.__qualname__}[RE{re_id}]"`.
- **Caches** the generated subclass per `(cls, re_id)` so two parametrized types with the same key are *the same Python object* (and therefore `hash`-equal).

The Acton port should pick a representation that yields equivalent dict-keying behavior — e.g., use a `(class_name: str, re_id: ?str)` tuple as the step key in the plan map.

---

## 5. State (`internal-state`)

Per-component `internal-state` is the JSON-pickled `State` object. It is the **only** mutable cross-step state — everything that needs to survive between steps in the same run, or between runs of the same request, lives here.

### 5.1 Common (`State`)

```
State {
    device: str                               # device name
    software_pack_component: SoftwarePackComponent  # snapshot
    _spc_path: str                            # internal — keypath for flush()
    virtual_router: bool = False              # affects step behavior on virtual platforms
    done: bool = False                        # set by Done step
}

State.flush(oper_t_write)
    Persist self to <_spc_path>/internal-state via jsonpickle.

State.reset()
    Default impl: virtual_router = False (subclasses extend)
    Called when CheckVersions detects that the previously "done" version
    is no longer running on the device — restart logic.
```

`State.from_component(request, component)` — load JSON if present, else construct a fresh one of the OS-appropriate subclass.

### 5.2 `GenericDevice` (mixin for "single-device" state)

```
GenericDevice {
    destination_volume: ?str = None           # filesystem prefix on device, e.g. "disk0:" or "cf3:/TiMOS-..."
    destination_paths: dict[str, str] = {}    # local file → remote path
    boot_time: ?datetime = None               # for reboot detection
    done: bool = False
}
```

### 5.3 OS-specific subclasses

```
StateIosXr(GenericDevice, State) {
    op_id_add: ?int                           # 'install add' op-id
    op_id_prepare: ?int
    op_id_activate: ?int
    op_id_commit: ?int
    packages: [str]                           # extracted from .iso/.tar
    reload_required: bool = False             # determined from activate's op-log
}
    restart_prepare_clean(): clear prepare + activate + commit op_ids
    restart_prepare(): clear activate + commit op_ids
    restart_activate(): clear commit op_id

RouteEngine(GenericDevice) {                  # one per Junos RE
    version: ?str
    rebooted: bool = False
}

StateJunos(State) {                           # NOT a GenericDevice — has many REs
    dual_re: ?bool                            # detected by ValidatePlatform
    switch: bool = False                      # QFX/EX detection
    failover_config: any                      # snapshot for Disable/EnableFailover
    route_engine: dict[str, RouteEngine]      # keyed by str(re_id)
    route_engine_priority: [int]              # master first, then backup
}
    reset(): clear failover_config; reset each RouteEngine

StateSros(GenericDevice, State) {
    version: ?str
    rebooted: bool = False
}
```

The OS-specific state subclass is determined by the component's `os` field at `from_component()` time.

---

## 6. Per-OS Step Lists

Steps are listed in execution order. `(pc)` = has `pre_check`. `→` = `next_step()` jump targets noted inline.

### 6.1 SROS (`sros.py`)

```
get_steps(_) = [
    ValidatePlatform,
    CheckFiles,
    CheckVersions,         # → Done if version matches | Reboot if rebooted | else default
    ActivatePrimary (pc),  # pre_check skips if active='A'; else fails over and waits up to 600s
    GetBootTime,
    CopyImage (pc),        # pre_check skips if all files present + checksum match
    PrepareCopyBootLdr (pc),
    PrepareConfigureBof (pc),  # skips if BOF already configured
    PrepareHackFormatStandby,  # NotImplementedError → SKIP_STEP
    PrepareSaveRollback,       # NotImplementedError → SKIP_STEP
    PrepareSynchronizeBootenv,
    Reboot (pc),               # pre_check: if rebooted+boot_time advanced → SKIP/FAIL based on version; else WAIT/SUCCESS
    Done                       # state.done = True
]
```

Step semantics in detail:

- **`ValidatePlatform.execute`**: checks device is one of `{SROS_NC, SROS_CLI}`; FAIL with explanation otherwise.
- **`CheckFiles.execute`**: each `state.software_pack_component.filenames[i]` must be a regular file on the NSO host. Local FS check.
- **`CheckVersions.execute`**:
  - `o.get_version()` — current running version
  - if `state.done and state.version != version`: device drifted → `state.reset()`
  - `state.version = version`; `state.virtual_router = o.is_virtual_router()`
  - if `state.rebooted and state.boot_time`: refetch boot_time; if newer but version doesn't match component.version → log warning, return `FAILURE`
  - else `SUCCESS`
  - `next_step`: if `version == component.version`: → `Done`; elif `state.rebooted`: → `Reboot`; else → `ActivatePrimary` (next default).
- **`ActivatePrimary.pre_check`**: get `(active_node, redundancy_status)`; expect `redundancy_status == 'standby'` else FAIL; if `active_node == 'A'` SKIP_STEP; else SUCCESS.
- **`ActivatePrimary.execute`**: `o.redundancy_failover()`; sleep 5; poll up to 600s for `(active='A', standby='standby')`; FAIL on timeout.
- **`CopyImage.pre_check`** (long): get free space, set `destination_volume = '<active_cf>:/TiMOS-SR-<version>'` (subdir), check each filename for presence + checksum; if all present → SKIP_STEP; if missing files don't fit in free space → FAIL; else SUCCESS.
- **`CopyImage.execute`**: SCP each missing file; verify; FAIL if any verification fails.
- **`PrepareCopyBootLdr`**: compute source vs. destination boot.ldr hash; SKIP_STEP if equal; else execute the platform's "prepare copy boot.ldr" command.
- **`PrepareConfigureBof`**: if device is in-sync AND BOF already points at our image → SKIP_STEP; else swap primary/secondary in the `bof-running` datastore via NETCONF lock/edit/validate/commit.
- **`PrepareHackFormatStandby`** / **`PrepareSaveRollback`**: both currently raise `NotImplementedError` and resolve to `SKIP_STEP` — placeholders for future device-side actions.
- **`PrepareSynchronizeBootenv.execute`**: invoke `admin/redundancy/synchronize` (boot-environment) RPC; FAIL on error.
- **`Reboot.pre_check`**: if not yet rebooted → SUCCESS (proceed to execute, which initiates reboot). If `rebooted+boot_time set`: refetch boot_time; if newer + version matches → SKIP_STEP; if newer + version mismatch → FAILURE; if not newer → WAIT.
- **`Reboot.execute`**: `o.reboot()`; if accepted → `state.rebooted = True`; return WAIT (run ends `failed-transient`, scheduler retries).
- **`Done.execute`**: `state.done = True`; SUCCESS.

### 6.2 Cisco IOS-XR (`cisco.py`)

```
get_steps(_) = [
    ValidatePlatform,
    CheckFiles,
    CheckVersions,         # → various jump targets — see below
    GetBootTime,
    CopyImage (pc),
    SoftwareAdd (pc),      # 'install add' — async via op-id
    InstallPrepareClean,
    SoftwarePrepare (pc),  # 'install prepare <op-id>' — async
    SoftwareActivate (pc), # 'install activate' — may set reload_required
    SoftwareCommit (pc),   # 'install commit' — gated on reboot completion if reload_required
    Cleanup,               # delete copied images
    Done
]
```

Notable behaviors:

- **`CheckVersions`**: parses `state.packages` from filenames (`.iso` → `get_version_from_iso`; else `get_file_packages` — list inside the .tar). Sets `state.virtual_router = 'xrv' in active_base_version`.
- **`CheckVersions.next_step`**:
  - For `SoftwarePackComponentPatch`:
    - `missing_active != []`  → `GetBootTime` (do the full install)
    - `missing_committed != []` (but all active)  → `SoftwareCommit`
    - else  → `Cleanup`
  - For `SoftwarePackComponentBase`:
    - `active != target` → `GetBootTime`
    - `committed != target` (active matches) → `SoftwareCommit`
    - else  → `Cleanup`
- **`CopyImage`**: like SROS but no checksum verification (`# file checksum not available on device`).
- **Asynchronous Cisco install ops** (Add/Prepare/Activate/Commit) follow this pattern:
  1. `pre_check`: if a prior `op_id_*` is set, `o.fetch_install_operation_log(op_id)`. If `done && success` → SKIP_STEP. If `not done` → WAIT. Else SUCCESS (re-execute).
  2. `execute`: invoke the corresponding RPC; record the new `op_id_*` on `state`; on synchronous error → FAILURE; else `_monitor_operation_log` (polls log up to 600s for "Ending operation N").
- **`SoftwareActivate`**: also runs an alternative "all packages already active" SKIP_STEP path. Detects `reload_required` from the activate op-log via `o.activate_requires_reload(log)`.
- **`SoftwareCommit.pre_check`**: gates on:
  - `reload_required && boot_time stale` → WAIT
  - `o.get_current_install_request()` non-None → WAIT
  - `op_id_commit && fetch... done+success` → SKIP_STEP (already committed)
  - any remaining "missing" packages still not active → WAIT
- **`InstallPrepareClean.execute`**: clears any pending prepare; on success calls `state.restart_prepare_clean()`.
- **`SoftwarePrepare.execute`**: calls `state.restart_prepare()` first; needs `state.op_id_add`.
- **`SoftwareActivate.execute`**: calls `state.restart_activate()` first.
- **`Cleanup.execute`**: SCP-deletes every file in `state.destination_paths.values()`.

### 6.3 Junos (`junos.py`)

```
get_steps(state) = [
    ValidatePlatform,
    CheckFiles,
    # if dual_re:
    GetBootTime[backup], CheckVersions[backup], CopyImage[master], CopyImage[backup],
    DisableFailover, PackageAdd[backup], Reboot[backup], Done[backup],
    # then (master, always):
    GetBootTime[master], CheckVersions[master], CopyImage[master], PackageAdd[master],
    Reboot[master], Done[master],
    # if dual_re:
    EnableFailover,
    # always:
    Done
]
```

Where `[backup]`/`[master]` are the parametrized step types (see §4.3). `state.route_engine_priority = [master_re_id, backup_re_id]` is set by `ValidatePlatform`.

Notable:

- **`ValidatePlatform`** populates `state.dual_re`, `state.switch`, `state.virtual_router`, and the per-RE `state.route_engine[<re_id>]`. If `dual_re` changes mid-request → FAILURE (request invalidated). Switch + dual-RE = unsupported (FAILURE).
- **`CheckVersions[re_id]`** mirrors SROS but per-RE; jump targets: `Done[re_id]` (already at target) or `Reboot[re_id]` (already rebooted).
- **`CopyImage[re_id]`** for `re_id != '0'` does an *on-device* `file copy re0:<path> re1:<volume>` instead of SCPing again.
- **`DisableFailover`**: stash current failover config in `state.failover_config`; SKIP_STEP if not configured; otherwise disable.
- **`EnableFailover`**: re-apply `state.failover_config` if it was set; else SKIP_STEP.
- **`PackageAdd[re_id]`**: `o.package_add(file, re_id)` per file; FAIL if any returns False.
- **`Reboot[re_id]`**: same WAIT-after-execute pattern as SROS; per-RE `state.route_engine[re_id].rebooted = True`.
- **`Done[re_id]`**: sets either `state.done = True` (no re_id) or `state.route_engine[re_id].done = True`.

### 6.4 VRP, Snabb, ONS-TL1, HGW

The `os` enum in YANG is `ios-xr | vrp | junos | sros`. Only `ios-xr`, `junos`, and `sros` have step modules wired into `_find_executor_module()` — `vrp` is recognized in capabilities but has **no implementation here**. Snabb, ONS-TL1, HGW are recognized only by the `device_os` detector (used by `validate_platform()`), and presumably exist only for legacy reasons. **The Acton port should target `ios-xr`, `junos`, `sros` only.**

---

## 7. Run-log

`run-log` is a flat per-request list keyed by `when` (yang:date-and-time, microsecond precision). Entries are produced by an `OperCdbLoggingHandler` attached to the `software-install` Python logger — every log record (≥ INFO level) emitted while a request is being executed creates an entry.

Each entry: `{when, run-id, component, step, message}`.

Records are enriched by a `_log_filter` that adds `swi_request_path`, `swi_component`, `swi_step`, `swi_run_id` from the active `Context`. Only records with `swi_component` (i.e., emitted in the package's logger) are persisted.

Truncation/retention: **none**. The user must call `clear-run-log` explicitly. There is no automatic pruning.

---

## 8. Background Worker Architecture

`request_worker.py` runs a producer/consumer setup using `bgworker` (NSO-side process management). Two queues:

- `q1` — scheduler → workers: `(request_path, user_session_id)` jobs
- `q2` — workers → scheduler: `(request_path, user_session_id, status)` updates **and** `(path, _, UNPROCESSED)` from `RequestWorker.new_request(...)` / `(path, _, CANCELLED)` from `cancel_request(...)`

### 8.1 Scheduler (`cdb_scheduler`)

A loop that:
- Pulls from `q2` (with a 1s timeout via `select`).
- On `CANCELLED`: kill the worker PID with SIGINT, set `request.status = cancelled`.
- On `PROCESSING`: record the worker PID in `workers[job]`.
- On `FAILED_OTHER` / `FAILED_TRANSIENT`: read `error_count.{transient, other, backoff}`, give up if `transient + other > config.max_retries`. Else compute next backoff:
  - if `config.backoff.factor` is set: `backoff = (request.error_count.backoff or 10) * factor`
  - else: `backoff = config.backoff.constant`
  - persist `request.error_count.backoff = backoff`
  - schedule `(time.time() + backoff, job, usess_id)` in a `PriorityQueue` waitroom.
- On `UNPROCESSED`: enqueue immediately to `q1`.
- On every iteration: pop ready waitroom entries (top of priority queue is in the past) and re-queue to `q1`.

**Key invariants:**
- Each request retries on its own backoff schedule.
- On `DONE`, `error_count` is cleared (in `_write_request_status`).
- Backoff state persists across worker restarts (it lives in CDB).

### 8.2 Worker (`cdb_worker`)

Per worker process loop:
1. Block on `q1.get(timeout=1)`.
2. Open `Maapi`+`Session` (system user).
3. Notify scheduler: `(job, pid, PROCESSING)` on `q2`.
4. Acquire `MaapiLocker.lock(/ncs:devices/ncs:device[name=<dev>], timeout=120)` — partial CDB lock blocking everyone else from touching the device.
5. On lock failure: log `'lock-error'` on `q2` and continue.
6. On lock success: invoke `step_executor(context, job)` (one full run, §3.3).
7. Notify scheduler: `(job, usess_id, result)` where `result` is the `RequestStatus` returned.

The `MaapiLocker` is a context-managed wrapper around `Maapi.lock_partial(RUNNING)` — busy-waits with sleep(1) until acquired or timeout.

`RequestWorker` (an `ncs.application.Application`):
- Spawns 1 scheduler + N workers (default 2).
- Provides static methods `new_request(path, usess_id)` and `cancel_request(path, usess_id)` to inject events into `q2`.
- `ExecuteRequestAction` calls `RequestWorker.new_request(...)` from the action callback.

### 8.3 Driving condition

The worker is scheduled by `execute-request` (which calls `new_request → q2 (UNPROCESSED) → q1 → cdb_worker`). It is *not* triggered automatically by configuration. The doc claims (`/devices/device/software-pack/name`) "a background process periodically checks if the current versions… match" — but in the code, this loop is **not implemented**. The worker is purely event-driven.

---

## 9. Logging Context

`Context` carries `(maapi, log, request_path, component, step, run_id)` and is the per-run-thread plumbing for:
- `enter_step(name)` context manager → sets `context.step` for the duration; clears at exit.
- `add_oper_log(message)` → emits an ERROR-level log enriched with the current step (used by step impls for surfaceable errors).
- `_log_filter` adds `swi_*` attributes to every log record (allowing `OperCdbLoggingHandler` to write the run-log entry to the right path).

---

## 10. Idempotency / Restart Story (cross-cutting)

Several places lean on this:

1. **`create-request`** is idempotent in the value of the pack (§3.1).
2. **`step_executor`** is restartable: a half-finished request resumes from the first non-`reached`/`skipped`/`failed` step, plus the step's own `pre_check` short-circuits work that has already been done (e.g., file already copied, `op_id_*` already finished).
3. **`internal-state` JSON** is the resumption record — every step that progresses meaningfully writes back state (`destination_paths`, `op_id_*`, `boot_time`, `rebooted`, etc.).
4. **`State.reset()`** is the explicit "the device drifted, start over" lever, called from `CheckVersions` when the previously-Done version is no longer running.
5. **Step refresh** (`refresh_steps`) lets a request grow new steps mid-run without losing progress on existing ones.

A correct port must preserve all five.

---

## 11. Things explicitly out of scope of the port

These are present in the Python code but are NSO-implementation artifacts we do **not** need to port:

- The `bgworker.background_process.Process` dance and `multiprocessing.Queue` ipc.
- `MaapiLocker` (NSO partial CDB locks). Stratoweave has its own per-device serialization story (DeviceMgr is a single actor per device).
- `OperCdbLoggingHandler`'s `cli_write` to a user session — there is no NSO CLI session in stratoweave.
- `jsonpickle` serialization of state — Acton's typed structs and `lmdb` persistence will replace this.
- The `software_install_script.py` plumbing for SCP via `paramiko` and SROS CLI/NETCONF strategy switching — replace with stratoweave's NETCONF adapter (and a separate file-transfer mechanism if needed).
- VRP, Snabb, ONS-TL1, HGW.

---

## 12. Things to confirm with the user (decision points for Phase 3)

These came up during extraction; they shape the design and need an answer before coding:

1. **Action-style RPCs (`execute-request`, `cancel-request`, `confirm-step`, `clear-run-log`, `create-request`) — how do we expose them in stratoweave?** Stratoweave is reactive — there is no NSO-style action node. Options: (a) reactive only (make the worker run when the request subtree changes); (b) RESTCONF actions (if/when supported); (c) explicit actor messages. Probably (a) for `create-request` (idempotent on pack data anyway), (a)+per-request status field for `execute-request`/`cancel-request`/`confirm-step`, and (a) zero-out for `clear-run-log`.
2. **Where does operational state (per-request status, plan, run-log) live?** Three candidates: (i) gdata in a dedicated oper layer with a `TreeProvider`, (ii) actor-owned state exposed through a layer, (iii) lmdb-backed but not in any layer. (i) is the most stratoweave-native; (ii) is the most ergonomic; (iii) is opaque.
3. **NETCONF-only MVP?** The Python port's `software_install_script.py` calls SCP (paramiko), CLI exec, and NETCONF. Stratoweave's first-class transport is NETCONF. CLI and SSH/SCP would be substantial additional dependencies. Recommendation: NETCONF-only for MVP; document explicitly which Python steps that excludes (SROS `prepare_format_standby` was already NotImplementedError; SROS `prepare_save_rollback` similar; SCP for image upload is the big one — alternative is a NETCONF file-server or an out-of-band assumption).
4. **Junos parametrized step types — how do we represent them in Acton's typed world?** Recommendation: `(class_name: str, re_id: ?str)` tuple as the plan key, no runtime-generated subclasses. Step instance carries `re_id` as a constructor arg.
5. **Backoff strategy YANG choice (`factor` vs. `constant`).** Stratoweave/Acton has no native YANG `choice` representation question — `gen_adata` handles it, but worth confirming the generated type-shape matches what we want to read.
6. **`software-install-matrix`** — entirely unused by the existing logic. Port it as data-only or omit?
7. **First OS to implement.** Recommendation: **SROS** — single-RE (no `_partial_RE` complexity), simplest step list, and the key NETCONF-only subset (BOF reconfiguration via NETCONF) is already exercised in `NokiaSrosNetconfStrategy` so it's the most NETCONF-pure existing implementation.
