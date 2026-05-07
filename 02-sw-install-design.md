# 02 — Stratoweave `software-install` Module Design (v5.3.2 — self-contained)

> **Status: v5.3.2 — locked in after 8 review rounds (HIGH count 6 → 5 → 3+3 → 2+2 → 1+1 → 0+0). v5.3.2 is a content-only readability pass: the 14 sections that earlier revisions had shortened to "(unchanged)" placeholders are now inlined with full prose so the doc stands alone for a fresh reader without git-diving. No semantic changes vs v5.3.1; the architecture, algorithms, and decisions are identical.**

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

If the topology is wrong, the runner publishes `runner-status: missing-global-config` to oper after `MISSING_GLOBAL_CONFIG_GRACE` (default 15s after runner construction) — operators get a fail-loud signal. See §7.2 for the exact algorithm.

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

### 3.1 The ownership rule

| Surface | Contents | Mutability |
|---------|----------|------------|
| Config (gdata) | desired pack assignment, control triggers | external read/write |
| Dynstate (gdata persisted via `update_dynstate`) | all runner-owned operational state | runner-internal |
| Oper (gdata published via `update_oper`) | pure projection of dynstate plus computed view | external read |
| Transform `memory` | unused | n/a |

### 3.2 Per-device dynstate schema

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
    materialized_by_request_generation: u64    # 🆕 v5.1 per CR5/H1: idempotency anchor for request-generation
    plan: ComponentPlan
    states: dict[str, State]
    status: RequestStatus
    run_id_count: u64
    run_log: list[RunLogEntry]
    run_log_dropped: u64
    error_count: ErrorCount
    next_wake_at: ?datetime
    generation_token: u64
    auto_started_after_confirm: bool          # CR4_5: idempotency anchor for auto-execute
    obsolete: bool
```

### 3.3 Diagnostic projections (🆕 v5.2 prose fix per L2)

OS-specific diagnostic leaves under `request/component/` use YANG `when` constraints:
- Common (always present): `destination-volume`, `destination-paths`, `boot-time`.
- SROS-only: `rebooted`.
- IOS-XR-only (modeled in v5+ YANG, populated when Phase 6 lands): `op-id-add`, `op-id-prepare`, `op-id-activate`, `op-id-commit`.
- IOS-XR-only **deferred to Phase 6 YANG additions**: `packages`, `reload-required`.
- Junos per-RE diagnostics: **deferred to Phase 6 YANG additions** (will need a per-RE list under `component/`).

### 3.4 Request history retention

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

### 3.6 How the runner receives restored dynstate — action-ref push (🆕 v5.1 CL5_1)

🆕 v5.1 changes the stash mechanism from "shared mutable field on `SwInstallTransform`" to "action ref pushed at the runner". Round-5 caught that v5's field-stash breaks actor isolation: `SwInstallTransform` is owned by `_TransformTransaction`'s actor (which writes the field via `transform_wrapper`), and the runner actor would read it from a different actor context. v5.1's action-ref push keeps everything inside actor message-passing.

**Lifecycle ordering** (verified against `app.act:138-152` `StartupBootstrap._run`, `ttt.act` 1942-1993, and the `_List`/`ListState` per-list-entry construction path at `ttt.act:1232-1331`):

🆕 **v5.3 prose correction (per round-7 claude r7 MED 1):** the per-list-entry `SwInstallTransform` is constructed when the list entry is materialized — NOT at `Layer(...)` rootgen time. There are two paths:

- **Restored case:** during `Layer.load_from_db()` (`ttt.act:610-643`), the LMDB scan delivers `ATTR_KEYS` records first. Each `ATTR_KEYS` invocation triggers `_List.restore` → `ListState.recreate(key, leaves)` → `template(PathKey(...), lower)` → `_create_transform_node` → `_TransformTransaction.__init__` (which calls `function_factory(log_handler)`, constructing `SwInstallTransform` and appending to `fn_holder`) → `init_dynstate` queued to the Transaction actor → eventually `act(params)` runs and constructs the `DeviceRunner`. **The runner is born for restored entries during `load_from_db`, not at Layer construction.**
- **Fresh case:** when the user's first `edit_config` creates the list entry via `ListState.acquire` (`ttt.act:1316-1331`), `template(...)` is invoked then. The runner is constructed at that moment with empty in-memory dynstate (no LMDB record exists for a fresh entry).

In either case, the runner exists with `dynstate_initialized = False` before `transform_wrapper` first fires for that entry. The full ordering for a restored entry is:

1. `Layer(...)` rootgen runs; the `_List`/`ListState` is constructed but no list entries are materialized yet.
2. `Layer.load_from_db()` begins. ATTR_KEYS records replay; for each key:
   - `_List.restore(key, ...)` → `ListState.recreate(key, leaves)` → `template(PathKey(...), lower)` constructs `SwInstallTransform` (via `function_factory`) and the `DeviceRunner` (via `act`).
   - During `act`, the factory installs `fn.stash_cb = runner._stash_dynstate`.
3. ATTR_DYNSTATE records replay; `_TransformTransaction.restore` populates `self.dynstate`.
4. `app.StartupBootstrap.recompute(force=True)` calls `transform_wrapper(cfg, linked, memory, dynstate)`. The `dynstate` argument is the restored value.
5. `transform_wrapper` invokes `self.stash_cb(dynstate)` — action dispatched to runner's mailbox.
6. `_TransformTransaction.finalize` calls `function.on_conf(self.get(), self.memory)` if `self.running` is non-empty. This is the runner's `on_local_config`.
7. Both `_stash_dynstate(dynstate)` and `on_local_config(cfg, mem)` are now in the runner's mailbox in FIFO order. Runner processes `_stash_dynstate` first (initializes dynstate, runs §3.5 consistency check), then `on_local_config` (begins reconciliation).

**Code shape:**

`SwInstallTransform` (carries the stash callback, no other state):

```acton
class SwInstallTransform(ttt.TransformFunction):
    # 🆕 v5.1 CL5_1: action-ref push, not shared mutable field
    var stash_cb: ?action(?gdata.Node) -> None = None
    var stash_done: bool = False

    def transform_wrapper(self, cfg, linked, memory, dynstate):
        # Runs during the post-restore recompute (StartupBootstrap force=True).
        # dynstate at this point is the LMDB-restored value.
        cb = self.stash_cb
        if cb is not None and not self.stash_done:
            cb(dynstate)              # action call — lands on runner mailbox
            self.stash_done = True
        # CL5_3: return None for memory (memory is unused for sw-install;
        # returning the input unchanged would surprise readers).
        return (gdata.Container(), None)
```

🆕 **v5.2 (per A1): the stash_cb is installed by the `act` callback in §7.3, not by the DeviceRunner actor.** The DeviceRunner has no back-reference to `SwInstallTransform`. The function instance only needs a forward-reference (the `stash_cb` action ref) to the runner; the runner does not need to read or write the function.

`DeviceRunner` (the runner — no transform_fn back-reference):

```acton
actor DeviceRunner(
    ...,
    # NO transform_fn parameter — see §7.3 for how stash_cb is installed.
    ...
):
    var dynstate: SwInstallDynstate = SwInstallDynstate.empty()
    var dynstate_initialized: bool = False

    action def _stash_dynstate(stashed: ?gdata.Node):
        # Called by SwInstallTransform.transform_wrapper during the post-
        # restore recompute. May be called with stashed=None for the empty-
        # restore case (no LMDB record exists) — see §3.6.1.
        if stashed is not None:
            self.dynstate = SwInstallDynstate.from_gdata(stashed)
        self.dynstate_initialized = True
        self._check_restore_consistency()

    action def on_local_config(cfg: ?gdata.Node, mem: ?gdata.Node):
        # 🆕 v5.2 (per A4): committed to FIFO mailbox ordering. _stash_dynstate
        # is enqueued before on_local_config in the post-restore path
        # (transform_wrapper runs first, then on_conf via finalize). Acton's
        # actor mailbox is FIFO, so dynstate_initialized is guaranteed True
        # by the time on_local_config processes — no fallback needed.
        # The empty-restore path uses _stash_dynstate(None) per §3.6.1.
        # ... reconciliation ...
```

### 3.6.1 Empty-restore signal (🆕 v5.2 per A4)

When the platform restores no dynstate (fresh install or first boot), `_TransformTransaction.dynstate` is `None`. `transform_wrapper` is still called during the forced recompute (with `dynstate=None`), and the stash mechanism passes `None` into `_stash_dynstate(stashed=None)`. The runner treats this as "fresh install — empty dynstate is correct" and sets `dynstate_initialized = True` without restoring.

This means `_stash_dynstate` is **always called** before any `on_local_config`, regardless of whether dynstate was actually restored. FIFO mailbox ordering then guarantees `dynstate_initialized == True` when `on_local_config` processes — no race, no fallback needed.

🆕 **Note on integration A4 deviation (v5.3 per round-7 LOW 1):** the round-6 integration recommended a separate `_signal_no_restore()` action for the empty-restore case. v5.2 instead overloads `_stash_dynstate(stashed=None)` for both restored-but-empty and never-persisted cases. The two are functionally indistinguishable to the runner (both lead to `SwInstallDynstate.empty()`), and a single entry point eliminates any risk of the two signals racing past each other in the mailbox. The simpler form is correct for v5.2's use case.

**Caveat (per claude r5 §A1):** if `_TransformTransaction.finalize` skips `on_conf` because `self.running` is empty (no `software-pack` has ever been authored for this device), the runner stays in `runner-status = starting` indefinitely. This is the no-software-pack-bound steady-state — benign (there's nothing to do). Operators should treat `starting` plus an empty `request[]` list as "no work configured" rather than "failure to initialize." Documented in §7.2.

**v2.0 platform ask**: a cleaner `on_restored_dynstate(dynstate)` actor callback that runs after restore but before first `on_conf`. More invasive than the v4 "five-line patch" — requires lifecycle restructuring. Captured in §14 as v2.0; not blocking.

### 3.7 Dynstate-write classification — reformulated (🆕 D6 / A2; v5.1 batching invariant)

🆕 v5 dropped the v4 "Tier A: synchronous await ack" wording. The platform's `update_dynstate` is fire-and-forget with no per-call commit ack — "block until LMDB write completes" is not implementable.

🆕 **v5.1 critical invariant** (per round-5 H1+H2): **Tier A field updates are always batched into the same `update_dynstate(...)` snapshot as the consequences they trigger.** The "Tier" classification governs **when** `update_dynstate` is called; what each call **contains** is the entire current `dynstate` snapshot — including any in-memory consequences of the trigger. Concretely: when a trigger fires, the runner mutates the trigger marker AND any consequent state (new request materialized, status flipped, etc.) in `self.dynstate` together, then calls `update_dynstate(self.dynstate.to_gdata())` once. A crash before that single call leaves no marker; a crash after leaves a fully-consistent snapshot. Re-fire after crash is then safe per the rules below.

**Tier A — publish before side effect; design side effect to be safe under re-fire after crash**:

- `last_<trigger>_generation` values — publish before the trigger's work begins. **Re-fire after crash** is safe per the trigger:
  - `last_start_generation` — re-fire of `start-generation` against an `unprocessed` request is a no-op (§4.2). Against a `processing` request the runner is already running. Safe.
  - `last_cancel_generation` — re-fire transitions `processing → cancelling` (idempotent — already in cancelling has no further effect). Safe.
  - `last_clear_run_log_generation` — re-fire empties an already-empty log. Safe.
  - `last_confirm_all_generation` — re-fire stamps the same confirmations. Safe.
  - 🆕 `last_request_generation` — **NOT idempotent by design** (request-generation explicitly forces a new request even on identical pack data). v5.1 adds an explicit anchor (see below).
- 🆕 **`current.materialized_by_request_generation: u64`** (per round-5 codex H1; **v5.3 wording correction per round-8 reviewers**) — when a new request is materialized in response to `request-generation`, this field on `RequestState` records which generation triggered it. The §4.1 reconciliation rule (verbatim): "if `current.materialized_by_request_generation == cfg_request_gen` AND `target_pack == last_pack_snapshot`, the current request already corresponds to this trigger and pack snapshot — no-op; otherwise allow Trigger A or Trigger B (pack-change) to materialize." Both conditions are required: a same-generation-but-changed-pack situation must NOT short-circuit, because the operator's pack-data change is a legitimate reason to materialize a new request even without bumping `request-generation`. This makes `request-generation` re-fire-safe across crash+restart while preserving Trigger B for pack-data changes.
- `next_request_id` — publish in the same snapshot as the new request being added to history. Re-fire would have already had the new request visible.
- `current.error_count.{transient,other,backoff}`, `current.next_wake_at` — publish before scheduling `after backoff: _start_run`. Re-fire schedules another `after`; the in-memory `after` is lost on restart, so the new schedule is correct.
- `auto_started_after_confirm: True` (🆕 CR4_5) — publish in the same snapshot as `current.status = processing` (per the v5.1 batching invariant above). A crash either leaves both written or both unwritten; re-fire on restart sees the snapshot consistently and proceeds correctly.

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

### 3.8 `internal-state` deviation

The Python `software-install` package serializes the per-component `State` object as `jsonpickle` JSON in the `request/component/internal-state` YANG leaf. This is dropped from the v5+ YANG; the operationally-useful fields are now surfaced as typed leaves under `request/component/` (see §3.3 diagnostic projections — `destination-volume`, `destination-paths`, `boot-time`, `rebooted`, IOS-XR `op-id-*`, etc.). Trade-off: better for typed inspection (RESTCONF clients see schema-defined leaves), worse for "show me everything via one CLI command" (the opaque blob is gone). Captured as conscious deviation §15.5 #1.

---

## 4. Control surface

(Mostly carried from v4; the substantive v5 changes are listed inline.)

### 4.1 `create-request` reconciliation (🆕 v5.2 per round-6 A3 / MED 1)

Round-6 reviewers caught that v5.1 left this section as "(unchanged)" — but the §3.7 Tier-A invariant references "the §4.1 reconciliation rule" that consumes the new `materialized_by_request_generation` anchor. v5.2 spells out the rule.

**Three independent triggers** for materializing a new request, in precedence order:

```acton
action def _reconcile_create_request(local_cfg: gdata.Node, global_cfg: GlobalSwInstallConfig):
    # Inputs:
    #   local_cfg — /sw-rfs:rfs[name=X]/software-pack/ subtree (config)
    #   global_cfg — /software-install/ subtree (from subscription)
    pack_name = local_cfg.get_leaf("name")
    if pack_name is None:
        # No pack assignment — nothing to materialize.
        return

    target_pack = global_cfg.find_software_pack(pack_name)
    if target_pack is None:
        # Pack assignment refers to undefined pack — defer until library has it.
        return

    cfg_request_gen = local_cfg.get_leaf("control/request-generation") or 0

    # ── Trigger A: explicit request-generation increment ──
    if cfg_request_gen > self.dynstate.last_request_generation:
        # 🆕 v5.3 (per round-7 codex H1): the idempotency short-circuit must
        # ALSO require pack equality to fire — otherwise a pack change with
        # the same request-generation gets suppressed. Crash-recovery anchor
        # only applies when both the explicit-generation marker AND the
        # pack data match; if either has moved, the trigger is genuinely new.
        if (self.dynstate.current is not None and
            self.dynstate.current.materialized_by_request_generation == cfg_request_gen and
            target_pack == self.dynstate.last_pack_snapshot):
            # Crash-recovery short-circuit: the current request was already
            # materialized for this generation against this exact pack.
            # No-op. Persists across restart because both fields are in
            # dynstate.
            return
        self._materialize_new_request(target_pack, materialized_by_gen=cfg_request_gen,
                                      cfg_local=local_cfg, global_cfg=global_cfg,
                                      reason="request-generation")
        # The materialization Tier-A-batches:
        #   dynstate.last_request_generation = cfg_request_gen
        #   dynstate.last_pack_snapshot = target_pack
        #   dynstate.current = NewRequestState(..., materialized_by_request_generation=cfg_request_gen, ...)
        #   prior dynstate.current → moved to history with obsolete=True
        # All in one update_dynstate snapshot per §3.7 invariant.
        return

    # ── Trigger B: pack-data changed since last snapshot ──
    # Reaches here when cfg_request_gen <= last_request_generation (no
    # explicit-generation trigger). A pack change is still a valid trigger.
    if target_pack != self.dynstate.last_pack_snapshot:
        self._materialize_new_request(target_pack, materialized_by_gen=self.dynstate.last_request_generation,
                                      cfg_local=local_cfg, global_cfg=global_cfg,
                                      reason="pack-change")
        # `materialized_by_gen` is set to the current observed generation
        # value (NOT incremented), preserving idempotency for any future
        # "same generation, same pack" reconciliation.
        return

    # ── Trigger C: last request was cancelled, even if pack-data unchanged ──
    if (self.dynstate.current is not None and
        self.dynstate.current.status == CANCELLED and
        cfg_request_gen == self.dynstate.last_request_generation):
        # Pack-change rule (B) didn't fire because pack hasn't changed.
        # Cancelled-forces-new requires explicit operator intent: bump
        # request-generation. Nothing to do here. (Documented in §15.5 #5.)
        return

    # No trigger fired — current state is consistent with config; idle.
```

**Precedence:** Trigger A (request-generation) wins over Trigger B (pack-change) when both fire simultaneously — the materialized request records the explicit generation, so subsequent crash-restart short-circuits cleanly via the idempotency check.

**`materialized_by_request_generation` semantics per path:**
- **Trigger A (request-generation):** stored value = the observed `cfg_request_gen` (greater than `last_request_generation`). The idempotency check above uses this to suppress duplicate materialization across crashes.
- **Trigger B (pack-change):** stored value = the current `last_request_generation` (no increment). Pack-change-driven materialization isn't tied to a specific generation; the field still anchors the "I already materialized for this state" check, just not against an explicit operator trigger.
- **Trigger C (cancelled-forces-new):** does NOT fire on its own. Operator must bump `request-generation` to reactivate after cancellation; that becomes Trigger A.

**Crash recovery example:**
1. Operator sets `request-generation=42` (was 41).
2. Runner reconciles: `cfg_request_gen (42) > last_request_generation (41)` → Trigger A fires.
3. Runner mutates dynstate in-memory: `last_request_generation=42`, `current = new RequestState(materialized_by_request_generation=42, ...)`. Calls `update_dynstate(...)` per Tier-A invariant.
4. Crash before LMDB commit completes.
5. Restart: dynstate restored to either pre-step-3 state (`last_request_generation=41`, no current update) or post-step-3 state (`last_request_generation=42`, current.materialized_by_request_generation=42). Atomic — no in-between.
6. Next reconciliation: if pre-step-3 state, Trigger A fires again — re-materializes the same request (idempotent because the operator's intent is unchanged). If post-step-3 state, the idempotency check at the top short-circuits — no-op.

This is the round-6 codex H1 / claude r5 H1 idempotency story made concrete.

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

### 4.5 `clear-run-log` ↔ `clear-run-log-generation`

**Trigger:** `clear-run-log-generation > dynstate.last_clear_run_log_generation`.

**Behavior:** drop all `run-log[]` entries for the targeted request (per §4.6 scoping), reset `run_log_dropped` to 0, reset the per-request `seq` counter to 0. Run-log `(when, seq)` keying guarantees no collision against pre-clear entries because the ring is empty post-clear.

Complements (does not replace) the bounded ring buffer (§6.6).

### 4.6 Per-request scoping + `last-trigger-result`

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

## 5. Typed data model

Two layers:

1. **`gen_adata`-generated typed accessors** for the YANG (`model.act` after build) — driven by the `software_install` YANG module the design ships in `yang.act`.
2. **Internal value types** in `pack.act` / `state.act` tuned for state-machine use (hashable as map keys, deep equality, immutability under value-typed semantics).

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
    def name(self) -> str: ...           # "base-<version>" or "patch-<version>"

class SoftwarePack(value):
    name: str
    os: SoftwarePackOs
    base: ?SoftwarePackComponent
    patches: list[SoftwarePackComponent]
    def components(self) -> list[SoftwarePackComponent]: ...
```

### 5.2 State types (`state.act`)

Per-OS State subclasses each implement `reset()` per logic-spec §5.1 (the "device drifted, restart from scratch" lever, called from `CheckVersions` when the previously-Done version is no longer running on the device):

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
    # restart_prepare_clean / restart_prepare / restart_activate per logic-spec §5.3

class RouteEngine(value):
    base: GenericDevice
    version: ?str
    rebooted: bool

class StateJunos(value):
    head: State
    dual_re: ?bool                       # cross-run invariant — see §6.3
    switch: bool
    failover_config: ?value
    route_engine: dict[str, RouteEngine]
    route_engine_priority: list[int]
    def reset(self) -> StateJunos: ...
```

`reset()` returns a new value (Acton value-typed semantics) rather than mutating; callers replace the State binding with the returned one. Per-component State is stored in `RequestState.states[<component_name>]` (dynstate; see §3.2).

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
    re_id: ?str        # None for non-Junos and for the trailing Done

# Step callback signature:
#   cb(result: StepResult, new_state: ?State, exc: ?Exception)
#
# `new_state = None` means "no state change for this step" (rare; most
# steps return an updated state even on SKIP_STEP because they may have
# observed something worth recording). `exc` is non-None iff the step
# body raised; the runner logs the traceback and treats as FAILURE
# regardless of the StepResult value.

class Step(object):
    key: StepKey
    proc def pre_check(self, state, ops: DeviceOps,
                       lfi: LocalFileInspector, rfi: RemoteFileInspector, ft: FileTransfer,
                       step_log: StepLogger,                                       # per-step logger (§6.6)
                       cb: action(StepResult, ?State, ?Exception) -> None) -> None: ...
    proc def execute(self, state, ops: DeviceOps,
                     lfi, rfi, ft, step_log,
                     cb: action(StepResult, ?State, ?Exception) -> None) -> None: ...
    def next_step(self, state) -> ?StepKey: ...
    def supports_pre_check(self) -> bool: ...
```

Step methods receive **six** parameters: `state`, `ops`, `lfi`, `rfi`, `ft`, `step_log`. Plus the `cb` callback.

- `state` — per-component value-typed State (per §5.2 / `state.act`); the step returns a possibly-updated State via `cb`.
- `ops: DeviceOps` — per-OS facade for NETCONF/CLI device operations (§9.7).
- `lfi: LocalFileInspector` — controller-side filesystem checks (§9.2).
- `rfi: RemoteFileInspector` — device-side file metadata via NETCONF (§9.3).
- `ft: FileTransfer` — byte-mover (Phase 5; `NoopFileTransfer` in Phase 4).
- `step_log: StepLogger` — pre-bound to the step's `swi_request_path/swi_component/swi_step/swi_run_id` so step authors don't need to remember the keys (§6.6).

State is value-typed and threaded through the callback. The runner persists the new state only when `result != FAILURE` (§6.3 flush ordering).

### 6.2 Step contract invariants

- **Steps are ordinary classes**, not actors. The runner constructs `cb` as an `action def` defined on itself; closing over `self` makes the callback dispatch on the runner mailbox automatically. Steps that need helper actors must terminate them before invoking `cb`.
- **Callback mailbox.** The `cb` passed to a step always dispatches on the per-device DeviceRunner mailbox. This is what makes the §8 generation-token check effective — stale callbacks land on the runner, not the step's own actor.
- **`next_step` jump-target validation.** If `next_step(state)` returns a `StepKey` not present in the current plan, the runner emits a clear log entry and returns `FAILURE` for the current step. (Mirrors Python `refresh_steps`'s regression guard.)
- **Failure isolation.** A step's exception surfaces as `(FAILURE, ?State, exc)` from the callback. The runner logs the traceback and proceeds with FAILURE handling — exceptions never propagate up the actor.
- **`exc` convention.** `exc` is non-None iff the step body raised. Runner logs traceback and treats as FAILURE regardless of the StepResult value.

### 6.3 `ComponentPlan` invariants

- **Refresh discipline (A8 from round-2):** `refresh_steps` runs **after every step's `_execute_step_action`**, not only at run start. This is what enables IOS-XR FPDs to be discovered mid-run (the FPD list only becomes known after `SoftwarePrepare`).
- **Monotonicity (A8):** the refresh may add steps but must not remove prior components or steps. A removal indicates a logic bug; the runner raises.
- **Flush ordering (CL8 from round-2):**
  1. Step's `pre_check` or `execute` returns `(StepResult, NewState)`.
  2. If `result != FAILURE`: persist NewState into the per-component State store (Tier B dynstate write — §3.7).
  3. If `result == FAILURE`: discard NewState; mark step `failed`; mark all subsequent steps in the component back to `not-reached` (or `waiting-confirmation` if confirm-mode).
  4. Refresh the plan.
  5. Persist the plan (Tier B dynstate write).

A FAILURE result must NOT persist the half-mutated NewState — this is a load-bearing invariant for crash safety.

### 6.4 Per-OS step lists

- **SROS** (`platform_sros.act`, Phase 4): step list per logic-spec §6.1. `ValidatePlatform → CheckFiles → CheckVersions → ActivatePrimary → GetBootTime → CopyImage → PrepareCopyBootLdr → PrepareConfigureBof → PrepareHackFormatStandby → PrepareSaveRollback → PrepareSynchronizeBootenv → Reboot → Done`. **No `Cleanup` step** — that's IOS-XR-only (logic-spec §6.2).
- **IOS-XR** (`platform_iosxr.act`, Phase 6): step list per logic-spec §6.2. **`Cleanup` is IOS-XR-only.** `CheckVersions` requires controller-side archive parsing (`.iso` via `get_version_from_iso`, `.tar` via `get_file_packages`) — Phase 6 dependency on iso/tar parsers in Acton.
- **Junos** (`platform_junos.act`, Phase 6): per logic-spec §6.3. **`StepKey(name, re_id=None)` for the trailing unparameterized `Done`** (CL12). **Cross-run invariant (CL10):** if `state.dual_re` changes between runs of the same request, `ValidatePlatform` returns FAILURE and invalidates the request.
- **VRP**: enum kept; `ValidatePlatform` fails cleanly with "unsupported platform". No step module wired.

### 6.5 Status mapping at run end

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

Counters are **consecutive**; FAILURE resets `transient`, WAIT resets `other`. DONE clears `error_count` entirely (including `error_count.backoff`).

### 6.6 Run-log filter, plumbing, bounded ring (🆕 CL4_10 redaction hook)

🆕 v5 adds to the `RunLogHandler` contract: "`RunLogHandler` skips records with `swi_redacted=True` in their structured-data dict — Phase 5 transcript redaction will use this." The handler check is one line; Phase 5 templates carrying secrets in `Send "${Pass}"` mark their transcript records with `swi_redacted=True` and don't pollute the persistent run-log.

### 6.7 Retry budget

Per-class budget (matches the Python spec exactly):

- After a WAIT terminal: if `error_count.transient > config.max_retries` → terminate as `FAILED_TRANSIENT`.
- After a FAILURE terminal: if `error_count.other > config.max_retries` → terminate as `FAILED_OTHER`.

**Backoff formula** (CR2 from round-4): `backoff = (error_count.backoff or 10) * factor` (factor mode) — exactly the Python spec, not `factor * max(...)`.

**Backoff rounding** (CR3_4 from round-3): internally compute as decimal; round to `ceil(seconds)` when persisting to `error_count.backoff` (uint32) and projecting `next_wake_at`. The fractional sequence `(10.0, 12.0, 14.4, 17.28, ...)` becomes `(10, 12, 15, 18, ...)` after ceiling.

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

🆕 **Concrete devname helper** (v5.1 hardened per CR5_1):

```acton
def devname_from_swi_path(path: ttt.Path) -> str:
    """Extract device name from a sw-install transform's params.path.

    Expected path shape: <ancestors>/sw-rfs:rfs[name=<dev>]/software-pack
    where 'software-pack' is a PathElem and its parent is a PathKey
    representing the /sw-rfs:rfs list entry.

    Hardened (v5.1): walks ancestors and verifies the PathKey's parent
    is a PathElem named 'rfs' in the stratoweave-rfs namespace. Refuses
    ambiguous shapes rather than silently returning the wrong key.
    """
    # 'software-pack' itself is a PathElem; its parent should be the PathKey
    # for the RFS list entry.
    if not isinstance(path, ttt.PathElem):
        raise ValueError("sw_install: expected PathElem, got {type(path)}: {path}")
    p_key = path.parent
    if not isinstance(p_key, ttt.PathKey):
        raise ValueError("sw_install: expected PathKey above software-pack, got {type(p_key)}: {path}")
    # Verify the PathKey's parent is the /sw-rfs:rfs list element.
    p_elem = p_key.parent
    if not (isinstance(p_elem, ttt.PathElem)
            and p_elem.name == "rfs"
            and p_elem.namespace() == NS_STRATOWEAVE_RFS):
        raise ValueError("sw_install: expected /sw-rfs:rfs above the device key, got {p_elem}: {path}")
    return p_key.name
```

(The closer-shaped existing helper is `devname_from_rfs_path` at `ttt.act:2042-2053`, which already walks ancestors looking for a `PathElem("rfs")`. We can't reuse it directly because (a) it lacks the namespace check our hardened version adds, and (b) it isn't part of the public substrate API. The structural pattern — walk ancestors, verify parent — is the same.)

🆕 v2.0 platform ask: parameterize `_RFSTransaction.finalize`'s empty-output suppression OR thread `lower` through `RFSFunction.init_dynstate`. Either change would let sw-install move to `RFSTransform` and the platform convention realigns. Captured in §14.

### 7.2 Global config subscription + runner-status guard (🆕 D8; v5.1 fixes)

See §7.3 below for the full `act` callback shape (closure-shared with `function_factory` to access the same `SwInstallTransform` instance for installing `stash_cb`). The §7.3 form is the canonical construction site; this section focuses on the global-config subscription + watchdog wiring that lives inside it.

The relevant parts of `act` for §7.2 are:

```acton
proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
    fn = fn_holder[0]                     # SwInstallTransform from §7.3 closure
    devname = devname_from_swi_path(params.path)
    runner = DeviceRunner(...)            # see §8.2 for the full constructor signature
    fn.stash_cb = runner._stash_dynstate  # 🆕 v5.2 per A1: same-actor write
    if params.lower is not None:
        params.lower.declare_subscriptions(
            owner_id="sw_install:" + devname,
            cb=runner.on_global_config,
            want={SubscriptionSpec(filt=SOFTWARE_INSTALL_FILTER, period=5.0)},  # period pinned to 5s
        )
    # The runner schedules its own missing-global-config watchdog at
    # construction time per §7.2 algorithm below.
    return lambda cfg, mem: runner.on_local_config(cfg, mem)
```

🆕 **Runner-status startup guard — concrete algorithm** (v5.2 per round-6 A2):

Round-6 reviewers caught that v5.1's three descriptions (§7.2 prose / state diagram / YANG) were mutually inconsistent. Worse, "first non-None on_global_config" cannot detect "missing forever" because `_owner_publish` doesn't fire the cb at all when `merge_subscription_tree` returns `None` (verified vs `ttt.act:436-445, 663-672`).

v5.2 algorithm — single source of truth, mirrored verbatim in the YANG `runner-status` description:

```acton
# Constants
MISSING_GLOBAL_CONFIG_GRACE = 15.0    # seconds; >= subscription period (5s) + load_from_db budget

# At runner construction (inside act callback):
after MISSING_GLOBAL_CONFIG_GRACE: self._check_global_config_seen()

action def on_global_config(merged: ?gdata.Node, err: ?Exception):
    # Layer.declare_subscriptions's _owner_publish (ttt.act:663-672) fires
    # the cb when EITHER err is not None OR merged != last_merged. The
    # latter cannot transition out of None == None, which is why a wrong
    # topology (where /software-install/ is absent from the lower layer
    # forever) yields no cb invocation at all — the watchdog below is the
    # only mechanism that catches that case. Errors do fire the cb; we
    # ignore err here because the watchdog is a sufficient backstop and
    # transient subscription errors during startup are noise.
    if merged is not None and contains_software_install_subtree(merged):
        self.global_config_cache = parse_global_config(merged)
        self.global_config_seen = True
        if self.runner_status in {STARTING, MISSING_GLOBAL_CONFIG}:
            self.runner_status = OK
            self._publish_oper()

action def _check_global_config_seen():
    # Watchdog fires unconditionally MISSING_GLOBAL_CONFIG_GRACE seconds
    # after construction. If global_config_seen is still False, topology is
    # wrong (or no /software-install/ has been authored yet) — fail loud.
    if not self.global_config_seen and self.runner_status == STARTING:
        self.runner_status = MISSING_GLOBAL_CONFIG
        self._publish_oper()

def contains_software_install_subtree(merged: gdata.Node) -> bool:
    # True if merged's tree includes the /software-install/ container at
    # any depth (even if empty — operator authored the subtree but no packs
    # yet). False if /software-install/ is genuinely absent (topology wrong).
    ...
```

**Key correctness properties:**
- The watchdog **always fires** (not gated on a callback that may never come) — catches "missing forever" topology errors.
- An empty `/software-install/` container counts as "seen"; the watchdog only complains when the container is genuinely absent from the tree.
- `global_config_seen` flips True permanently on first valid observation; subsequent observations cannot regress it.

State transitions:

```
starting (T=0)   actor constructed; watchdog scheduled at T+15s
   │
   ├─ T<15s, on_global_config delivers /software-install/ → ok
   │
   └─ T=15s, global_config_seen still False → missing-global-config
       │
       └─ later on_global_config delivers /software-install/ → ok

ok ──(§3.5 dynstate-internal inconsistency on startup)──▶ restore-inconsistent
ok / processing ──(/software-install/enabled = false)──▶ paused-by-enabled
ok ──(DeviceMgr returns NoAdapter — no DMC set)──▶ waiting-for-device
```

**Precedence rules** (per CR5_2): when multiple non-`ok` states apply simultaneously, status is set to the highest-priority one:

```
restore-inconsistent  (highest — refuses new requests)
> missing-global-config
> waiting-for-device
> paused-by-enabled
> starting
> ok                  (lowest — only when nothing's wrong)
```

🆕 **Steady state with no software-pack bound** (per CL5_5): if the operator has not bound any `software-pack` to the device, `_TransformTransaction.finalize` skips `on_conf` (because `self.running` is empty), so the runner never receives `on_local_config`. The runner stays at `starting` indefinitely. **This is benign** (there's nothing to do); operators should treat `starting` plus an empty `request[]` list as "no work configured" rather than "failure to initialize." Documented in §15.5.

**v2.0 platform ask** (strengthened per round-5): `Layer` "subscribe to current layer root" API would close the subscription-vs-period race entirely — the host could subscribe to the same layer the transform sits in, and the data is guaranteed present after recompute. Captured in §14 — and per round-5 this is a correctness improvement, not just ergonomics.

### 7.3 Transform body and factory skeleton (🆕 v5.1: action-ref stash + factory)

The `SwInstallTransform` class (carries the action-ref push for dynstate stash):

```acton
class SwInstallTransform(ttt.TransformFunction):
    # 🆕 v5.1 CL5_1: action-ref push, not field stash. Installed by the runner
    # at construction time (before transform_wrapper can fire).
    var stash_cb: ?action(?gdata.Node) -> None = None
    var stash_done: bool = False

    def transform_wrapper(self, cfg, linked, memory, dynstate):
        cb = self.stash_cb
        if cb is not None and not self.stash_done:
            cb(dynstate)
            self.stash_done = True
        # 🆕 v5.1 CL5_3: return None for memory (memory is unused for sw-install).
        return (gdata.Container(), None)
```

🆕 **Concrete `make_sw_install_transform` factory body** (v5.1 per CL5_2). The skeleton shows how the function-factory and the act callback share the `SwInstallTransform` instance via a closure-captured holder — this is the key shape Phase 4 implementation must follow:

```acton
def make_sw_install_transform(
    dev_registry: swdev.DeviceRegistry,
    file_cap: file.FileCap,
    log_handler: ?logging.Handler = None,
    ...
) -> proc(ttt.Path, ?ttt.Layer) -> ttt.Node:

    proc def factory(path: ttt.Path, lower: ?ttt.Layer) -> ttt.Node:
        # Per-device-transform-instance holder — function and act share it.
        # `list[T]` is used as a single-cell mutable container because Acton
        # has no first-class `Cell[T]`/`mut[T]` and tuples are immutable, so
        # closures need a mutable list to share the constructed instance
        # between function_factory and act.
        fn_holder: list[SwInstallTransform] = []

        def function_factory(log_handler: ?logging.Handler) -> SwInstallTransform:
            fn = SwInstallTransform(log_handler=log_handler)
            fn_holder.append(fn)
            return fn

        proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
            fn = fn_holder[0]                          # the same instance used by transform_wrapper
            devname = devname_from_swi_path(params.path)
            dev = dev_registry.get(devname)
            # 🆕 v5.3.1 per round-8 LOW 1: file_transfer_factory's signature is
            # proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer (per §1
            # public API). DMC comes from the existing DeviceMgr.get_dmc()
            # API (§9.5; verified at device.act:401). Read DMC once at
            # FileTransfer construction; if the FileTransfer impl needs fresh
            # creds per transfer (DMC is mutable via repeated set_dmc(...)),
            # it should call dev.get_dmc() inside its put/delete methods —
            # see §9.5 for the use-time-not-construction-time discipline.
            ft = file_transfer_factory(dev, dev.get_dmc()) if file_transfer_factory is not None else NoopFileTransfer()
            runner = DeviceRunner(
                params.path,
                params.update_oper,                    # proc(?gdata.Node) -> None
                params.update_dynstate,                # proc(?gdata.Node) -> None
                dev,
                # Remaining args closed over from make_sw_install_transform's
                # outer scope, passed positionally per §8.2's DeviceRunner sig.
                local_file_inspector or default_local_fi(file_cap),
                remote_file_inspector_factory or default_remote_fi_factory,
                ft,
                ops_factory_for(devname),              # selected by device OS, see §9.7
                cli_session_factory,
                log_handler,
            )
            # 🆕 v5.2 per A1: single-owner stash_cb install. The act callback
            # runs in the same actor that owns `fn` (the Transaction actor),
            # so this is a same-actor write — no cross-actor field mutation.
            # `runner._stash_dynstate` is an action ref, sendable across actor
            # boundaries; transform_wrapper later invokes it from the same
            # Transaction actor (queues to runner mailbox).
            fn.stash_cb = runner._stash_dynstate
            if params.lower is not None:
                params.lower.declare_subscriptions(
                    owner_id="sw_install:" + devname,
                    cb=runner.on_global_config,
                    want={SubscriptionSpec(filt=SOFTWARE_INSTALL_FILTER, period=5.0)},
                )
            return lambda cfg, mem: runner.on_local_config(cfg, mem)

        return ttt.Transform(function_factory, act=act, log_handler=log_handler)(path, lower)

    return factory
```

⚠️ALERT (CL5_6 platform note): `ttt.Transform`'s `lower=` keyword arg is dead code in the current platform — it's shadowed by the `_create_transform_node(path, lower)` parameter and never actually used. Sw-install doesn't pass it; if a future maintainer adds it, it'll silently no-op. Captured in §14.

---

## 8. DeviceRunner architecture

### 8.1 Lease scope — honest downgrade

The Python `MaapiLocker.lock_partial(/devices/device[name=X])` was a **system-wide** mutex over the device subtree: every other writer blocked while the install was in flight. The Acton `DeviceRunner` does **not** provide an equivalent guarantee. While a sw-install run is active:

- An RFS transform may push config via `DeviceMgr.configure(...)`.
- A monitoring transform may issue `rpc_xml` against the same adapter.
- Subscriptions continue to deliver oper updates.

**This is a real safety gap for OS upgrades.** The Acton `DeviceMgr` does not currently expose an exclusive-operation API; adding `DeviceMgr.acquire_exclusive(owner_id, timeout)` is a platform-side change outside sw-install's scope.

**v1 contract:** sw-install serializes its **own** runs per device. Operators must ensure no RFS layer is actively reconciling against a device under upgrade — typically by gating upstream config or by understanding that the install will likely race with normal reconciliation.

This is documented prominently in:

- The README ("Important safety note").
- §15.5 conscious deviations.
- The runtime log at request start ("warning: sw-install does not preempt other DeviceMgr writers").

**Platform prerequisite for v2.0:** see §14 item 1.

### 8.2 The DeviceRunner actor — type fixes (🆕 CL4_1, CL4_6)

```acton
actor DeviceRunner(
    path: ttt.Path,
    update_oper: proc(?gdata.Node) -> None,             # 🆕 CL4_1: was action(...)
    update_dynstate: proc(?gdata.Node) -> None,         # 🆕 CL4_1: was action(...)
    dev: swdev.DeviceMgr,                               # 🆕 v5.2 per A1: no transform_fn parameter
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

### 8.4 Restart story (🆕 v5.1 corrected lifecycle ordering per round-5 A3)

On platform startup, the actual ordering (verified against `app.act:138-152` and the per-list-entry construction path at `ttt.act:1232-1331`):

1. **Layer rootgen** — `Layer(...)` calls `rootgen(...)`. For sw-install (which lives inside a `ttt.List`), this constructs the `_List` / `ListState` but does NOT yet materialize any list entries — `SwInstallTransform` and `DeviceRunner` do not exist for any specific device until step 2.
2. **`Layer.load_from_db()`** — replays LMDB records. ATTR_KEYS records arrive first (lexicographic ordering); for each restored device key, the platform calls `template(PathKey(...), lower)` which constructs `SwInstallTransform` + `DeviceRunner` for that device. During `act`, the factory installs `fn.stash_cb = runner._stash_dynstate`. ATTR_DYNSTATE records then replay, populating `_TransformTransaction.dynstate`. (Fresh installs with no LMDB records skip directly to step 4 when the user authors the device's `software-pack` subtree via `edit_config`.)
3. **`app.StartupBootstrap.recompute(force=True)`** — forces a recompute that calls `transform_wrapper(cfg, linked, memory, dynstate)`. The `dynstate` arg is the restored value. `transform_wrapper` invokes `self.stash_cb(dynstate)` — the action lands on the runner's mailbox.
4. **`_TransformTransaction.finalize`** — calls `function.on_conf(self.get(), self.memory)` if `self.running` is non-empty. This is the runner's `on_local_config`.
5. **Runner reconciliation** — by now both `_stash_dynstate(dynstate)` and `on_local_config(cfg, mem)` have fired; the runner has restored its dynstate, run §3.5 consistency check, and applied recovery rules (processing → failed-transient; cancelling → cancelled; etc.).
6. Persists Tier B and publishes oper.
7. **Backoff resume**: if `next_wake_at` future, schedule fresh `after`.

🆕 **Oper data is NOT platform-persisted across restart** (CL4_7) — clients polling during steps 1-6 see empty oper data and must retry.

🆕 **Caveat (no software-pack bound):** if `_TransformTransaction.finalize` skips `on_conf` because `self.running` is empty (no `software-pack` ever authored for this device), step 4 is a no-op. Runner stays at `runner-status = starting`. Benign; see §7.2 and §15.5 #22.

🆕 §15.5 entry: "Oper data is not platform-persisted across stratoweave restart, unlike Python NSO CDB oper. Clients polling during the runner's first reconciliation gap see empty data and must retry."

### 8.5 Cancel implementation

Per §4.4's state machine: cancel-generation increments transition `processing → cancelling`, and a drain notification (or watchdog timeout) transitions `cancelling → cancelled`.

```
on cancel-generation increment for current request:
    if cfg.cancel-target-id is set and not in current+history:
        publish last-trigger-result {kind: cancel, result: rejected,
                                     reason: "no such request id <N>"}
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
        dynstate.current.status = CANCELLED        # nothing in flight, instant
        self._persist_dynstate(Tier.B); self._publish_oper()
```

`CANCEL_DRAIN_TIMEOUT` defaults to 600s (matches IOS-XR's `_monitor_operation_log` 600s timeout — never less than the longest in-flight RPC). `_force_drain` checks the token (in case a normal drain happened first), transitions to `cancelled` if still `cancelling`.

The drain itself happens via `_drain_notify` (§8.2): when an in-flight callback returns with a stale token, the guard routes to `_drain_notify` which checks if `dynstate.current.status == CANCELLING` and `stale_token < dynstate.current.generation_token` (the v5.1 CL4_6 fix — was `stale_token + 1 == ...`, which broke after multi-generation drains). If so, transitions to `cancelled` and persists Tier B.

### 8.6 Backoff

Per-device, per-request, runs at the end of every failed run:

```
on FAILURE / WAIT terminal of a run:
    error_count.<class> += 1
    error_count.<other-class> = 0
    if error_count.<class> > config.max_retries:
        # Per-class budget exhausted — terminal status matches the failure class
        publish FAILED_<class>; gave_up = True; done
    backoff_decimal = (error_count.backoff_decimal or 10.0) * factor    # CR2: exact spec
    error_count.backoff = ceil(backoff_decimal)                          # uint32 in oper
    error_count.backoff_decimal = backoff_decimal                        # internal precise state
    next_wake_at = now() + ceil(backoff_decimal)
    self._persist_dynstate(Tier.A)                                       # before scheduling
    after error_count.backoff: self._start_run()


action def _start_run():
    # CL3_4 (round-3): re-check enabled at firing time
    if not global_config_cache.enabled:
        dynstate.current.status = PAUSED
        self._persist_dynstate(Tier.B); self._publish_oper()
        return
    # ... normal start
```

`error_count.backoff` and `next_wake_at` are projected into oper (CR3 from round-3) so operators see the retry schedule.

On restart with `next_wake_at` in the future, schedule a fresh `after max(0, next_wake_at - now): _start_run()`.

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

**`dev_registry.get(devname)` call shape** is **synchronous** — confirmed by precedent in `_RFSTransaction.__init__` (`ttt.act:2071`) and `testing.act:38`. The §7.2 `act()` callback uses the result synchronously without a callback chain.

---

## 9. Transport scope

The Python `software_install_script.py` mixes NETCONF, CLI, and SCP per-OS. v3+ splits these into clean abstractions and ships only what Phase 4 needs.

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

Used by `CheckFiles` (every per-OS step list starts with this — verify the controller has the file before doing anything else). Default Phase 4 impl uses Acton stdlib filesystem primitives accessed via the `file.FileCap` injected through `make_sw_install_transform`.

**Phase 4 deviation** (CR3_2 from round-3): when `file_transfer.caps().put == false` (Phase 4's `NoopFileTransfer`), `CheckFiles.execute` skips the controller-side filesystem check entirely. The pre-staged-image scenario relies on `RemoteFileInspector` (§9.3) for verification instead. Documented as a conscious deviation in §15.5.

### 9.3 `RemoteFileInspector` — device-side metadata via NETCONF

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

- **SROS** (`ops_sros.act`, Phase 4): `oper-file` get + `state state` queries via NETCONF — the same paths the Python `NokiaSrosNetconfStrategy.copy_file` and `is_bof_configured` use.
- **IOS-XR / Junos** (Phase 6): per-OS NETCONF state queries.

Used by `CopyImage.pre_check`: stat each filename → compare size/checksum → `SKIP_STEP` if all present and match (no upload needed); `FAILURE` if missing and no FileTransfer available; else `SUCCESS` (proceed to execute / byte transfer). The separation from `FileTransfer` is what lets Phase 4 (with `NoopFileTransfer`) still verify pre-staged images cleanly.

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

Phase 4 ships **`NoopFileTransfer`** with `caps()` returning all-false and `put`/`delete` returning `NotImplementedError`. Because `RemoteFileInspector` is a separate abstraction (§9.3), Phase 4 SROS can still verify pre-staged images via NETCONF and SKIP_STEP `CopyImage` cleanly.

`CopyImage.execute` in Phase 4: returns FAILURE with a clear "no FileTransfer configured — pre-stage the image" message if files are missing per `RemoteFileInspector`. Phase 5 fills in real `FileTransfer` impls (SCP/SFTP via `ssh_client` or device-pull).

### 9.5 Credential reuse — `DeviceMgr.get_dmc()`

### 9.6 `scp-port` placement — back to nested (🆕 D7)

🆕 v5 reverts to v3's nested placement: `/sw-rfs:rfs[name]/software-pack/scp-port`. The v4 sibling placement was driven by "scp-port should survive pack unbinding" but the v4 reviewers caught the actual cost: with the transform attached at `software-pack`, the runner doesn't see a sibling `scp-port`. The "survives unbinding" benefit was speculative (no concrete non-sw-install consumer exists today).

§15.5 #14 corrected: "scp-port lives inside software-pack/. Removing the software-pack also removes scp-port — but the operator removing the pack is disabling sw-install for the device anyway, so the loss is acceptable."

### 9.7 `DeviceOps` facade — CLI strategy boundary

The Python original mixes NETCONF and CLI per-OS — `NokiaSrosCliStrategy` vs `NokiaSrosNetconfStrategy` both implement `NokiaSrosOperationsProto`, with `NokiaSrosOperations` delegating based on device capabilities. v3+ preserves this strategy pattern.

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

`SrosOps` (and `IosXrOps`, `JunosOps` in Phase 6) implement the facade; internally they select NETCONF or CLI strategy based on `dev.get_capabilities()` / `dev.get_modules()` snapshot. **Capability snapshot is per-run** (not per-step) — fail clean on incompatible drift between runs (CR6 from round-3).

Phase 4: `SrosOps` is constructed with `cli_session = None`. NETCONF strategy methods are real; **no per-method CLI stubs raising `NotImplementedError`** — that would be dead surface that increases test burden without delivering behavior. SROS Phase 4 only invokes NETCONF paths.

**Phase 5** adds CLI strategy methods alongside the existing NETCONF ones inside the per-OS `Ops` actor; the `DeviceOps` facade signature doesn't change. **TextFSMPlus implementation details** (templates, `Send`/`Preset`/`Done` semantics, aycalc-equivalent expression eval, prompt synchronization, terminal width, command echo, secrets handling, the acton-utils textfsm extension dependency) live in `docs/adr/cli-driver.md`. That ADR is a Phase 5 design artifact; this design doc commits only to the `DeviceOps` boundary.

---

## 10. Testing strategy

(Mostly carried from v3.)

🆕 v5.2 (per round-6 A5): **`test_topology_misconfigured.act` is a required Phase 4 test.** Builds a layer stack where `/software-install/` is NOT passed through to the layer below the sw-install transform; asserts the runner reaches `runner-status = missing-global-config` after `MISSING_GLOBAL_CONFIG_GRACE` (15s), even with a valid pack assignment in local config. Serves as the regression guard against app-integration mistakes — the only "operator can shoot foot" surface in the design. Should also serve as a copy-paste template for app integrators showing the correct vs incorrect topologies side by side.

---

## 11. Implementation phasing within Phase 4 (🆕 D5 unblocks)

**🆕 v5 has NO platform prerequisites for Phase 4.** Implementation can begin immediately on the existing platform.

(Otherwise unchanged.)

---

## 12. Open decisions (round-5 questions)

| # | Question | Lean |
|---|----------|------|
| **❓Q1** | (was: verify `transform_wrapper` post-restore recompute timing) — **resolved** by claude r5 §A1 trace through `app.act:138-152` `StartupBootstrap._run`; v5.1 §3.6 prose reflects the verified ordering. | resolved |
| **❓Q2** | (was: `dev_registry.get(devname)` synchronous?) — **resolved by existing precedent**: `_RFSTransaction.__init__` (`ttt.act:2071`) and `testing.act:38` both call it synchronously and use the result directly. | resolved |
| **❓Q3** | (was: TransformActorParams.dynstate platform addition) — **resolved** by D5 stash path. | resolved |
| **❓Q4** | (was: layer topology — fresh-integrator footgun) — **resolved** by §2 concrete topology + §7.2 runner-status guard. | resolved |
| **❓Q5** | Run-log default bound. | 1000 entries/request. |
| **❓Q6** | Phase 5 = TextFSMPlus + DeviceOps CLI + FileTransfer infra; Phase 6 = IOS-XR + Junos + polish. | confirmed. |
| **❓Q7** | `CANCEL_DRAIN_TIMEOUT` default. | 600s. |

---

## 13. Implementation details deferred to first-cut coding

These resolve naturally during Phase 4 implementation; flagged here so they don't become surprises:

- **Exact lmdb key layout for runner dynstate.** Follows `_TransformTransactionBase.db_ops` patterns; the platform handles the layout, sw-install just produces a single `gdata.Node` per transform via `update_dynstate(...)`.
- **`update_oper` snapshot frequency.** Coalesce per-tick polls; flush on every state-class transition (request status change, step boundary, run-log entry that crosses a tier-class threshold per §3.7).
- **Acton stdlib logging-handler glue for `swi_*` attribute filtering.** §6.6's `RunLogHandler` needs to be implemented as a concrete `logging.Handler` subclass that inspects `record.data` for the `swi_component` key. Trivial; flagged only because Acton's `logging` module is less feature-rich than Python's.
- **Acton-stdlib filesystem primitives used by `LocalFileInspector`.** Likely `file.FileCap` + `file.ReadFile` + a stat helper; the public API takes `file_cap` so this is encapsulated behind the inspector boundary.

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

## 14.1 Platform observations (not blocking, worth flagging upstream)

- 🆕 v5.1 (CL5_6): `ttt.Transform`'s `lower=` keyword arg (`ttt.act:1907`) is **dead code** — shadowed by the inner `_create_transform_node(path, lower)` parameter. Callers passing `lower=` see it silently dropped. Either make the platform consume it or remove the kwarg from the public signature.

---

## 15. Deferred features

Things deliberately **not** in this design, captured here so future contributors find them rather than rediscovering them:

- **`software-install-matrix`** (Python YANG): unused by Python core logic. Dropped from YANG; not modelled in Acton; re-add only if a real use case surfaces.
- **CLI / TextFSMPlus parsing**: Phase 5 dependency on the acton-utils textfsm extension (Send/Preset/Done line actions + aycalc-equivalent expression eval). Reference impl `/Users/ayourtch/rust/ayclic/aytextfsmplus`. See `docs/adr/cli-driver.md`.
- **IOS-XR archive parsing** (`get_version_from_iso` / `get_file_packages`): controller-side iso/tar reading. Phase 6 dependency on iso/tar libraries in Acton.
- **VRP step module**: enum value preserved, `ValidatePlatform` fails clean. Implementation deferred indefinitely.
- **Snabb / ONS-TL1 / HGW**: dropped from the Acton port (logic-spec §6.4). Dropped, not deferred — re-adding would require revisiting the device-OS detection logic.
- **Approval-required / multi-stage commit hooks**: out of scope for sw-install — that's the platform's job (`DeviceMetaConfig.approval_required` already exists).
- **`DeviceMgr.acquire_exclusive(...)` lease API**: v2.0 platform prerequisite (§14 item 1).
- **acton-utils textfsm extension**: parallel platform workstream that sw-install Phase 5 consumes. See ADR.

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
20. 🆕 v5.1 **Per-device `runner-status` oper enum** (`starting`, `ok`, `missing-global-config`, `restore-inconsistent`, `paused-by-enabled`, `waiting-for-device`) surfaces lifecycle/topology/restore/pause/device-readiness conditions. No Python NSO equivalent — Python's NSO oper status was solely the `request[].status` enum.
21. 🆕 v5.1 **Trigger mechanism: NSO actions (per-call RPCs with immediate output) → durable config generation counters sampled by reconciliation.** This is the single biggest API-shape deviation operators porting from NSO will trip over: repeated writes of the same generation are ignored; concurrent clients can race on the same/different values; `*-target-id` is sampled at observation time; backup/restore can make counters move backward relative to dynstate; `request-generation` explicitly forces a NEW request (not idempotent like Python's `create-request`). See §4 control surface for details and §3.7's `materialized_by_request_generation` anchor for the v5.1 idempotency mechanism.
22. 🆕 v5.1 **Steady-state `runner-status = starting` when no software-pack is bound to a device.** Operator should treat `starting` plus an empty `request[]` list as "no work configured" rather than "failure to initialize." Consequence of the platform's `_TransformTransaction.finalize` skipping `on_conf` when `self.running` is empty.

---

## 16. LOCK-IN

v5.3.1 is the lock-in version. Round-8 review produced zero HIGH findings; both reviewers (codex r8, claude r8) independently surfaced the same 1 MED + 2 LOW items, all of which v5.3.1 lands. Both reviewers' verdict: **"Green-light Phase 4."** No further design iteration is warranted.

**Convergence trajectory across 8 review rounds (HIGH count, codex+claude):**
```
r3: 6
r4: 5
r5: 3+3
r6: 2+2
r7: 1+1 (different items per reviewer; both real)
r8: 0+0 (zero HIGHs; matched 1 MED + 2 LOWs)
```

This is the strongest convergence signal achievable: independent reviewers, fresh context each round, all items addressed, both green-lit.

**Phase 4 implementation can start.** The next concrete step is `acton build` of the `stratoweave/sw-install/` scaffold + first TDD cycle (`pack.act` + `test_pack.act`). See §11 for the implementation phasing within Phase 4.
