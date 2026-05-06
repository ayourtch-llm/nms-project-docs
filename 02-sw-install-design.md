# 02 — Stratoweave `software-install` Module Design (v5)

> **Status: v5 — incorporates round-4 review feedback from `docs/reviews/09-codex-design-r4.md` and `docs/reviews/10-claude-design-r4.md`, integrated in `docs/reviews/11-integration-r4.md`. Pending round-5 review before Phase 4 implementation begins.**

> **No platform-side prerequisites required for Phase 4 implementation** (down from v4's "platform addition required"). v5 promoted the `transform_wrapper`-stash dynstate-restore path to primary because actor construction runs before `Layer.load_from_db()` in the live lifecycle — so the originally-proposed `TransformActorParams.dynstate` field would be `None` at construction even if added. The stash path leverages the existing post-restore recompute and works without platform changes. See §3.6.

Read `00-orientation.md` (project context + stratoweave concepts) and `01-software-install-logic.md` (language-agnostic spec extracted from the Python source) first. v5 changes are anchored to review items in `docs/reviews/11-integration-r4.md`.

Companion documents:
- `docs/adr/cli-driver.md` — CLI driver via TextFSMPlus templates (Phase 5 implementation detail).

Markers used in this doc:
- **❓DECISION** — open question requiring user/team input
- **⚠️ASSUMPTION** — choice made absent input; should be sanity-checked
- **🆕** — added or substantially changed in v5

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
    │   └── src/gen_adata.act
    └── src/
        └── sw_install/
            ├── sw_install.act                 # public API
            ├── yang.act                       # raw YANG strings
            ├── model.act                      # GENERATED — typed accessors
            ├── pack.act                       # SoftwarePack / Component types
            ├── plan.act                       # ComponentPlan + StepStatus
            ├── state.act                      # State / StateSros / ...
            ├── step.act                       # Step protocol + StepResult
            ├── step_logger.act                # StepLogger (swi_* plumbing)
            ├── transform.act                  # the per-device Transform
            ├── device_runner.act              # actor DeviceRunner
            ├── runlog.act                     # RunLogHandler + ring helpers
            ├── local_file.act                 # LocalFileInspector
            ├── remote_file.act                # RemoteFileInspector
            ├── file_transfer.act              # FileTransfer + NoopFileTransfer
            ├── ops.act                        # DeviceOps facade
            ├── platform_sros.act              # Phase 4 SROS step impls
            ├── ops_sros.act                   # SROS DeviceOps (NETCONF strategy real)
            ├── platform_iosxr.act              # Phase 6
            ├── ops_iosxr.act                  # Phase 6
            ├── platform_junos.act             # Phase 6
            ├── ops_junos.act                  # Phase 6
            └── test_*.act                     # tests
```

The module is a **library**: apps opt in by adding the YANG to their layer stack and wiring the transform.

**Public API** (`sw_install.act`):

```acton
def make_sw_install_transform(
    dev_registry: swdev.DeviceRegistry,
    file_cap: file.FileCap,                                                    # for LocalFileInspector
    local_file_inspector: ?LocalFileInspector = None,
    remote_file_inspector_factory: ?proc(swdev.DeviceMgr) -> RemoteFileInspector = None,
    file_transfer_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer = None,
    cli_session_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> CliSession = None,
    log_handler: ?logging.Handler = None,
) -> proc(ttt.Path, ?ttt.Layer) -> ttt.Node

SOFTWARE_INSTALL_YANG: yang.Module
```

---

## 2. Module boundary — what apps integrate against (🆕 concrete topology per A3)

Apps:

1. **Add the YANG model** to their layer stack. The model augments `/sw-rfs:rfs[name]/` with a `software-pack` presence container.
2. **Wire one `ttt.Transform` per RFS-list entry.**
3. **Optionally provide factories** for `FileTransfer`, `CliSession`, and `RemoteFileInspector`.
4. **Provide a `file.FileCap`** for `LocalFileInspector`.

🆕 **Critical layer topology requirement** (per A3 — the round-4 reviewers flagged this as a footgun if undocumented):

The runner subscribes to `/software-install/...` (the global pack library, `enabled` master switch, error-handling policy) via `params.lower.declare_subscriptions(...)`. **`params.lower` reads the layer BELOW the sw-install layer**, NOT the sw-install layer itself. Therefore:

- **`/software-install/...` MUST be in a layer below the sw-install transform's host layer** for the subscription to see it.
- Concretely: the host layer transforms `/software-install/...` from a layer above (or it's authored in a parent layer) and passes it through to the lower layer.

Example layer topology:

```
                                 northbound config (RESTCONF / file)
                                              │
                                              ▼
        ┌─────────────────────────────────────────────────────────┐
        │  Layer N (top)                                          │
        │  /software-install/...     ← user authors here          │
        │  /sw-rfs:rfs[name]/...     ← user authors here          │
        │                                                         │
        │  No sw-install transform at this layer.                 │
        └─────────────────────────────────────────────────────────┘
                                              │ pass-through both subtrees
                                              ▼
        ┌─────────────────────────────────────────────────────────┐
        │  Layer N-1 (host)                                       │
        │  /software-install/...     ← visible here               │
        │  /sw-rfs:rfs[name]/                                     │
        │      software-pack/        ← swi transform attached     │
        │                                                         │
        │  swi transform's params.lower is Layer N-2.             │
        └─────────────────────────────────────────────────────────┘
                                              │ pass /software-install/ down
                                              ▼
        ┌─────────────────────────────────────────────────────────┐
        │  Layer N-2 (lower of sw-install)                        │
        │  /software-install/...     ← runner subscribes here     │
        └─────────────────────────────────────────────────────────┘
```

If the topology is wrong, the runner publishes `runner-status: missing-global-config` to oper after a 5-second startup window — operators get a fail-loud signal. See §7.2.

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
    # Wire per-app: layer N-1 contains the swi transform; layer N-2
    # carries /software-install/... downward for the runner's subscription
    # to see. See app's t_*.act for the exact composition.
    layer_lower = ttt.Layer('lower', my_lower_t_n2(), None, db)
    layer_host = ttt.Layer('host', my_host_t_n1(swi_factory), layer_lower, db)
    layer_top = ttt.Layer('top', my_top_t_n(), layer_host, db)
    return layer_top
```

---

## 3. Where state lives

### 3.1 The ownership rule (unchanged)

| Surface | Contents | Mutability |
|---------|----------|------------|
| Config (gdata) | desired pack assignment, control triggers | external read/write |
| Dynstate (gdata persisted via `update_dynstate`) | all runner-owned operational state | runner-internal |
| Oper (gdata published via `update_oper`) | pure projection of dynstate plus computed view | external read |
| Transform `memory` | unused | n/a |

### 3.2 Per-device dynstate schema (unchanged from v4)

```acton
class SwInstallDynstate(value):
    last_pack_snapshot: ?SoftwarePack
    last_request_generation: u64
    last_start_generation: u64
    last_cancel_generation: u64
    last_confirm_all_generation: u64
    last_clear_run_log_generation: u64
    next_request_id: u32
    current: ?RequestState
    history: list[RequestState]

class RequestState(value):
    request_id: u32
    pack: SoftwarePack
    confirm_steps: bool
    plan: ComponentPlan
    states: dict[str, State]
    status: RequestStatus
    run_id_count: u64
    run_log: list[RunLogEntry]
    run_log_dropped: u64
    error_count: ErrorCount
    next_wake_at: ?datetime
    generation_token: u64
    auto_started_after_confirm: bool          # 🆕 CR4_5: idempotency anchor for auto-execute
    obsolete: bool
```

### 3.3 Diagnostic projections (unchanged shape; CR4_3/CR4_4 deferral noted)

OS-specific diagnostic leaves under `request/component/` use YANG `when` constraints:
- Common: `destination-volume`, `destination-paths`, `boot-time`.
- SROS-only: `rebooted`.
- IOS-XR-only (Phase 6): `op-id-add`, `op-id-prepare`, `op-id-activate`, `op-id-commit`. **Phase 6 also adds `packages` and `reload-required` per §1's SROS+IOS-XR roadmap; v5 YANG models only the common+SROS subset.**
- Junos per-RE diagnostics (Phase 6): not modeled in v5 YANG; explicit Phase-6 deferral.

### 3.4 Request history retention (unchanged)

Top-level `dynstate.last_pack_snapshot` is the authoritative idempotency baseline. `history` retains: latest of each terminal status + up to 50 entries total.

### 3.5 Restore-consistency check — clarified scope (🆕 CL4_2)

🆕 Claude r4 (H3) caught that the v4 prose described "config newer than dynstate" but the actual check is **dynstate-internal**. v5 corrects the framing.

The runner's startup check (after restoring dynstate from lmdb):

```
if max(dynstate.history.<request_id>, dynstate.current.<request_id>) >= dynstate.next_request_id:
    runner_status = restore-inconsistent
    refuse to materialize new requests
    publish error and wait for operator intervention
```

This catches **dynstate-blob internal inconsistency** (e.g., a partial dynstate restore where `next_request_id` was rolled back relative to history). It does **not** catch cross-cutting config-vs-dynstate mismatch — the dangerous direction (config newer than dynstate, e.g., older dynstate restored against current config) is genuinely undetectable without an independent record outside dynstate.

🆕 v5 §15.5 wording: "v1 design detects only dynstate-internal inconsistency. Cross-cutting backup-restore safety is a v2.0 follow-up requiring an independent record (e.g., a config-side `last-request-id-hint` written periodically by the runtime)."

### 3.6 How the runner receives restored dynstate — primary path (🆕 D5 = was D3b)

🆕 v5 promotes the `transform_wrapper`-stash mechanism to primary. v4's "platform addition + restore at construction" was discovered to be broken: actor construction runs during `Layer(...)` rootgen, BEFORE `Layer.load_from_db()` restores LMDB state — so even with `TransformActorParams.dynstate` added as a field, the value would be `None` at actor construction time.

The stash mechanism:

```acton
class SwInstallTransform(ttt.TransformFunction):
    var stashed_dynstate: ?gdata.Node = None      # set on first transform_wrapper after restore
    var stashed_dynstate_consumed: bool = False

    def transform_wrapper(self, cfg, linked, memory, dynstate):
        # transform_wrapper runs DURING the post-restore recompute,
        # so dynstate here IS the restored value.
        if not self.stashed_dynstate_consumed:
            self.stashed_dynstate = dynstate
        # No downward output; memory unchanged.
        return (gdata.Container(), memory)
```

The runner reads it on its first `on_local_config` call:

```acton
actor DeviceRunner(...):
    # ... fields ...

    var dynstate: SwInstallDynstate = SwInstallDynstate.empty()
    var dynstate_initialized: bool = False

    action def on_local_config(cfg: ?gdata.Node, mem: ?gdata.Node):
        if not dynstate_initialized:
            stashed = self.transform_fn.stashed_dynstate
            if stashed is not None:
                self.dynstate = SwInstallDynstate.from_gdata(stashed)
            self.transform_fn.stashed_dynstate_consumed = True
            self.dynstate_initialized = True
            self._check_restore_consistency()
        # ... reconciliation ...
```

The `transform_fn` reference is the same `SwInstallTransform` instance that the platform threaded through `_TransformTransaction.compute(...)` — both the function and the runner actor close over it (the runner spawns from `init_dynstate`'s `act` callback, which is set up after the function is constructed).

**v2.0 platform ask**: a cleaner `on_restored_dynstate(dynstate)` actor callback that runs after restore but before first `on_conf`. More invasive than the v4 "five-line patch" — requires lifecycle restructuring. Captured in §14 as v2.0; not blocking v1.

⚠️ASSUMPTION: the transform_wrapper-stash pattern works as described. **❓Round-5 question Q1:** verify the `transform_wrapper` post-restore recompute timing matches expectation.

### 3.7 Dynstate-write classification — reformulated (🆕 D6 / A2)

🆕 v5 drops the v4 "Tier A: synchronous await ack" wording. The platform's `update_dynstate` is fire-and-forget with no per-call commit ack — "block until LMDB write completes" is not implementable.

v5 reframes around **idempotent-on-re-fire side effects**:

**Tier A — publish before side effect; design side effect to be safe under re-fire after crash**:
- `last_<trigger>_generation` values — publish before the trigger's work begins. **Re-fire after crash** is safe because the work itself is idempotent: re-creating a request with the same pack returns the same id (§4.1); re-starting an unprocessed request is a no-op (§4.2).
- `next_request_id` — publish before externalizing. Re-fire would assign a new id, but §3.5's restore-consistency check catches dynstate corruption.
- `current.error_count.{transient,other,backoff}`, `current.next_wake_at` — publish before scheduling `after backoff: _start_run`. Re-fire schedules another `after`; the in-memory `after` is lost on restart, so the new schedule is correct.
- `auto_started_after_confirm: True` (🆕 CR4_5) — publish before the auto-start side effect; re-fire on restart sees the flag set and skips re-auto-start.

**Tier B — persist at step boundary** (write at step completion, NOT mid-step):
- `current.plan` (after each step's status transition).
- `current.states[<component>]`.
- `current.run_id_count` (at step boundary, not at run start as a separate event).
- `current.status`, `current.generation_token`.
- `destination_volume`, `destination_paths`, `boot_time`, `rebooted`.
- 🆕 **IOS-XR `op_id_*` (Phase 6)** — moved out of Tier A. Persist after the device returns the op-id. Recovery story for crash-between-issue-and-persist: the runner's next reconciliation calls `o.get_current_install_request()` (matches Python `SoftwareCommit.pre_check`) — the device knows about the in-flight install op even if the runner forgot the op-id.

**Tier C — best-effort telemetry** (in-memory; periodic flush; lost on crash):
- `current.run_log` entries.
- Intra-step polling counters (reset on retry).

### 3.8 `internal-state` deviation (unchanged)

Dropped from YANG; replaced by typed diagnostic projections (§3.3).

---

## 4. Control surface

(Mostly carried from v4; the substantive v5 changes are listed inline.)

### 4.1 `create-request` (unchanged)

### 4.2 `execute-request` ↔ `start-generation` — auto-execute idempotency anchor (🆕 CR4_5)

`unprocessed → processing` requires either:
- `start-generation > dynstate.last_start_generation`, OR
- `auto-execute-after-confirm = true` AND all required confirmations are in place AND `current.auto_started_after_confirm == false`.

🆕 The third clause is the v5 fix per CR4_5: without it, restart could repeatedly auto-start because there's no `start-generation` to consume. v5 sets `auto_started_after_confirm = true` (Tier A) before the auto-start side effect; subsequent restarts see it set and skip.

### 4.3 `confirm-step` ↔ writeable confirmations (🆕 CL4_4 sentinel)

(Unchanged shape from v4.)

🆕 **`confirm-all-generation` `by-user` value:** when triggered by `confirm-all-generation` (no operator-supplied `by-user` in the trigger), the runner stamps `confirmed.by-user = "<confirm-all>"` in the oper projection. Operationally legible; RESTCONF-friendly.

### 4.4 `cancel-request` — drain comparison fix (🆕 CL4_6)

🆕 v5 fixes the `_drain_notify` token comparison: `stale_token < current.generation_token` (was `stale_token + 1 == current.generation_token` in v4). The wider comparison handles drain-from-multi-generations-ago — relevant after cancel-then-restart-then-execute. Watchdog (`CANCEL_DRAIN_TIMEOUT` 600s) remains the backstop.

### 4.5 `clear-run-log` (unchanged)

### 4.6 Per-request scoping + `last-trigger-result` (unchanged from v4)

🆕 **Single-slot `last-trigger-result` is documented as not suitable for high-throughput automation polling** (per CL4_5). Per-call audit lives in the run-log: the runner emits a swi_*-tagged log entry on every trigger consumption. v2.0 enhancement: widen to a small ring of last 5 results.

### 4.7 `enabled` state machine — inter-step gate (🆕 CL4_7 M2)

🆕 v5 §8.3 (re-entrancy invariants) adds invariant 4: **"between steps, the runner re-checks `enabled` and `dynstate.current.status` (cancelling, terminal); if either disqualifies continuation, transitions cooperatively."** §8 run-loop sketch shows this gate.

(Rest of v4 §4.7 carried unchanged.)

### 4.8 YANG diff vs Python (🆕 D7 scp-port placement back to nested)

| Item | v5 plan |
|------|---------|
| (carry v4 entries unchanged unless noted) | |
| 🆕 `scp-port` placement | **Inside** `/sw-rfs:rfs[name]/software-pack/scp-port` (nested, NOT sibling). Per CR4_1 D7: the v4 sibling placement made scp-port invisible to the runner. The "survives unbinding the pack" benefit was speculative. Reverting to nested loses that benefit; gains: the runner can read it directly. |
| 🆕 `runner-status` | New oper enum under `software-pack/`: `{starting, ok, missing-global-config, restore-inconsistent, paused-by-enabled, waiting-for-device}`. |

---

## 5. Typed data model (unchanged)

---

## 6. Plan + step semantics

(Mostly carried.)

### 6.1 `Step` protocol (carried; type formalization unchanged)

### 6.2 Step contract invariants (unchanged)

### 6.3 ComponentPlan invariants (unchanged)

### 6.4 Per-OS step lists (unchanged)

### 6.5 Status mapping at run end (unchanged)

### 6.6 Run-log filter, plumbing, bounded ring (🆕 CL4_10 redaction hook)

🆕 v5 adds to the `RunLogHandler` contract: "`RunLogHandler` skips records with `swi_redacted=True` in their structured-data dict — Phase 5 transcript redaction will use this." The handler check is one line; Phase 5 templates carrying secrets in `Send "${Pass}"` mark their transcript records with `swi_redacted=True` and don't pollute the persistent run-log.

### 6.7 Retry budget (unchanged)

---

## 7. The Transform substrate — plain `ttt.Transform` (🆕 v5: deliberate-departure note + concrete devname helper)

### 7.1 Wiring topology + devname helper (🆕 A5)

The host layer composition wires sw-install as a per-device transform inside `/sw-rfs:rfs`:

```acton
ttt.List(
    ttt.Container({
        q("name"): ttt.Leaf(),
        q("software-pack"): swi_factory,
    }),
    [q("name")],
)
```

🆕 **Deliberate departure from platform convention** (per CL4_3 H5): sorespo's per-device transforms use `RFSTransform` (which gives them `params.dev` automatically). sw-install uses **plain `Transform`** because:

- `_RFSTransaction.finalize` (`ttt.act:2151-2158`) suppresses `on_conf` when output is empty Container. sw-install produces empty downward output (it's an observer); on_conf would be silently lost.
- `RFSFunction.init_dynstate` (`ttt.act:2184-2185`) doesn't pass `lower` through. sw-install needs `params.lower` for the global-config subscription.

Trade-off: the runner doesn't receive `params.dev`. Instead, it extracts the devname from `params.path` and calls `dev_registry.get(devname)` itself.

🆕 **Concrete devname helper** (per A5):

```acton
def devname_from_swi_path(path: ttt.Path) -> str:
    """Extract device name from a sw-install transform's params.path.

    Path shape: <ancestors>/sw-rfs:rfs[name=<dev>]/software-pack
    The devname-bearing key is one level up from 'software-pack'.
    """
    p = path.parent          # the PathKey for /sw-rfs:rfs[name=<dev>]
    if isinstance(p, ttt.PathKey):
        return p.name
    raise ValueError("sw_install: unexpected path shape: {path}")
```

(Cannot reuse `_DeviceTransaction.devname_from_device_path` directly — that helper expects `path: PathKey`, but our `params.path` is a `PathContainer`.)

🆕 v2.0 platform ask: parameterize `_RFSTransaction.finalize`'s empty-output suppression OR thread `lower` through `RFSFunction.init_dynstate`. Either change would let sw-install move to `RFSTransform` and the platform convention realigns. Captured in §14.

### 7.2 Global config subscription + runner-status guard (🆕 D8)

```acton
proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
    devname = devname_from_swi_path(params.path)
    runner = DeviceRunner(
        params.path, params.update_oper, params.update_dynstate,
        dev_registry.get(devname),
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

🆕 **Runner-status startup guard** (per A3 D8): the runner publishes `runner-status` to oper:
- `starting` — initial value, until first `on_local_config` fires.
- After 5s if no global-config data arrived from the subscription → `runner-status = missing-global-config`. Operators see fail-loud.
- After successful first reconciliation with global config seen → `runner-status = ok`.
- §3.5 inconsistency → `runner-status = restore-inconsistent`.
- §4.7 paused via enabled → `runner-status = paused-by-enabled`.
- §8.7 waiting for device adapter → `runner-status = waiting-for-device`.

### 7.3 Transform body (🆕 stashed_dynstate)

```acton
class SwInstallTransform(ttt.TransformFunction):
    var stashed_dynstate: ?gdata.Node = None        # 🆕 D5
    var stashed_dynstate_consumed: bool = False

    def transform_wrapper(self, cfg, linked, memory, dynstate):
        if not self.stashed_dynstate_consumed:
            self.stashed_dynstate = dynstate
        return (gdata.Container(), memory)
```

---

## 8. DeviceRunner architecture

### 8.1 Lease scope (unchanged from v4)

### 8.2 The DeviceRunner actor — type fixes (🆕 CL4_1, CL4_6)

```acton
actor DeviceRunner(
    path: ttt.Path,
    update_oper: proc(?gdata.Node) -> None,             # 🆕 CL4_1: was action(...)
    update_dynstate: proc(?gdata.Node) -> None,         # 🆕 CL4_1: was action(...)
    dev: swdev.DeviceMgr,
    transform_fn: SwInstallTransform,                   # 🆕 D5: for stashed_dynstate
    local_fi: LocalFileInspector,
    remote_fi_factory: proc(swdev.DeviceMgr) -> RemoteFileInspector,
    file_transfer: ?FileTransfer,
    ops_factory: proc(swdev.DeviceMgr, ?CliSession) -> DeviceOps,
    cli_session_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> CliSession,
    log_handler: ?logging.Handler,
):
    var dynstate: SwInstallDynstate = SwInstallDynstate.empty()
    var dynstate_initialized: bool = False
    var global_config_cache: ?GlobalSwInstallConfig = None
    var runner_status: RunnerStatus = STARTING

    action def on_local_config(cfg: ?gdata.Node, mem: ?gdata.Node):
        if not dynstate_initialized:
            self._initialize_dynstate_from_stash()
        # ... idempotent reconciliation ...

    action def _drain_notify(stale_token: u64):
        # 🆕 CL4_6: was '== generation_token - 1'; now '<' to handle multi-gen drains
        if dynstate.current is not None and dynstate.current.status == CANCELLING and stale_token < dynstate.current.generation_token:
            dynstate.current.status = CANCELLED
            self._persist_dynstate(Tier.B)
            self._publish_oper()

    # ... (other helpers) ...
```

### 8.3 Re-entrancy invariants — four rules (🆕 invariant 4 per CL4_7)

1. `on_local_config` and `on_global_config` are pure idempotent reconciliation functions.
2. Generation observations are persisted (Tier A) before any work begins.
3. Tier-classified writes per §3.7.
4. 🆕 **Between steps, the runner re-checks `global_config_cache.enabled` and `dynstate.current.status`** (cancelling, terminal); if either disqualifies continuation, transitions cooperatively (e.g., `processing → paused`). The §8 run-loop body shows this gate.

### 8.4 Restart story (🆕 D5 + oper-startup-window)

On platform startup:
1. Platform restores transform's dynstate from lmdb.
2. Forced post-restore recompute fires `transform_wrapper(cfg, linked, memory, dynstate)`; `transform_fn.stashed_dynstate = dynstate`.
3. The `act` callback fires; runner constructed with default-empty dynstate.
4. First `on_local_config(cfg, mem)` reads `transform_fn.stashed_dynstate`, sets `self.dynstate`, marks initialized.
5. Runs §3.5 consistency check; if inconsistent, `runner_status = restore-inconsistent`.
6. Inspects `dynstate.current.status` and applies recovery rules (processing → failed-transient; cancelling → cancelled; etc., per v4 §8.4).
7. Persists Tier B and publishes oper. **🆕 Oper data is not platform-persisted across restart** (per CL4_7) — clients polling during steps 1-7 see empty oper data and must retry.
8. Backoff resume: if `next_wake_at` future, schedule fresh `after`.

🆕 §15.5 entry: "Oper data is not platform-persisted across stratoweave restart, unlike Python NSO CDB oper. Clients polling during the runner's first reconciliation gap see empty data and must retry."

### 8.5 Cancel implementation (carried v4; CL4_6 fix in §8.2)

### 8.6 Backoff (carried v4; CL3_4 enabled gate carried)

### 8.7 Device-not-yet-ready case — fixed get_modules shape (🆕 CR4_2)

```acton
action def _poll_device_readiness():
    if dynstate.current is not None and dynstate.current.status == WAITING_FOR_DEVICE:
        try:
            modules, _modset_id = dev.get_modules()           # 🆕 CR4_2: tuple shape
            if len(modules) > 0:                              # non-empty modset
                # tighten readiness rule per CR4_2:
                # check the adapter is not NoAdapter (proxy: capabilities non-empty)
                caps = dev.get_capabilities()
                if len(caps) > 0:
                    dynstate.current.status = UNPROCESSED
                    self._persist_dynstate(Tier.B); self._publish_oper()
                    return
        except Exception:
            pass
        after 30: self._poll_device_readiness()
```

🆕 v2.0 platform ask: `DeviceMgr.on_status_change(cb)` event-driven equivalent. Captured in §14.

🆕 **`dev_registry.get(devname)` call shape** (per CL4_8): one-line check during Phase 4 skeleton — verify whether returned `DeviceMgr` is value-typed-synchronous or async-future. If async, `act()` must defer construction to a callback chain. ❓Round-5 question Q2.

---

## 9. Transport scope

### 9.1 (unchanged)

### 9.2-9.4 LocalFileInspector / RemoteFileInspector / FileTransfer (unchanged)

### 9.5 Credential reuse — `DeviceMgr.get_dmc()` (unchanged from v4)

### 9.6 `scp-port` placement — back to nested (🆕 D7)

🆕 v5 reverts to v3's nested placement: `/sw-rfs:rfs[name]/software-pack/scp-port`. The v4 sibling placement was driven by "scp-port should survive pack unbinding" but the v4 reviewers caught the actual cost: with the transform attached at `software-pack`, the runner doesn't see a sibling `scp-port`. The "survives unbinding" benefit was speculative (no concrete non-sw-install consumer exists today).

§15.5 #14 corrected: "scp-port lives inside software-pack/. Removing the software-pack also removes scp-port — but the operator removing the pack is disabling sw-install for the device anyway, so the loss is acceptable."

### 9.7 DeviceOps facade (unchanged)

---

## 10. Testing strategy (unchanged)

---

## 11. Implementation phasing within Phase 4 (🆕 D5 unblocks)

**🆕 v5 has NO platform prerequisites for Phase 4.** Implementation can begin immediately on the existing platform.

(Otherwise unchanged.)

---

## 12. Open decisions (round-5 questions)

| # | Question | Lean |
|---|----------|------|
| **❓Q1** | 🆕 Verify `transform_wrapper` post-restore recompute timing for the stashed_dynstate path (§3.6 D5). | Should work; verify in Phase 4 skeleton. |
| **❓Q2** | `dev_registry.get(devname)` call shape — synchronous return or async future? (§8.7) | Verify in Phase 4 skeleton. |
| **❓Q3** | (was: TransformActorParams.dynstate platform addition) — **resolved** by D5 stash path. | resolved |
| **❓Q4** | (was: layer topology — fresh-integrator footgun) — **resolved** by §2 concrete topology + §7.2 runner-status guard. | resolved |
| **❓Q5** | Run-log default bound. | 1000 entries/request. |
| **❓Q6** | Phase 5 = TextFSMPlus + DeviceOps CLI + FileTransfer infra; Phase 6 = IOS-XR + Junos + polish. | confirmed. |
| **❓Q7** | `CANCEL_DRAIN_TIMEOUT` default. | 600s. |

---

## 13. Implementation details deferred (unchanged)

---

## 14. Platform prerequisites — Phase 4 / v2.0 (🆕 Phase 4 list now empty)

🆕 **Phase 4 prerequisites: NONE.** v5's D5 path uses only existing platform mechanisms.

**v2.0 prerequisites** (sw-install lives without these in v1):

1. **`DeviceMgr.acquire_exclusive(owner_id, timeout, cb)`** — system-wide install lease (closes §8.1 gap).
2. 🆕 **`update_dynstate(node, cb: action() -> None)` ack variant** — would support a true "Tier A synchronous" semantics; v1 uses idempotent-on-re-fire instead (§3.7).
3. 🆕 **`on_restored_dynstate(dynstate)` actor callback or post-restore lifecycle restructuring** — would replace the §3.6 transform_wrapper-stash hack with a clean restore-aware actor init.
4. 🆕 **`Layer` "subscribe to current layer root" API** — would simplify §2 layer-topology constraint (currently the runner subscribes to lower layer; if /software-install/ isn't there, runner-status alerts but a current-layer-root API would remove the constraint entirely).
5. **Config-restore event hook on `Layer`** — auto-reset `last_observed_*` counters (§3.5).
6. **`DeviceMgr.on_status_change(cb)`** — event-driven replacement for §8.7 polling.
7. **`next_request_id` recovery API** — for §3.5 restore-inconsistency recovery.
8. 🆕 **Parameterize `_RFSTransaction.finalize` empty-output suppression OR thread `lower` through `RFSFunction.init_dynstate`** — would let sw-install move to `RFSTransform` and realign with platform convention (§7.1 deliberate-departure note).

---

## 15. Deferred features (unchanged)

## 15.5 Conscious deviations from the Python spec

1. `internal-state` opaque JSON blob dropped; typed leaves under `request/component/` are RESTCONF-visible.
2. Run-log bounded at 1000 entries/request; `dropped-count` surfaced.
3. Cancel takes effect at "next step boundary or RPC return," with explicit `cancelling` enum and 600s drain watchdog.
4. Per-device install lease is sw-install-internal only.
5. 🆕 Generation counters: dynstate-internal restore inconsistency is detected; cross-cutting backup-restore safety is a v2.0 follow-up requiring an independent record outside dynstate.
6. `software-install-matrix` dropped.
7. `vrp` enum kept; no step module.
8. Snabb/ONS-TL1/HGW dropped.
9. Action-style return values replaced by per-device `last-create-result` and single-slot `last-trigger-result` (last-writer-wins; not suitable for high-throughput automation polling — use run-log for per-call audit).
10. 🆕 **CLI strategy not selected in Phase 4**; per-OS `Ops` modules carry a `cli_session: ?CliSession` field but no CLI methods are stubbed. Phase 5 adds CLI methods alongside existing NETCONF ones.
11. `waiting-for-device` request status (no Python equivalent).
12. `paused` request status (no Python equivalent).
13. Run-log `(when, seq)` keying.
14. 🆕 `scp-port` lives at `/sw-rfs:rfs[name]/software-pack/scp-port` (nested). Removing the pack also removes scp-port.
15. Phase 4 `CheckFiles` is a no-op when no `FileTransfer` is configured. 🆕 **`Cleanup` (IOS-XR Phase 6) is also a no-op when `FileTransfer.caps().delete == false`** — parallel deviation.
16. Backoff rounded to ceiling integer seconds when projected to YANG `uint32`.
17. 🆕 **Tier A semantics are "publish-before-side-effect + idempotent-on-re-fire," not "await commit."** Platform's update_dynstate is fire-and-forget. IOS-XR `op_id_*` (Phase 6) are NOT in Tier A; their recovery is via `o.get_current_install_request()` device-side observation.
18. 🆕 **Oper data is not platform-persisted across stratoweave restart**, unlike Python NSO CDB oper. Clients polling during the runner's first-reconciliation gap see empty data and must retry.
19. 🆕 `confirm-all-generation`-driven confirmations stamp `confirmed.by-user = "<confirm-all>"` in oper.

---

## 16. Round-5 review

This v5 design integrates all round-4 feedback. Both reviewers' five HIGH-priority items (lifecycle/D3a-broken, Tier A unimplementable, scp-port sibling visibility, subscription-wiring footgun, action vs proc type) are addressed. v5 unblocks Phase 4 with no platform prerequisites; round-3's "platform addition required" is gone.

**Stop here for round-5 review.** Both reviewers will be re-briefed against the full revised doc set (`00-orientation.md`, `01-software-install-logic.md`, `02-sw-install-design.md` v5, `docs/adr/cli-driver.md`, `yang.act` v5) with no carry-over context.
