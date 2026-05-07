# 02 — Stratoweave `software-install` Module Design (v5.1)

> **Status: v5.1 — incorporates round-5 review feedback from `docs/reviews/12-codex-design-r5.md` and `docs/reviews/13-claude-design-r5.md`, integrated in `docs/reviews/14-integration-r5.md`. Both r5 reviewers explicitly endorsed v5's architectural direction; v5.1 lands the 13 doc/skeleton-shape fixes both reviewers identified. Pending round-6 review for green-light to Phase 4 implementation.**

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

### 3.6 How the runner receives restored dynstate — action-ref push (🆕 v5.1 CL5_1)

🆕 v5.1 changes the stash mechanism from "shared mutable field on `SwInstallTransform`" to "action ref pushed at the runner". Round-5 caught that v5's field-stash breaks actor isolation: `SwInstallTransform` is owned by `_TransformTransaction`'s actor (which writes the field via `transform_wrapper`), and the runner actor would read it from a different actor context. v5.1's action-ref push keeps everything inside actor message-passing.

**Lifecycle ordering** (verified against `app.act:138-152` `StartupBootstrap._run` and `ttt.act` line 1942-1993):

1. `Layer(...)` rootgen constructs `SwInstallTransform` and calls `act(...)`. The runner is constructed **here**, before any restore. Runner starts with empty in-memory dynstate (`SwInstallDynstate.empty()`).
2. During `act(...)`, the runner installs an action-ref `stash_cb` on the `SwInstallTransform` instance. (The function instance and the runner are constructed in the same lexical scope of the factory; both close over the holder that links them.)
3. `Layer.load_from_db()` restores `_TransformTransaction.dynstate` from LMDB.
4. `app.StartupBootstrap.recompute(force=True)` calls `transform_wrapper(cfg, linked, memory, dynstate)`. The `dynstate` argument is the restored value at this point.
5. `transform_wrapper` invokes `self.stash_cb(dynstate)` — an action ref pointing at the runner. Acton dispatches the action to the runner's mailbox.
6. `_TransformTransaction.finalize` calls `function.on_conf(self.get(), self.memory)` if `self.running` is non-empty. This is the runner's `on_local_config`.
7. The runner has now received both the dynstate (via stash_cb) and the first config (via on_local_config). It applies the dynstate, runs the §3.5 consistency check, and begins reconciliation.

**Code shape:**

```acton
class SwInstallTransform(ttt.TransformFunction):
    # 🆕 v5.1: action-ref push, not shared mutable field
    var stash_cb: ?action(?gdata.Node) -> None = None
    var stash_done: bool = False

    def transform_wrapper(self, cfg, linked, memory, dynstate):
        # Runs during the post-restore recompute (StartupBootstrap force=True).
        # dynstate at this point is the LMDB-restored value.
        cb = self.stash_cb
        if cb is not None and not self.stash_done:
            cb(dynstate)              # action call — lands on runner mailbox
            self.stash_done = True
        # 🆕 v5.1 CL5_3: return None for memory (memory is unused for sw-install;
        # returning the input unchanged would surprise readers).
        return (gdata.Container(), None)


actor DeviceRunner(
    ...,
    transform_fn: SwInstallTransform,    # for installing stash_cb at construction
    ...
):
    var dynstate: SwInstallDynstate = SwInstallDynstate.empty()
    var dynstate_initialized: bool = False

    # Install the stash callback at runner-construction time, before the
    # platform's recompute can fire transform_wrapper.
    transform_fn.stash_cb = self._stash_dynstate

    action def _stash_dynstate(stashed: ?gdata.Node):
        if stashed is not None and not self.dynstate_initialized:
            self.dynstate = SwInstallDynstate.from_gdata(stashed)
        self.dynstate_initialized = True
        self._check_restore_consistency()

    action def on_local_config(cfg: ?gdata.Node, mem: ?gdata.Node):
        if not self.dynstate_initialized:
            # No restored dynstate (fresh install) — proceed with empty state.
            self.dynstate_initialized = True
            self._check_restore_consistency()
        # ... reconciliation ...
```

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
- 🆕 **`current.materialized_by_request_generation: u64`** (per round-5 codex H1) — when a new request is materialized in response to `request-generation`, this field on `RequestState` records which generation triggered it. The §4.1 reconciliation rule becomes: "if the trigger's `request-generation` value equals the current request's `materialized_by_request_generation`, the current request already corresponds to this trigger — no-op." This makes `request-generation` re-fire-safe across crash+restart.
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

(Cannot reuse `_DeviceTransaction.devname_from_device_path` directly — that helper expects `path: PathKey`, but our `params.path` is a `PathElem` with a PathKey parent.)

🆕 v2.0 platform ask: parameterize `_RFSTransaction.finalize`'s empty-output suppression OR thread `lower` through `RFSFunction.init_dynstate`. Either change would let sw-install move to `RFSTransform` and the platform convention realigns. Captured in §14.

### 7.2 Global config subscription + runner-status guard (🆕 D8; v5.1 fixes)

```acton
proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
    devname = devname_from_swi_path(params.path)
    runner = DeviceRunner(
        params.path, params.update_oper, params.update_dynstate,
        dev_registry.get(devname),
        transform_fn,             # 🆕 v5.1: for stash_cb installation (§3.6)
        ...
    )
    transform_fn.stash_cb = runner._stash_dynstate    # 🆕 v5.1: action-ref push
    if params.lower is not None:
        params.lower.declare_subscriptions(
            owner_id="sw_install:" + devname,
            cb=runner.on_global_config,
            want={SubscriptionSpec(filt=SOFTWARE_INSTALL_FILTER, period=5.0)},  # 🆕 v5.1: pinned period
        )
    return lambda cfg, mem: runner.on_local_config(cfg, mem)
```

🆕 **Runner-status startup guard** (v5.1 rewrite per round-5 A2):

The runner publishes `runner-status` to oper. Critical fix from v5: the 5s `missing-global-config` timer starts at "first non-None on_global_config callback", **NOT** at "first on_local_config + 5s." Round-5 (claude r5 H3) caught that the platform's first `_sub_tick` fires synchronously during `declare_subscriptions` (before `Layer.load_from_db()` has populated the lower layer), delivering `None`. v5's earlier wording would false-alarm on every well-wired startup.

State transitions:

```
starting          (initial — actor constructed, neither config callback has fired with non-None data)
   │
   │ first on_local_config(cfg!=None) AND first on_global_config(merged!=None containing /software-install/)
   ▼
ok                (steady state)

starting / ok ──(cb sequence: on_global_config(None) repeats past 5s timeout from runner construction)──▶ missing-global-config
                  (operator's app composition is wrong — /software-install/ is not in the lower layer)

ok ──(§3.5 dynstate-internal inconsistency on startup)──▶ restore-inconsistent

ok / processing ──(/software-install/enabled = false)──▶ paused-by-enabled

ok ──(DeviceMgr returns NoAdapter — no DMC set)──▶ waiting-for-device
```

**"Global config seen" defined precisely** (per round-5 codex r5 HIGH 3 data semantics): the cb received a non-None `gdata.Node` whose root contains the `/software-install/` container at any depth in its merged tree. An empty `/software-install/` container counts as "seen" (operator authored the subtree but no packs yet); absent `/software-install/` does NOT count.

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
        fn_holder: list[SwInstallTransform] = []

        def function_factory(log_handler: ?logging.Handler) -> SwInstallTransform:
            fn = SwInstallTransform(log_handler=log_handler)
            fn_holder.append(fn)
            return fn

        proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
            fn = fn_holder[0]                          # the same instance used by transform_wrapper
            devname = devname_from_swi_path(params.path)
            runner = DeviceRunner(
                params.path,
                params.update_oper,                    # proc(?gdata.Node) -> None
                params.update_dynstate,                # proc(?gdata.Node) -> None
                dev_registry.get(devname),
                fn,                                    # for installing stash_cb
                ...
            )
            fn.stash_cb = runner._stash_dynstate       # 🆕 v5.1: action-ref push setup
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

### 8.1 Lease scope (unchanged from v4)

### 8.2 The DeviceRunner actor — type fixes (🆕 CL4_1, CL4_6)

```acton
actor DeviceRunner(
    path: ttt.Path,
    update_oper: proc(?gdata.Node) -> None,             # 🆕 CL4_1: was action(...)
    update_dynstate: proc(?gdata.Node) -> None,         # 🆕 CL4_1: was action(...)
    dev: swdev.DeviceMgr,
    transform_fn: SwInstallTransform,                   # for installing stash_cb (§3.6)
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

On platform startup, the actual ordering (verified against `app.act:138-152`):

1. **Layer rootgen** — `Layer(...)` calls `rootgen(...)`, which constructs `SwInstallTransform`. The `act` callback fires during `init_dynstate`; runner is constructed **here** with empty in-memory dynstate. Runner installs `stash_cb` on `SwInstallTransform`. The actor is alive and running with empty state.
2. **`Layer.load_from_db()`** — restores `_TransformTransaction.dynstate` from LMDB. The transform-transaction's dynstate field now holds the restored value, but the runner doesn't know yet.
3. **`app.StartupBootstrap.recompute(force=True)`** — forces a recompute that calls `transform_wrapper(cfg, linked, memory, dynstate)`. The `dynstate` arg is the restored value. `transform_wrapper` invokes `self.stash_cb(dynstate)` — the action lands on the runner's mailbox.
4. **`_TransformTransaction.finalize`** — calls `function.on_conf(self.get(), self.memory)` if `self.running` is non-empty. This is the runner's `on_local_config`.
5. **Runner reconciliation** — by now both `_stash_dynstate(dynstate)` and `on_local_config(cfg, mem)` have fired; the runner has restored its dynstate, run §3.5 consistency check, and applied recovery rules (processing → failed-transient; cancelling → cancelled; etc.).
6. Persists Tier B and publishes oper.
7. **Backoff resume**: if `next_wake_at` future, schedule fresh `after`.

🆕 **Oper data is NOT platform-persisted across restart** (CL4_7) — clients polling during steps 1-6 see empty oper data and must retry.

🆕 **Caveat (no software-pack bound):** if `_TransformTransaction.finalize` skips `on_conf` because `self.running` is empty (no `software-pack` ever authored for this device), step 4 is a no-op. Runner stays at `runner-status = starting`. Benign; see §7.2 and §15.5 #22.

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
| **❓Q1** | (was: verify `transform_wrapper` post-restore recompute timing) — **resolved** by claude r5 §A1 trace through `app.act:138-152` `StartupBootstrap._run`; v5.1 §3.6 prose reflects the verified ordering. | resolved |
| **❓Q2** | (was: `dev_registry.get(devname)` synchronous?) — **resolved by existing precedent**: `_RFSTransaction.__init__` (`ttt.act:2071`) and `testing.act:38` both call it synchronously and use the result directly. | resolved |
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

## 14.1 Platform observations (not blocking, worth flagging upstream)

- 🆕 v5.1 (CL5_6): `ttt.Transform`'s `lower=` keyword arg (`ttt.act:1907`) is **dead code** — shadowed by the inner `_create_transform_node(path, lower)` parameter. Callers passing `lower=` see it silently dropped. Either make the platform consume it or remove the kwarg from the public signature.

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
20. 🆕 v5.1 **Per-device `runner-status` oper enum** (`starting`, `ok`, `missing-global-config`, `restore-inconsistent`, `paused-by-enabled`, `waiting-for-device`) surfaces lifecycle/topology/restore/pause/device-readiness conditions. No Python NSO equivalent — Python's NSO oper status was solely the `request[].status` enum.
21. 🆕 v5.1 **Trigger mechanism: NSO actions (per-call RPCs with immediate output) → durable config generation counters sampled by reconciliation.** This is the single biggest API-shape deviation operators porting from NSO will trip over: repeated writes of the same generation are ignored; concurrent clients can race on the same/different values; `*-target-id` is sampled at observation time; backup/restore can make counters move backward relative to dynstate; `request-generation` explicitly forces a NEW request (not idempotent like Python's `create-request`). See §4 control surface for details and §3.7's `materialized_by_request_generation` anchor for the v5.1 idempotency mechanism.
22. 🆕 v5.1 **Steady-state `runner-status = starting` when no software-pack is bound to a device.** Operator should treat `starting` plus an empty `request[]` list as "no work configured" rather than "failure to initialize." Consequence of the platform's `_TransformTransaction.finalize` skipping `on_conf` when `self.running` is empty.

---

## 16. Round-5 review

This v5 design integrates all round-4 feedback. Both reviewers' five HIGH-priority items (lifecycle/D3a-broken, Tier A unimplementable, scp-port sibling visibility, subscription-wiring footgun, action vs proc type) are addressed. v5 unblocks Phase 4 with no platform prerequisites; round-3's "platform addition required" is gone.

**Stop here for round-5 review.** Both reviewers will be re-briefed against the full revised doc set (`00-orientation.md`, `01-software-install-logic.md`, `02-sw-install-design.md` v5, `docs/adr/cli-driver.md`, `yang.act` v5) with no carry-over context.
