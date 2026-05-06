# 02 — Stratoweave `software-install` Module Design (v4)

> **Status: v4 — incorporates round-3 review feedback from `docs/reviews/06-codex-design-r3.md` and `docs/reviews/07-claude-design-r3.md`, integrated in `docs/reviews/08-integration-r3.md`. Pending round-4 review before Phase 4 implementation begins.**

> **Conditional on a small additive platform change**: `TransformActorParams.dynstate: ?gdata.Node` field (added by `_TransformTransaction.init_dynstate` and `_RFSTransaction.init_dynstate`). See §14 platform prerequisites. Without this, the runner cannot read its own restored dynstate at startup; an architectural workaround exists (§3.6 fallback) but is uglier.

Read `00-orientation.md` (project context + stratoweave concepts) and `01-software-install-logic.md` (language-agnostic spec extracted from the Python source) first. Where v4 differs structurally from v3, the change is anchored to a review item — see `docs/reviews/08-integration-r3.md`.

Companion documents:
- `docs/adr/cli-driver.md` — CLI driver via TextFSMPlus templates (Phase 5 implementation detail).

Markers used in this doc:
- **❓DECISION** — open question requiring user/team input
- **⚠️ASSUMPTION** — choice made absent input; should be sanity-checked
- **🆕** — added or substantially changed in v4

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
            ├── step_logger.act                # 🆕 StepLogger (swi_* attribute plumbing)
            ├── transform.act                  # the per-device Transform (plain ttt.Transform)
            ├── device_runner.act              # actor DeviceRunner (per-device)
            ├── runlog.act                     # RunLogHandler + bounded-ring helpers
            ├── local_file.act                 # LocalFileInspector (Acton-stdlib filesystem)
            ├── remote_file.act                # RemoteFileInspector (per-OS NETCONF)
            ├── file_transfer.act              # FileTransfer interface + NoopFileTransfer
            ├── ops.act                        # DeviceOps facade (NETCONF + CLI strategy)
            ├── platform_sros.act              # Phase 4 — SROS step impls
            ├── ops_sros.act                   # SROS DeviceOps impl (NETCONF strategy real)
            ├── platform_iosxr.act             # Phase 6
            ├── ops_iosxr.act                  # Phase 6
            ├── platform_junos.act             # Phase 6
            ├── ops_junos.act                  # Phase 6
            └── test_*.act                     # tests
```

The module is a **library**: apps opt in by adding the YANG to their layer stack and wiring the transform.

**Public API** (`sw_install.act`) — 🆕 takes `file.FileCap` per CL3_9:

```acton
def make_sw_install_transform(
    dev_registry: swdev.DeviceRegistry,
    file_cap: file.FileCap,                                                    # 🆕 for LocalFileInspector
    local_file_inspector: ?LocalFileInspector = None,                          # default: derived from file_cap
    remote_file_inspector_factory: ?proc(swdev.DeviceMgr) -> RemoteFileInspector = None,  # default: per-OS NETCONF
    file_transfer_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer = None,
    cli_session_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> CliSession = None,
    log_handler: ?logging.Handler = None,
) -> proc(ttt.Path, ?ttt.Layer) -> ttt.Node

SOFTWARE_INSTALL_YANG: yang.Module
```

This factory builds a per-device transform (see §7).

---

## 2. Module boundary — what apps integrate against

Apps:

1. **Add the YANG model** to their lowest config layer. The model augments `/sw-rfs:rfs[name]/` (the RFS-layer per-device list — same place sorespo's RFS transforms attach), plus `/sw-rfs:rfs[name]/scp-port?` as a sibling of `software-pack` (per H4 fix).
2. **Wire one `ttt.Transform` per RFS-list entry** — sw-install attaches inside the RFS list as a per-device transform with its own dynstate slice.
3. **Optionally provide factories** for `FileTransfer`, `CliSession`, and `RemoteFileInspector`. Phase 4 ships sane defaults: real `RemoteFileInspector` over NETCONF for SROS; `NoopFileTransfer`; no CLI session.
4. **Provide a `file.FileCap`** for the controller-side `LocalFileInspector` (caps aren't ambient in Acton).

App side:

```acton
import sw_install
import file

def get_layers(dev_registry, log_handler, db, file_cap):
    swi_factory = sw_install.make_sw_install_transform(
        dev_registry,
        file_cap,
        log_handler=log_handler,
    )
    layer0 = ttt.Layer('0', my_t_0_with_swi(swi_factory), layer1, db)
```

---

## 3. Where state lives — corrected (v3 carried; v4 adds §3.6 restart bridge and §3.7 dynstate-write classification)

Round-2 reviews showed the v2 split between transform `memory` and `dynstate` was incompatible with the platform. v3 collapsed everything into dynstate. v4 keeps that consolidation (both r3 reviewers explicitly endorsed it) and adds the missing piece: how restored dynstate reaches the runner actor at startup.

### 3.1 The ownership rule

| Surface | Contents | Mutability |
|---------|----------|------------|
| **Config (gdata)** | desired pack assignment, control triggers (generations + confirmations + per-request overrides) | external read/write |
| **Dynstate (gdata persisted via `update_dynstate`)** | **all** runner-owned operational state | runner-internal |
| **Oper (gdata published via `update_oper`)** | pure projection of dynstate plus computed view, including diagnostic projections of internal state | external read |
| **Transform `memory`** | **unused.** `transform_wrapper` returns `(empty, memory)` unchanged. | n/a |

### 3.2 Per-device dynstate schema

```acton
class SwInstallDynstate(value):
    # Idempotency anchors
    last_pack_snapshot: ?SoftwarePack          # last materialized pack-data — AUTHORITATIVE
    last_request_generation: u64
    last_start_generation: u64
    last_cancel_generation: u64
    last_confirm_all_generation: u64
    last_clear_run_log_generation: u64

    # Request lifecycle
    next_request_id: u32                       # always >= 1
    current: ?RequestState                     # at most one active request
    history: list[RequestState]                # bounded — see §3.4
```

`RequestState`:

```acton
class RequestState(value):
    request_id: u32
    pack: SoftwarePack
    confirm_steps: bool                        # per-request override resolved at materialization
    plan: ComponentPlan
    states: dict[str, State]                   # per-component
    status: RequestStatus
    run_id_count: u64
    run_log: list[RunLogEntry]                 # bounded ring; see §6.6
    run_log_dropped: u64
    error_count: ErrorCount                    # consecutive transient/other
    next_wake_at: ?datetime
    generation_token: u64                      # bumped on each new run / cancel
    obsolete: bool
```

### 3.3 Diagnostic projection of dynstate into oper

The Python `internal-state` opaque JSON blob is dropped from the v4 YANG. Operationally-useful fields project into the oper subtree as **typed leaves** under `request/component/`:

- per request `error-count/{transient, other, backoff}`, `next-wake-at`
- per component `destination-volume`, `destination-paths` (small map), `boot-time`
- SROS-only (per OS): `rebooted`
- IOS-XR-only (per OS, Phase 6): `op-id-add`, `op-id-prepare`, `op-id-activate`, `op-id-commit`, `packages`, `reload-required`
- Junos-only per RE (Phase 6): per-RE list with `version`, `boot-time`, `rebooted`

The OS-specific leaves use YANG `when` constraints so they don't appear on irrelevant requests. **What is no longer externally visible**: fields not surfaced as named leaves (rare; everything operationally interesting is named).

🆕 §15.5 wording fix per CL3_6: `internal-state` (opaque JSON blob) dropped; the diagnostic projections **are** RESTCONF-visible — that's the whole point.

### 3.4 Request history retention — simplified (🆕 CR3_6)

Top-level `dynstate.last_pack_snapshot` is the **authoritative idempotency baseline**. `history` retains:

- the latest of each terminal status (`done`, `cancelled`, `failed-other`, `failed-transient`, `obsolete`) — gives operators "what happened most recently per outcome";
- additional older entries up to a bound of 50 total (configurable in v2.0+);
- pruning never depends on idempotency state — `last_pack_snapshot` lives at the dynstate root, not in any request entry.

This removes v3's doubly-stored idempotency baseline (the entry retention rule was redundant with the top-level snapshot).

### 3.5 Generation-counter restore semantics — both directions (🆕 CL3_1)

Generation counters are durable in dynstate (each `last_<trigger>_generation`) and visible in config (each `<trigger>-generation`). The runner fires a trigger when `cfg > dynstate`. Restore from backup can break this in two directions:

- **`cfg < dynstate`** (config restored from older backup; dynstate current) → `cfg > dynstate` evaluates false; trigger appears non-fired until user manually bumps. **Safe direction** — only loses the trigger.
- **`cfg > dynstate`** (dynstate restored from older backup; config current) → `cfg > dynstate` evaluates **true** spuriously, possibly firing a trigger the operator didn't ask for. **Dangerous direction.** Worse: `dynstate.next_request_id` may be smaller than `max(published request[].id)`, causing fresh-request-id collisions.

**Defensive runner-side check on first reconciliation after startup:** if `max(history request_id, current.request_id) ≥ dynstate.next_request_id`, the runner enters a `restore-inconsistent` mode: refuses to materialize new requests, publishes `last-trigger-result = {kind: rejected, reason: "restore inconsistency: published request id ≥ next_request_id"}` to oper. Operator must explicitly increment `next_request_id` (a v2.0 platform recovery API; for v1, manually edit dynstate or restore both backups together).

This lands in §15.5 #5: generation counters are not designed to survive a config-only restore that doesn't restore dynstate alongside it. Recovery is fail-loud, not silent.

### 3.6 How the runner receives restored dynstate (🆕 H2 — depends on platform addition)

**Preferred path (D3a — platform addition):** `TransformActorParams` gains a `dynstate: ?gdata.Node` field; `_TransformTransaction.init_dynstate` and `_RFSTransaction.init_dynstate` thread `self.dynstate` through. Five-line additive platform change in `ttt.act`. The runner reads its restored dynstate at construction time:

```acton
proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
    # params.dynstate is the restored dynstate (None on first boot)
    initial_dynstate = SwInstallDynstate.from_gdata(params.dynstate) if params.dynstate else SwInstallDynstate.empty()
    runner = DeviceRunner(
        params.path, params.update_oper, params.update_dynstate,
        dev_registry.get(devname_from_path(params.path)),
        initial_dynstate,
        ...
    )
    return lambda cfg, mem: runner.on_local_config(cfg, mem)
```

**Fallback (D3b — if platform team rejects):** `transform_wrapper(cfg, linked, memory, dynstate)` does receive `dynstate`. Stash it on the `TransformFunction` instance (which is the same instance that holds `_on_conf`); runner reads it on first `on_conf`:

```acton
class SwInstallTransform(ttt.TransformFunction):
    var stashed_dynstate: ?gdata.Node = None      # set on first transform_wrapper call after restore
    def transform_wrapper(self, cfg, linked, memory, dynstate):
        self.stashed_dynstate = dynstate          # stash for runner to read
        return (gdata.Container(), memory)
```

Runner's first `on_conf(cfg, memory)` checks `self.fn.stashed_dynstate`; subsequent calls ignore it (runner's own writes are authoritative thereafter).

⚠️ASSUMPTION (D3 D-DECISION): platform team accepts the additive `TransformActorParams.dynstate` change. **Round-4 question.** D3b workaround is uglier but viable.

### 3.7 Dynstate-write classification — three tiers (🆕 CR3_3)

v3 §8.3 said high-frequency writes coalesce. That's right for telemetry, NOT for restart-critical fields. v4 classifies dynstate fields into three tiers per their persistence requirements:

**Tier A — MUST persist before side effect** (block until LMDB write completes):
- `last_<trigger>_generation` values — persist before the trigger's work begins.
- `next_request_id` — persist before publishing a fresh request id externally.
- `current.error_count.*`, `current.next_wake_at` — persist before scheduling `after backoff: _start_run`.
- IOS-XR `op_id_*` values (Phase 6) — persist before issuing the device RPC that returns the op-id.

A crash between persist and side-effect leaves the trigger consumed but the side-effect un-issued; the runner's reconciliation handles "trigger consumed, no work in flight" cleanly (no double-fire).

**Tier B — persist at step boundary** (write at step completion, NOT mid-step):
- `current.plan` (after each step's status transition).
- `current.states[<component>]` (after each step's NewState commit).
- `current.run_id_count` (at run start).
- `current.status` (at status transition).
- `current.generation_token` (at bump).
- `destination_volume`, `destination_paths`, `boot_time`, `rebooted` (after the step that updates them).

**Tier C — best-effort telemetry** (in-memory; periodic flush, lost on crash):
- `current.run_log` entries — typically a flush-on-step-completion is sufficient. A few seconds of run-log loss on crash is acceptable; the persistent tiers carry restart-critical info.
- `current.error_count.transient`/`.other` between WAIT polls within a single step (intra-step polling counts are not persisted; they reset on retry).

The runner's `_persist_dynstate(tier)` accepts a tier argument; Tier A is synchronous (await ack), Tier B is sync at step boundaries, Tier C is coalesced.

### 3.8 What about `internal-state`?

Dropped from YANG; replaced by typed diagnostic projections (§3.3). Detailed §15.5 #1 entry (corrected per CL3_6).

---

## 4. Control surface — reactive triggers replacing NSO actions

(Substantially unchanged from v3 §4 — both reviewers explicitly endorsed the generation-counter pattern. v4 changes: drop `request-target-id` per CR3_9; add `last-trigger-result` per CR3_5; add confirmation pruning rules per CR3_7; tighten silent-error policy.)

### 4.1 `create-request` ↔ pack-data change OR `request-generation` increment

**Trigger:** `(pack-data ≠ dynstate.last_pack_snapshot) OR (request-generation > dynstate.last_request_generation)`.

**Behavior:**
- Materialize a new request with id = `dynstate.next_request_id` (starts at 1).
- Bump `dynstate.next_request_id` and persist Tier A.
- Mark `dynstate.current` (if any) as obsolete; move to `dynstate.history`.
- Snapshot pack-data into `RequestState.pack`.
- Update `dynstate.last_pack_snapshot` and `dynstate.last_request_generation` (Tier A).
- Publish via `update_oper`, including `last-create-result`.

**Cancelled-reactivation:** pack-data unchanged + last request was `cancelled` → user must increment `request-generation` to force a fresh request.

🆕 v4: drop `request-target-id` from YANG (per CR3_9 — request-generation creates a *new* request, scoping by id is conceptually nonsensical).

### 4.2 `execute-request` ↔ `start-generation`

`unprocessed → processing` requires either `start-generation > dynstate.last_start_generation` OR `auto-execute-after-confirm = true` AND all required confirmations are in place.

### 4.3 `confirm-step` ↔ writeable confirmations under `control/` (🆕 pruning rules per CR3_7)

Per-step confirmations live in config:

```yang
container control {
    list confirmation {
        key "request-id component step";
        leaf request-id { type uint32; }
        leaf component { type string; }
        leaf step { type string; }
        leaf by-user { type string; mandatory true; }   // 🆕 mandatory per L1
    }
    leaf confirm-all-generation { type uint64; default 0; }
    ...
}
```

Pruning / lifecycle rules:

- **Stale entries (request id pruned from history):** silently retained in config (the runner doesn't modify user-controlled config) but no-op'd. Operator can clean them up; no automated removal.
- **Future-id entries (request id not yet materialized):** observed when the matching request materializes. Allows pre-staging confirmations before the request exists — useful for automation.
- **`confirm-all-generation` increment:** does NOT create persistent `control/confirmation[]` entries. Sets internal `confirmed_implicitly` markers in `RequestState.plan` (dynstate); runner stamps `confirmed.{by-user, when}` in oper. The next request created after a confirm-all is NOT auto-confirmed unless its own confirm-all-generation increments past the device's last-observed.

### 4.4 `cancel-request` — state machine with `cancelling` (🆕 CR3_1 two-lane callback)

**State transitions:**

```
processing  ──(cancel-generation +1)──▶  cancelling
                                            │
                                            │  (in-flight RPC eventually returns OR adapter
                                            │   timeout fires; in either case
                                            │   _drain_notify(token) advances state)
                                            ▼
                                        cancelled
```

🆕 **Two-lane callback rule** (per CR3_1): the runner's stale-callback discipline now distinguishes two cases:

- A stale callback from a step's `pre_check`/`execute` (returning `(StepResult, NewState, ?Exception)`) — its **plan-state mutations are dropped** (the step result is no-op'd). But the runner is notified via `_drain_notify(token)` that the in-flight operation has drained.
- `_drain_notify(token)` checks whether the device runner's `current.status == cancelling` and `token == current.generation_token - 1` (i.e., this is the cancelled run's drain); if so, transitions `cancelling → cancelled` and persists Tier B.

🆕 **Adapter timeout watchdog** (per CR3_1): cancel adds a watchdog `after CANCEL_DRAIN_TIMEOUT: _force_drain()` that, if the in-flight callback never returns within (default) 600 seconds, transitions to `cancelled` regardless. This avoids `cancelling` becoming permanent if the device or adapter wedges.

### 4.5 `clear-run-log` ↔ `clear-run-log-generation`

Empties the targeted request's run-log ring + resets `run_log_dropped` to 0. Run-log `seq` resets to 0 (and `when` baselines such that `(when, seq)` collisions are impossible — the ring is empty post-clear, so nothing to collide with).

### 4.6 Per-request scoping — fail-loud on missing target (🆕 CR3_5)

Each generation counter (except `request-generation`, which doesn't take a target — H4 fix) has an optional companion `*-target-id` leaf. If set, the trigger applies to that specific request id only. **🆕 Missing target id is reported via `last-trigger-result`:**

```yang
container last-trigger-result {
    config false;
    leaf trigger-kind { type enumeration { enum start; enum cancel; enum confirm-all; enum clear-run-log; } }
    leaf generation { type uint64; }
    leaf target-id { type uint32; }
    leaf result { type enumeration { enum accepted; enum rejected; } }
    leaf reason { type string; }
    leaf at { type yang:date-and-time; }
}
```

When a trigger fires:
- if `*-target-id` is set and the request exists → result=accepted, runner acts.
- if `*-target-id` is set and no such request id → result=rejected, reason="no such request id <N>", runner does nothing.
- if `*-target-id` is unset → applies to latest request; result=accepted with target-id=<latest>.

Single-slot last-writer-wins; preserves the simple model. Operator gets feedback for invalid target ids.

### 4.7 `enabled` semantics — state transition table (🆕 §8.6 gate per CL3_4)

| `enabled` transition | Current request status | New request status | Action |
|----------------------|------------------------|--------------------|--------|
| `true → false` | `unprocessed` | `unprocessed` | nothing in flight; new requests still materialize |
| `true → false` | `processing` | `paused` | `generation_token` bumps; current step's RPC may complete; no further steps execute |
| `true → false` | `failed-transient` (mid-backoff, `next_wake_at` set) | `paused` | `next_wake_at` retained; **`_start_run()` checks `enabled` and refuses to start** (CL3_4); when fired, transitions to `paused` |
| `true → false` | `waiting-confirmation` / `waiting-for-device` / `cancelling` / terminal | unchanged | nothing to pause |
| `false → true` | `paused` | `processing` (if `start-generation` was already consumed) OR `unprocessed` (if not) | resume per-policy; replay backoff schedule with `after max(0, next_wake_at - now): _start_run()` |
| `false → true` | other | unchanged | normal triggers apply |

🆕 **`_start_run()` first-line check** (CL3_4): `if not current_global_config.enabled: transition to paused; return`. Backoff `after` block fires regardless of enabled; runner re-checks at execution.

### 4.8 YANG diff vs Python original

| Item | v4 plan |
|------|---------|
| `software-pack` list (global) | unchanged |
| `software-install-matrix` | dropped |
| `confirm-steps`, `auto-execute-after-confirm`, `error-handling/...` | unchanged |
| `request-status` enum | adds `paused`, `cancelling`, `waiting-for-device` |
| `request[]` and below | all `config false` |
| 🆕 augment `/sw-rfs:rfs[name]/scp-port` | direct sibling of `software-pack` (H4 — corrected from "direct device augmentation"; lives on the RFS list, not on the device meta-config list, and that's correct) |
| 🆕 per-device `/sw-rfs:rfs[name]/software-pack/control/` subtree | generation counters (no `request-target-id`) + target-ids on others + confirmations + per-request options |
| 🆕 per-device `/sw-rfs:rfs[name]/software-pack/last-create-result` | oper feedback for create-request |
| 🆕 per-device `/sw-rfs:rfs[name]/software-pack/last-trigger-result` | oper feedback for start/cancel/confirm-all/clear-run-log |
| `internal-state` leaf | dropped — replaced by typed diagnostic projections under `request/component/` |
| Action nodes | all dropped |
| `vrp` enum value | kept; `ValidatePlatform` fails clean |
| 🆕 `run-log` key | `(when, seq)` |
| 🆕 OS-specific diagnostic leaves | `when` constraints scope them to the relevant OS |

---

## 5. Typed data model

(Unchanged from v3 — both reviewers endorsed.)

Two layers: `gen_adata`-generated typed accessors for the YANG, plus internal value types in `pack.act` / `state.act` tuned for state-machine use.

Per-OS State subclasses each implement `reset()`. See `state.act` skeleton in v3 §5; carried unchanged.

---

## 6. Plan + step semantics

### 6.1 `Step` protocol — formalized (🆕 CL3_7)

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

# StepCallback signature:
#   cb(result: StepResult, new_state: ?State, exc: ?Exception)
#
# Convention: `exc` is non-None iff the step body raised; runner logs traceback and
# treats as FAILURE regardless of the StepResult value. `new_state = None` means
# "no state change for this step" (rare; most steps return an updated state even on
# SKIP_STEP because they may have observed something worth recording).

class Step(object):
    key: StepKey
    proc def pre_check(self, state, ops: DeviceOps,
                       lfi: LocalFileInspector, rfi: RemoteFileInspector, ft: FileTransfer,
                       step_log: StepLogger,                                       # 🆕 per-step logger
                       cb: action(StepResult, ?State, ?Exception) -> None) -> None: ...
    proc def execute(self, state, ops: DeviceOps,
                     lfi, rfi, ft, step_log,
                     cb: action(StepResult, ?State, ?Exception) -> None) -> None: ...
    def next_step(self, state) -> ?StepKey: ...
    def supports_pre_check(self) -> bool: ...
```

Step methods receive **six** parameters (corrected from v3's "four"): `state`, `ops`, `lfi`, `rfi`, `ft`, `step_log`. Plus the `cb` callback.

### 6.2 Step contract invariants (🆕 CL3_2 enforcement clarity)

- **Steps are ordinary classes**, not actors. The runner constructs `cb` as an `action def` defined on itself; closing over `self` makes the callback dispatch on the runner mailbox automatically. Steps that need helper actors must terminate them before invoking `cb`.
- **`next_step` jump target validation:** if `next_step(state)` returns a `StepKey` not present in the current plan, the runner emits a clear log entry and returns `FAILURE` for the current step.
- **Failure isolation:** a step's exception surfaces as `(FAILURE, ?State, exc)` from the callback. Runner logs and proceeds with FAILURE handling; doesn't propagate up the actor.

### 6.3 `ComponentPlan` invariants (unchanged)

- Refresh after every step.
- Monotonic — may add steps, never removes prior components or steps.
- Flush ordering: NewState persisted only if `result != FAILURE`; refresh; persist plan (Tier B).

### 6.4 Per-OS step lists

(Unchanged from v3.)

### 6.5 Status mapping at run end (unchanged)

Consecutive counters; FAILURE resets `transient`, WAIT resets `other`. DONE clears `error_count` (including `error_count.backoff`).

### 6.6 Run-log filter, plumbing, and bounded ring (🆕 CL3_3 logging plumbing)

Acton's stdlib logging (`acton/base/src/logging.act:Logger`) takes structured data per-call as `data: ?dict[str, ?value]`. There's no thread-local context, no `logging.Filter` chain. Two consequences:

🆕 **`StepLogger` injected into step methods** (CL3_3): the runner constructs a `StepLogger` per-step, pre-bound to `swi_request_path / swi_component / swi_step / swi_run_id`:

```acton
class StepLogger(object):
    _runner_logger: logging.Logger
    _swi_extra: dict[str, ?value]      # request_path, component, step, run_id

    def info(self, msg: str, extra: ?dict[str, ?value] = None):
        merged = dict(self._swi_extra)
        if extra is not None:
            merged.update(extra)
        self._runner_logger.info(msg, merged)

    def warning(self, msg: str, extra: ?dict[str, ?value] = None): ...
    def error(self, msg: str, extra: ?dict[str, ?value] = None): ...
    def debug(self, msg: str, extra: ?dict[str, ?value] = None): ...
```

Step authors use `step_log.info(msg, extra)` and don't need to remember the swi_* keys.

🆕 **`RunLogHandler`** — concrete handler installed on the per-device runner's logging chain:

```acton
class RunLogHandler(logging.Handler):
    _runner: DeviceRunner

    def emit(self, record: logging.LogRecord):
        # Records bearing 'swi_component' in their structured-data dict
        # are persisted into the runner's bounded run-log ring.
        # Records without 'swi_component' pass through to other handlers.
        if 'swi_component' in record.data:
            self._runner._append_runlog_entry(record)
```

Records flowing past from other modules (without swi_* keys) are not persisted. The handler is installed on the runner's `Logger` chain at runner construction.

**Bounded ring buffer:** default 1000 entries/request; oldest dropped when full; `run_log_dropped` counter incremented; `(when, seq)` keying with seq monotonic per request, reset to 0 on `clear-run-log-generation` increment (ring is also emptied at clear, so post-clear seq=0 starts fresh with no collision risk).

### 6.7 Retry budget (unchanged)

Per-class budgets; backoff formula `(error_count.backoff or 10) * factor` matches the spec exactly.

🆕 **Backoff rounding** (CR3_4): internally compute as decimal; round to `ceil(seconds)` when persisting to `error_count.backoff` (uint32) and projecting `next_wake_at`. The fractional sequence `(10.0, 12.0, 14.4, 17.28, ...)` becomes `(10, 12, 15, 18, ...)` after ceiling.

---

## 7. The Transform substrate — plain `ttt.Transform` (🆕 H1 — switched from RFSTransform)

Both r3 reviewers identified that v3's `RFSTransform` plan doesn't work:
- `_RFSTransaction.finalize()` suppresses `on_conf` when output is empty (sw-install produces empty downward output, so callbacks would be lost).
- `RFSFunction.init_dynstate` doesn't pass `params.lower` (Option B's subscription path needs it).

v4 switches to plain `ttt.Transform`. Concrete differences:
- **Plain `Transform` accepts empty output without suppressing on_conf** (no equivalent guard in `_TransformTransaction.finalize`).
- **Plain `Transform` populates `params.lower`** — the subscription path works.
- **Plain `Transform` does NOT populate `params.dev`** — the runner extracts devname from `params.path` and calls `dev_registry.get(devname)` itself. Same pattern as `_DeviceTransaction.devname_from_device_path` (`ttt.act:2206`).

### 7.1 Wiring topology

The host layer composition (in the per-app `t_<n>.act`) wires sw-install as a per-device transform inside the `/sw-rfs:rfs` list:

```acton
# Host layer composition (sketch):
ttt.List(
    ttt.Container({
        q("name"): ttt.Leaf(),
        q("scp-port"): ttt.Leaf(),
        q("software-pack"): swi_factory,        # ← sw-install per-device transform
    }),
    [q("name")],
)
```

Each per-device transform's `act`-spawned actor is the `DeviceRunner` for that device. No top-level coordinator.

### 7.2 Global config subscription

Per-device transforms read `/software-install/...` (pack library, `enabled`, `error-handling`) via `params.lower.declare_subscriptions(...)`:

```acton
proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
    devname = devname_from_path(params.path)
    initial_dynstate = SwInstallDynstate.from_gdata(params.dynstate) if params.dynstate else SwInstallDynstate.empty()
    runner = DeviceRunner(
        params.path, params.update_oper, params.update_dynstate,
        dev_registry.get(devname),
        initial_dynstate,
        ...
    )
    if params.lower is not None:
        params.lower.declare_subscriptions(
            owner_id="sw_install:" + devname,
            cb=runner.on_global_config,
            want={SubscriptionSpec(filt=SOFTWARE_INSTALL_FILTER, period=...)},
        )
    return lambda cfg, mem: runner.on_local_config(cfg, mem)
```

⚠️ASSUMPTION (round-4 question): `Layer.declare_subscriptions` reads from the layer it's called on (`params.lower`). The host-layer global config (`/software-install/...`) needs to be present in the lower layer for this subscription to see it. **Apps integrating sw-install must ensure `/software-install/...` is in the layer below the sw-install layer** — typically by passing it through the layer-stack composition. This is a wiring constraint apps need to know about. Document in the README + §2 integration guide.

If the platform team would prefer a "subscribe to current layer root" API, that's a v2.0 platform ask in §14.

### 7.3 Transform body

```acton
class SwInstallTransform(ttt.TransformFunction):
    def transform_wrapper(self, cfg, linked, memory, dynstate):
        # No downward output. Memory unchanged. Dynstate is observed by the runner via
        # params.dynstate at construction (§3.6 D3a) — transform_wrapper itself doesn't
        # transform it.
        return (gdata.Container(), memory)
```

Empty output is delivered to the lower layer; plain `Transform.finalize` does NOT suppress `on_conf` (verified vs `ttt.act` `_TransformTransaction.finalize` — no equivalent guard).

---

## 8. DeviceRunner architecture (per-device transform)

### 8.1 Lease scope — honest downgrade (unchanged)

(See v3 §8.1; both reviewers accepted the downgrade story unchanged.)

The DeviceRunner is sw-install-internal serialization, NOT a system-wide device lease. RFS transforms and external `rpc_xml` callers are not blocked. v2.0 platform task: `DeviceMgr.acquire_exclusive(...)`.

### 8.2 The DeviceRunner actor

```acton
actor DeviceRunner(
    path: ttt.Path,
    update_oper: action(?gdata.Node) -> None,
    update_dynstate: action(?gdata.Node) -> None,
    dev: swdev.DeviceMgr,
    initial_dynstate: SwInstallDynstate,        # 🆕 from params.dynstate restore (§3.6)
    local_fi: LocalFileInspector,
    remote_fi_factory: proc(swdev.DeviceMgr) -> RemoteFileInspector,
    file_transfer: ?FileTransfer,
    ops_factory: proc(swdev.DeviceMgr, ?CliSession) -> DeviceOps,
    cli_session_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> CliSession,
    log_handler: ?logging.Handler,
):
    var dynstate: SwInstallDynstate = initial_dynstate    # 🆕
    var global_config_cache: ?GlobalSwInstallConfig = None

    # Restore-inconsistency guard (§3.5)
    var restore_inconsistent: bool = self._check_restore_consistency()

    action def on_local_config(cfg: ?gdata.Node, mem: ?gdata.Node):
        if restore_inconsistent:
            self._publish_restore_error()
            return
        # Pure idempotent reconciliation — A4 invariant 1.
        ...

    action def on_global_config(g: ?gdata.Node, err: ?Exception):
        ...

    action def _step_callback_guard(token: u64, then: action() -> None):
        # Stale callbacks: drop plan-state mutations
        if dynstate.current is not None and token == dynstate.current.generation_token:
            then()
        else:
            self._drain_notify(token)             # 🆕 CR3_1 — two-lane

    action def _drain_notify(stale_token: u64):
        # Stale callback drained — if we're cancelling that token, advance to cancelled
        if dynstate.current is not None and dynstate.current.status == CANCELLING and stale_token + 1 == dynstate.current.generation_token:
            dynstate.current.status = CANCELLED
            self._persist_dynstate(Tier.B)
            self._publish_oper()

    # ... (other helpers per §8.3-§8.7)
```

### 8.3 Re-entrancy invariants — three rules

(Unchanged from v3; carried.)

1. `on_local_config` and `on_global_config` are pure idempotent reconciliation functions.
2. Generation observations are persisted (Tier A) before any work begins.
3. Tier-classified writes per §3.7.

### 8.4 Restart story (🆕 H2 — depends on §3.6 platform addition)

On platform startup:
1. Platform restores the transform's `dynstate` from lmdb (`_TransformTransaction.restore`).
2. The `act` callback fires; runner is constructed with `initial_dynstate = SwInstallDynstate.from_gdata(params.dynstate)` (or `empty()` if first boot).
3. Runner runs §3.5 restore-consistency check; if inconsistent, sets `restore_inconsistent = True` and publishes the error.
4. Runner inspects `dynstate.current.status`:
   - `processing` at crash → `failed-transient` + `error_count.transient += 1`. Scheduler's normal retry loop re-runs.
   - `cancelling` at crash → no live RPC remains; transition directly to `cancelled`.
   - `paused` → remain paused.
   - `waiting-confirmation` / `waiting-for-device` → remain.
   - terminal → no action.
5. Runner persists Tier B and publishes oper.
6. First `on_local_config` call after startup fires reconciliation against restored dynstate.
7. **Restore backoff resume**: if `next_wake_at` is in the future, schedule `after max(0, next_wake_at - now): _start_run()`.

### 8.5 Cancel implementation (🆕 CR3_1 two-lane + watchdog)

```
on cancel-generation increment for current request:
    if cfg.<trigger>-target-id is set and not in current+history:
        publish last-trigger-result {kind: cancel, result: rejected, reason: "no such request"}
        return                                    # CR3_5: fail-loud
    persist dynstate.last_cancel_generation := new_value (Tier A)
    if dynstate.current.status == processing:
        dynstate.current.status = CANCELLING
        dynstate.current.generation_token += 1     # invalidates in-flight callbacks
        log "cancellation requested at <step>"
        self._persist_dynstate(Tier.B); self._publish_oper()
        # Schedule watchdog: if the in-flight RPC never returns, force drain.
        after CANCEL_DRAIN_TIMEOUT: self._force_drain(dynstate.current.generation_token)
    elif dynstate.current.status in (waiting-confirmation, paused, failed-transient, unprocessed):
        dynstate.current.status = CANCELLED
        self._persist_dynstate(Tier.B); self._publish_oper()
```

`CANCEL_DRAIN_TIMEOUT` default 600s (matches IOS-XR's `_monitor_operation_log` 600s timeout — never less than the longest in-flight RPC). `_force_drain` checks the token (in case a normal drain happened first), transitions to `cancelled` if still `cancelling`.

### 8.6 Backoff (🆕 CL3_4 enabled gate, 🆕 CR3_4 rounding)

```
on FAILURE / WAIT terminal of a run:
    error_count.<class> += 1
    error_count.<other-class> = 0
    if error_count.<class> > config.max_retries:
        publish terminal status, gave_up = True
        return
    backoff_decimal = (error_count.backoff_decimal or 10.0) * factor
    error_count.backoff = ceil(backoff_decimal)              # uint32 in oper
    error_count.backoff_decimal = backoff_decimal            # internal precise state
    next_wake_at = now() + ceil(backoff_decimal)
    self._persist_dynstate(Tier.A)                           # before scheduling
    after error_count.backoff: self._start_run()

action def _start_run():
    # 🆕 CL3_4: re-check enabled at firing time
    if not global_config_cache.enabled:
        dynstate.current.status = PAUSED
        self._persist_dynstate(Tier.B); self._publish_oper()
        return
    # ... normal start
```

### 8.7 Device-not-yet-ready case (🆕 CR3_8 polling spec)

If `dev_registry.get(name)` returns a DeviceMgr with `NoAdapter`, the runner enters `waiting-for-device`. Polling at fixed interval (default 30s):

```
action def _poll_device_readiness():
    if dynstate.current is not None and dynstate.current.status == WAITING_FOR_DEVICE:
        try:
            modules = dev.get_modules()
            if len(modules) > 0:                    # adapter is real, has modules
                dynstate.current.status = UNPROCESSED  # let on_local_config re-evaluate
                self._persist_dynstate(Tier.B); self._publish_oper()
                return
        except Exception:
            pass
        after 30: self._poll_device_readiness()      # keep polling
```

v2.0 platform ask: `DeviceMgr.on_status_change(cb)` event-driven equivalent.

---

## 9. Transport scope: NETCONF + tiered file abstractions

### 9.1 Op coverage (unchanged from v3)

(See v3 §9.1.)

### 9.2 `LocalFileInspector` — controller-side (unchanged shape; 🆕 CL3_9 file_cap)

```acton
class LocalFileInspector(object):
    proc def is_file(self, path: str, cb: action(bool, ?Exception) -> None) -> None: ...
    proc def get_size(self, path: str, cb: action(?u64, ?Exception) -> None) -> None: ...
    proc def hash(self, path: str, algo: str, cb: action(?str, ?Exception) -> None) -> None: ...
```

🆕 Default impl uses `file.FileCap` injected via `make_sw_install_transform`. The factory builds the LocalFileInspector if not provided by the app.

### 9.3 `RemoteFileInspector` — device-side via NETCONF (unchanged shape)

(See v3 §9.3.)

### 9.4 `FileTransfer` — Phase 5 byte-mover (🆕 CR3_2 CheckFiles deviation)

(Shape unchanged; see v3 §9.4.)

🆕 **Phase 4 `CheckFiles` deviation (CR3_2):** when `file_transfer.caps().put == false` (Phase 4's `NoopFileTransfer`), `CheckFiles.execute` skips the controller-side filesystem check entirely. `CopyImage.pre_check` (using RemoteFileInspector) becomes the sole file-presence verification: SKIP_STEP if all files present on device, FAILURE if missing (with clear "no FileTransfer configured — pre-stage the image" message).

This is a conscious deviation from the Python original (which always ran controller-side `CheckFiles`). Documented in §15.5 #15 (new entry).

### 9.5 Credential reuse — `DeviceMgr.get_dmc()` (🆕 H3 — already exists)

🆕 `DeviceMgr.get_dmc()` **exists in `device.act:401`** (verified in r3). v3's "platform ask" framing was documentation drift. v4 frames this correctly: `FileTransfer` uses an **existing** `DeviceMgr.get_dmc()` API.

```acton
file_transfer_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer = None
```

The factory builds a `FileTransfer` instance bound to the `DeviceMgr`; the instance calls `dev.get_dmc()` per transfer to get fresh credentials (DMC is mutable via `set_dmc(...)` which is called repeatedly).

⚠️ Note (per CR3_1 sub-note): `get_dmc()` is an actor method, so externally it's a mailbox-bound call. Verify the call shape compiles cleanly in Phase 5 — straightforward, but worth a one-line check.

### 9.6 `scp-port` placement — corrected narrative (🆕 H4)

The original Python YANG had `scp-port` directly on the per-device augment under the NSO `tailf-ncs` namespace. v4's YANG places it on `/sw-rfs:rfs[name]/scp-port`, **as a sibling of `software-pack`** (not nested inside).

Why `/sw-rfs:rfs` and not `/sw-rfs:device`: the RFS list is where per-device service config lives in stratoweave (sorespo augments `/sw-rfs:rfs` for the same reason). The `/sw-rfs:device` list is for *device-meta-config* (credentials, address, type). `scp-port` is closer to a service-side setting (it informs a Phase 5+ FileTransfer implementation) than to device meta-config.

**What "survives unbinding the pack" precisely means:** removing the `software-pack` container (presence-container) leaves `scp-port` intact (it's a sibling, not nested). It does NOT survive removing the device from the RFS list. Future non-sw-install consumers (e.g., a config-backup tool) can read `scp-port` from `/sw-rfs:rfs[name]/scp-port` if they want.

### 9.7 `DeviceOps` facade — CLI strategy boundary

(Unchanged from v3 §9.7 — both reviewers endorsed the boundary; details deferred to ADR.)

🆕 ADR cleanup per CR3_10: Phase 4 `SrosOps` has a `cli_session: ?CliSession` field; CLI strategy is not selected (NETCONF only). **No per-method NotImplementedError stubs** — that's dead surface. Phase 5 wires real CLI when ready.

---

## 10. Testing strategy (unchanged from v3)

(Carry; both reviewers explicitly approved.)

🆕 L8 test ordering: `test_dynstate.act` restart scenarios depend on §3.6 platform addition (or D3b workaround) being in place.

---

## 11. Implementation phasing within Phase 4

(Unchanged from v3.)

🆕 Phase 4 prerequisite landing the `TransformActorParams.dynstate` platform addition (see §14). If platform team rejects, fall back to D3b workaround (§3.6).

---

## 12. Open decisions (round-4 questions)

| # | Question | My current lean |
|---|----------|-----------------|
| **❓Q1** | `Layer.declare_subscriptions` from per-device transform: app composition must put `/software-install/...` in the lower layer for the subscription to see it. Is this an acceptable wiring constraint, or should we ask for a "current layer root" subscription API? | Acceptable for v1; document in README and §2. v2.0 platform ask if friction. |
| **❓Q2** | (was: DeviceMgr.get_dmc() platform addition) — **resolved**: get_dmc() already exists. Drop. | resolved |
| **❓Q3** | `auto-execute-after-confirm` default. | `false` (matches Python). |
| **❓Q4** | (was: credential reuse a/b/c) — **resolved**: use `DeviceMgr.get_dmc()` (§9.5). | resolved |
| **❓Q5** | Run-log default bound. | 1000 entries/request. |
| **❓Q6** | Phase 5 = TextFSMPlus + DeviceOps CLI + FileTransfer infra; Phase 6 = IOS-XR + Junos + polish. | confirmed by user. |
| **❓Q7** | 🆕 `TransformActorParams.dynstate` platform addition (§3.6 / §14). | Preferred; fallback exists (D3b). |
| **❓Q8** | 🆕 `CANCEL_DRAIN_TIMEOUT` default (§8.5). | 600s (matches IOS-XR longest poll). |

---

## 13. Implementation details deferred (unchanged)

(See v3 §13.)

---

## 14. Platform prerequisites for Phase 4 / v2.0

🆕 **Phase 4 prerequisites** (must land before Phase 4 implementation begins):

1. **`TransformActorParams.dynstate: ?gdata.Node`** — five-line additive change in `ttt.act`. Threaded through `_TransformTransaction.init_dynstate` and `_RFSTransaction.init_dynstate`. Existing callers ignore the new field. Without this, Phase 4 falls back to D3b (§3.6 transform_wrapper stash) — workable but uglier. **❓Round-4 question Q7.**

**v2.0 prerequisites** (sw-install lives without these in v1, can be requested for follow-up):

1. **`DeviceMgr.acquire_exclusive(owner_id, timeout, cb)`** + matching `release_exclusive` — gates `configure`, `rpc_xml`, `fetch_config`, `declare_subscriptions` paths under exclusive ownership. Closes the §8.1 lease gap.
2. **`Layer` "subscribe to current layer root" API** — current `declare_subscriptions` is on the lower-layer object; would simplify §7.2 wiring constraint.
3. **Config-restore event hook on `Layer`** — fires when config is restored from backup; lets sw-install reset `last_observed_*` counters automatically (§3.5).
4. **`DeviceMgr.on_status_change(cb)`** — event-driven replacement for the §8.7 polling fallback.
5. **`next_request_id` recovery API** — for the §3.5 restore-inconsistency case where operator manually resets dynstate counters.

---

## 15. Deferred features (unchanged)

(See v3 §15.)

## 15.5 Conscious deviations from the Python spec (🆕 entries from r3)

A consolidated list of intentional fidelity-vs-operability tradeoffs.

1. 🆕 **`internal-state` opaque JSON blob is dropped**; operationally-useful fields are typed leaves under `request/component/`, **still RESTCONF-visible** via diagnostic projections (§3.3). Fields not surfaced as named leaves are no longer externally inspectable — but the named-leaf set covers everything operationally interesting (corrected wording per CL3_6).
2. **Run-log bounded at 1000 entries/request** with `dropped-count` surfaced. Python was unbounded.
3. **Cancel takes effect at "next step boundary or RPC return," with explicit `cancelling` enum and a 600s drain watchdog.** Python `cancel-request` SIGINT'd the worker.
4. **Per-device install lease is sw-install-internal only.** Python `MaapiLocker` was system-wide. Operators must avoid concurrent RFS reconciliation against a device under upgrade.
5. 🆕 **Generation counters can go backward OR forward on partial backup-restore.** *Backward* (config older than dynstate): trigger appears non-fired until manual bump. *Forward* (dynstate older than config): runner detects via `next_request_id < max(published request id)` and enters `restore-inconsistent` mode, refusing new requests until operator resets dynstate counters. Python had no equivalent generation concept.
6. **`software-install-matrix` dropped from YANG.** Python had it but unused.
7. **`vrp` enum kept; no `vrp` step module.**
8. **Snabb/ONS-TL1/HGW dropped.**
9. **Action-style return values replaced by per-device `last-create-result` and `last-trigger-result` (single-slot, last writer wins).**
10. **CLI strategy methods exist as stubs in Phase 4.**
11. **`waiting-for-device` request status (no Python equivalent)** — handles DMC-not-yet-set / NoAdapter cases.
12. **`paused` request status (no Python equivalent).**
13. **Run-log key change: `(when, seq)` not just `when`.**
14. 🆕 **`scp-port` placement: `/sw-rfs:rfs[name]/scp-port` (sibling of `software-pack`, on the RFS-layer per-device list — NOT directly on `/sw-rfs:device`).** Survives unbinding the pack but not removing the device from the RFS list.
15. 🆕 **Phase 4 `CheckFiles` is a no-op when no `FileTransfer` is configured** (the pre-staged-image scenario). Python always ran controller-side `CheckFiles`. Phase 5 with real `FileTransfer` restores the original behavior.
16. 🆕 **Backoff is rounded to ceiling integer seconds** when projected to YANG `uint32` leaves. Python may have stored decimals internally; v1 surface is integer.

---

## 16. Round-4 review

This v4 design integrates all round-3 review feedback. Both reviewers' four convergent HIGH-priority issues (RFSTransform substrate, dynstate restore path, get_dmc docs drift, YANG path narrative) are addressed. All medium and low items folded in unless explicitly listed as deferred or platform-side.

**Stop here for round-4 review.** Both reviewers will be re-briefed against the full revised doc set (`00-orientation.md`, `01-software-install-logic.md`, `02-sw-install-design.md` v4, `docs/adr/cli-driver.md`) with no carry-over context.
