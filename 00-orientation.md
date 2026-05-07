# 00 — Orientation

This doc gets you (Andrew) and any other reader onto the same page about **what we're building**, **what we're building it inside of**, and **what we're building it from**. If you read just one doc before joining a conversation about this project, read this one.

> **Reading order for the full doc set:**
> 1. `docs/00-orientation.md` (this doc) — project context + stratoweave concepts
> 2. `docs/01-software-install-logic.md` — language-agnostic spec extracted from the Python source
> 3. `docs/02-sw-install-design.md` — proposed Acton module design (current: v5.3 — locked in for Phase 4 implementation)
> 4. `docs/adr/cli-driver.md` — Phase 5 CLI driver design intent
> 5. `stratoweave/sw-install/src/sw_install/yang.act` — the v5 YANG model
>
> Reviews and integration docs live under `docs/reviews/` for historical reference.

---

## TL;DR

We are taking the existing **NSO/Python `software-install` package** (`activities/software-install/`) and **reimplementing equivalent functionality as a new module inside the StratoWeave platform** (`stratoweave/stratoweave/`), written in **Acton**. The YANG model carries over largely intact; the imperative NSO worker logic gets re-thought as reactive transforms and adapter calls.

The end state: a stratoweave-native module that lets a user declaratively specify "this device should run software pack X" and have a background worker drive the device through the install/upgrade steps, with run logs, confirm-step gating, and per-OS step modules — but living inside stratoweave's typed, layered, reactive world.

---

## Where things live

```
acton/
├── acton/                       # The Acton language itself (compiler, runtime, stdlib)
├── activities/                  # Reference material — things to be ported
│   └── software-install/        #   ← NSO/Python source of truth for this project
├── stratoweave/                 # 11 sibling repos that make up the platform & ecosystem
│   ├── stratoweave/             #   ← the platform core (where our new module lives)
│   ├── sorespo/                 #   reference implementation; closest analog to what we're building
│   ├── netconf/                 #   NETCONF transport adapter (we'll use this)
│   ├── acton-yang/              #   YANG → Acton-types compiler library
│   ├── netcli, netclics/        #   CLI/SSH transports (relevant only if we add CLI later)
│   └── …
└── docs/                        # ← onboarding + design docs (this directory)
```

The user's working directory is `/Users/ayourtch/acton`.

---

## Part 1 — What is StratoWeave?

> StratoWeave is a platform for the development of robust network orchestration systems based on **model-driven declarative transforms**.

Mental model: it's a **typed, layered, reactive data pipeline** between your northbound APIs (config you receive) and your southbound devices (NETCONF/gNMI/CLI sessions you drive). You write **transforms** that take an input data tree and produce an output data tree; transforms compose into **layers**; layers stack to form a **system spec**.

### The four big abstractions

| Abstraction | Lives in | What it is |
|-------------|----------|------------|
| **`gdata.Node`** | `acton-yang` package | The runtime tree value. A `Container` / `List` / `Leaf` / `Absent` etc. — a YANG-typed tree. Everything that flows between layers is a `gdata.Node`. |
| **`ttt.Layer`** | `stratoweave/ttt.act` | One stage in the pipeline. Holds a tree, accepts `edit_config(diff)`, runs its transform(s), publishes downward. Has a `lower` reference forming a stack. |
| **`ttt.Transform`** | `stratoweave/ttt.act` (`Transform`, `TransformFunction`) | A pure-ish function from the tree-segment above to a tree-segment below. Subclass `TransformFunction` and override `transform_wrapper`. |
| **`swdev.DeviceAdapter`** | `stratoweave/adapters/adapter.act` | Abstract base class for talking to a real device. `configure(...)`, `fetch_config(...)`, `rpc_xml(...)`, `declare_subscriptions(...)`. Concrete impls: `NetconfAdapter`, `MockAdapter`, `NoAdapter`. |

### The data flow, end to end

```
            northbound config (XML/JSON via RESTCONF/NETCONF/files)
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  Layer 0  ("CFS" — customer-facing service, highest level)   │
   │  ─ TransformFunction subclasses convert L0 trees → L1 trees  │
   └──────────────────────────────────────────────────────────────┘
                                  │   edit_config(diff) downward
                                  ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  Layer 1, Layer 2, ...                                       │
   │  ─ each one transforms what it receives from above           │
   └──────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  Bottom layer ("RFS" — resource-facing service)              │
   │  ─ produces per-device configuration                         │
   └──────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                         swdev.DeviceMgr (per device)
                                  │
                                  ▼
                   DeviceAdapter (NETCONF/gNMI/CLI/Mock)
                                  │
                                  ▼
                            real device
```

### The "system spec" (`SysSpec`)

`stratoweave/app.act` defines `SysSpec(device_types, get_layers, schema_for_layer)`. Every concrete app (sorespo, our future sw-install user, etc.) provides one of these. `get_layers` builds the layer stack from top to bottom; `device_types` maps device-type names to `(adapter_class, bundled YANG schema)` pairs.

`sorespo/src/sorespo/sysspec.act` is a real example — it builds 4 layers (`t_3` through `t_0`) and registers three device types (Cisco IOS-XR, Juniper cRPD, Nokia SR Linux), all using `NetconfAdapter`.

### YANG-as-types

Acton has static types. StratoWeave compiles YANG models into Acton classes via the `acton-yang` library and a small per-app `gen_adata` build step (`stratoweave/gen_adata/src/gen_adata.act`). After running it you get:

- `device_meta_config.act` — typed access to the per-device meta config (credentials, address, type, …)
- `ietf_yang_library.act`, `ietf_restconf_monitoring.act` — typed accessors for IETF standard models
- For sorespo: `layers/y_0.act`, `y_1.act`, …, `devices/CiscoIosXr_25_3_1.act` etc.

The point: **typos in YANG paths become compile errors**, not 3 a.m. pages.

### Key actors / concurrency

Acton is actor-based. The code uses `actor` for things-with-state:

- `actor Layer(...)` (defined in `ttt.act`) — the per-layer actor that owns a tree node, accepts `edit_config`, and runs transforms
- `actor Session(...)` — a transactional view into a stack of layers (open one with `layer.newsession()`, then `edit_config(...)` / `commit()`)
- `actor DeviceRegistry(...)` — singleton-per-app registry mapping device name → `DeviceMgr` actor
- `actor DeviceMgr(...)` — per-device state machine (running config, target config, txid, adapter). Delegates protocol-specific work to a `DeviceAdapter` (`NetconfAdapter`, `MockAdapter`, ...); modules that drive devices typically interact with `DeviceMgr`'s public surface (`configure`, `rpc_xml`, `fetch_config`, `declare_subscriptions`) and don't reach into the adapter directly.
- `actor NetconfDriver(...)` / `NetconfAdapter` — wraps the NETCONF session

Communication is async; callbacks are typed `action(...)`. There is no shared mutable state across actors.

### Operational state and the observer-transform pattern

Most stratoweave transforms produce *downward config* — the input tree is transformed into the next-layer-down tree. But there is a second important pattern: a transform that produces **no downward config**, instead consuming an input config tree and publishing **operational state** (status, plan, telemetry, anything not user-written). sw-install fits this shape — the user writes a desired pack assignment; the module publishes request status, plan, and run-log as oper data.

The platform supports this via `TransformActorParams` (`stratoweave/ttt.act`), which gives a transform's `act`-spawned actor four handles:

- **`update_oper(?gdata.Node)`** — publishes oper-data tree merged into the transform's output. Read by `Layer.get_data(...)` callers (RESTCONF, CLI tools, other transforms).
- **`update_dynstate(?gdata.Node)`** — persists actor-private state via lmdb, restored on platform startup. The runner's source of truth.
- **`dev: ?swdev.DeviceMgr`** — for RFS transforms; lets the actor talk to the device.
- **`lower: ?Layer`** — read-only access to the layer below; supports `declare_subscriptions(...)` for observing data outside the transform's local input.

A working example is `sorespo/src/sorespo/rfs.act:BBInterfaceTransform` — a per-list-entry transform whose actor uses `update_oper` to publish telemetry and `update_dynstate` to persist accumulated counters across restarts. sw-install follows the same per-list-entry shape, attached inside the `/sw-rfs:rfs` list (the RFS-layer per-device list — same place sorespo's RFS transforms attach), with one transform instance per device. Note: sw-install uses **plain `ttt.Transform`**, not `RFSTransform` — sw-install produces no downward config and needs `params.lower` (not `params.dev`) for global-config subscription.

`transform_wrapper(cfg, linked, memory, dynstate)` returns `(downward_config_diff, new_memory)`. For an observer-shaped module, `downward_config_diff` is empty (`gdata.Container()`) — the transform mechanism still gives access to all the auxiliary services. `memory` is also typically unused for observer-shaped modules; everything goes in `dynstate`.

Transforms can also subscribe to data **outside their local input** via `params.lower.declare_subscriptions(...)` — this is how, for example, sw-install's per-device runner reads the global `/software-install/...` config that lives elsewhere in the layer stack. (See `ttt.act:735` — `Layer.declare_subscriptions`.)

#### Quick reference: where data lives

| Surface | What it is | Mutability | Persisted? |
|---------|-----------|------------|-----------|
| **Config (gdata)** | User-authored input to the layer (or transformed-into-this-layer from above). The transform's `cfg` arg comes from here. | external read/write | by the platform (lmdb) |
| **Memory (gdata)** | Per-transform persistent state, returned from `transform_wrapper` as the second return value. Visible to subsequent calls of the same transform's `transform_wrapper`. | transform-internal (returns shape it) | by the platform (lmdb, ATTR_MEMORY) |
| **Dynstate (gdata)** | Per-transform actor-private state, written via `update_dynstate(...)`. Visible to `transform_wrapper` as the fourth arg. Doesn't currently flow to the actor at restore time without a workaround (see sw-install design §3.6). | actor-internal (writes via update_dynstate) | by the platform (lmdb, ATTR_DYNSTATE) |
| **Oper (gdata)** | Operational state published via `update_oper(...)` and read by `Layer.get_data(...)` callers. Pure projection of dynstate (or computed view). | external read-only | NOT persisted; rebuilt on restart |

Most transforms use `cfg` and `memory`. Observer-shaped modules like sw-install lean heavily on dynstate + oper and barely touch memory.

### Persistence

`lmdb.Db` is used to back layer state. Pass it into `Layer(...)`. Layers can `load_from_db()` on restart. Optional — not all apps persist.

### Testing

Tests live next to source files as `test_<thing>.act`. They define `actor <test_name>(t: testing.AsyncT):` — the `AsyncT` is a test-completion handle; you call `t.success()` when the actor's flow finishes, or let an `assertEqual` failure propagate. Run with `acton test` (top-level Makefile has `make test`).

The `stratoweave/testing.act` module provides `TestRig` — a convenience wrapper that bundles a `Layer`, a `DeviceRegistry`, and a `SysSpec` for integration-style tests.

### Build & licensing

- Each repo has `Build.act` declaring deps (URL + sha-pinned)
- REUSE compliance: `REUSE.toml` + `LICENSES/` directory; **every file under `src/`, `Makefile`, `**/Build.act` etc. must carry an SPDX header or be covered by a glob in `REUSE.toml`**. License is **BSD-3-Clause** for code, **CC0-1.0** for docs/config, copyright **Deutsche Telekom AG** in stratoweave.

---

## Part 2 — What is `software-install` (the existing package)?

The NSO/Python package at `activities/software-install/` provides a **declarative interface for installing OS images and patches on network devices**. Think: "I want this device to be running IOS-XR 6.5.2 with these three SMUs" — the package figures out the steps to get there.

### Vocabulary

| Term | Means |
|------|-------|
| **software-pack** | A named profile = `{ os, base?, patches }`. Defined globally, applied to a device. |
| **component** | One element of a pack — either the `base` image or one `patch`. Each has `version` + `filename(s)`. |
| **request** | A snapshot of a software-pack bound to a specific device. Created by the `create-request` action. Immutable once created. |
| **plan** | The list of *components × steps* generated for a request (e.g. for SROS: `CheckFiles, CheckVersions, ActivatePrimary, …, Reboot, Done`). |
| **step** | An atomic action with `execute()` returning `SUCCESS / FAILURE / SKIP / WAIT`. Per-OS Python class. |
| **run** | One execution attempt of a request. A request can be retried; each retry is a new run with its own `run-id` and run-log. |
| **confirm-steps** | Optional gating mode — pause before each step waiting for an explicit `confirm-step` action. |

### Architecture (concrete files)

```
activities/software-install/packages/sw-install/
├── src/yang/software-install.yang         # 442 lines — the data model
├── src/yang/software-install-ann.yang     # YANG annotations
└── python/software_install/
    ├── actions.py                         # NSO action callbacks: create-request, execute-request, confirm-step
    ├── component_plan.py                  # ComponentPlan: per-request plan state, refresh, advance
    ├── context.py                         # SoftwarePack, State, Context (logger), enums
    ├── device_os.py                       # OS enum + dispatch
    ├── step_executor.py                   # Step execution loop, retry/backoff, run-log writing
    ├── step_executor_classes.py           # StepExecutor base + StepResult enum
    ├── request_worker.py                  # Background worker — pulls queued requests, runs them
    ├── cisco.py / junos.py / sros.py      # Per-OS step lists & step classes
    └── software_install_script.py         # Low-level device interaction (1424 lines!) — NETCONF/CLI ops per OS
```

### State machine, summarized

```
unprocessed
    │ execute-request action
    ▼
processing  ←──────────────┐
    │                      │
    ├─ step.execute()      │ retry (transient error, within max_retries)
    │  ├─ SUCCESS          │
    │  ├─ FAILURE ─────────┘
    │  ├─ SKIP            
    │  └─ WAIT (e.g., reboot in progress) → re-poll later
    │
    ├─ all components & steps reached?
    │   └─ done
    ├─ permanent error?  → failed-other
    ├─ retries exhausted?→ failed-transient
    ├─ user cancels?     → cancelled
    ├─ pack changed?     → obsolete
    └─ confirm-steps on, next step needs confirm?
                          → waiting-confirmation  ─(confirm-step action)→ processing
```

### Per-OS step lists (preview)

- **SROS**: `ValidatePlatform → CheckFiles → CheckVersions → ActivatePrimary → GetBootTime → CopyImage → PrepareCopyBootLdr → PrepareConfigureBof → PrepareHackFormatStandby → PrepareSaveRollback → PrepareSynchronizeBootenv → Reboot → Done`
- **IOS-XR**: similar shape; `cisco.py` (343 lines)
- **Junos**: similar shape; `junos.py` (351 lines)

We will do a precise, line-by-line extraction in Phase 2 (`docs/01-software-install-logic.md`) — this orientation just notes the rough shape.

---

## Part 3 — How do we map one onto the other?

This is the conceptual mapping that the design doc (Phase 3) will turn into concrete types.

| NSO/Python concept | Stratoweave/Acton equivalent |
|--------------------|------------------------------|
| YANG model `software-install.yang` (config + oper) | The same YANG, fed through `gen_adata` to produce typed Acton classes |
| `software-install/software-pack` config list | A node in some layer's gdata tree |
| `devices/device/software-pack name=<x>` association | Reference under the per-device subtree (likely RFS-ish layer) |
| `create-request` action | A transform that materializes a "request" gdata subtree when its inputs change. Action-as-RPC could be added separately for explicit control. |
| `execute-request` action | A side-effecting actor — picks up new request subtrees and starts execution |
| `request_worker` thread pool | A stratoweave actor (one per request, or pool) that drives steps |
| `StepExecutor` Python classes | Acton classes implementing a `Step` protocol; per-OS modules export `get_steps(state)` |
| Device interaction (`software_install_script.py` NETCONF calls) | Calls through `swdev.DeviceMgr` → `NetconfAdapter` (`rpc_xml`, `fetch_config`, `configure`) |
| `OperCdbLoggingHandler` (run-log into CDB) | Either a writable oper-data subtree maintained by the worker actor, or an `actor RunLog` that owns log entries |
| `confirm-step` action | An explicit input on the request subtree, gated by the worker actor |
| Retry + backoff (`error-handling`) | Worker actor's internal scheduling using `after <delay>:` |

### The tricky bits we'll need to decide on in Phase 3

1. **Where do request lifecycle and state live?** They're both *config-derived* (the pack and the device association are config) and *operationally evolving* (status, current step, run-log). Stratoweave's pattern is layers transform config → config; per-device DeviceMgr holds running config + target config. Operational state is a separate axis. We'll need a clear answer for "what part of the request lives in a layer's gdata, and what lives in actor state."

2. **Actions vs. transforms.** Stratoweave is reactive; you don't have NSO-style `request execute-request` actions natively. Options: (a) treat configuration changes as the trigger (idempotent, more "stratoweave-like"); (b) add an explicit RPC mechanism (RESTCONF actions, an actor message). (a) is more native; (b) is more familiar to NSO users. Probably (a) with a thin (b) façade for compatibility.

3. **Step execution context.** NSO steps get a `Context` with logger, MAAPI handles, and a mutable `State`. The Acton equivalent is messier — actors don't share mutable state. Likely shape: each step is a `proc def execute(state: StepState, dev: DeviceMgr, ctx: StepCtx) -> StepResult` and `StepState` is a value type that gets returned/replaced rather than mutated.

4. **CLI vs. NETCONF.** The Python `software_install_script.py` uses both NETCONF and platform CLI. Stratoweave's first-class adapter is NETCONF. CLI exists (`netcli`, `netclics`) but adding a CLI dependency widens the surface. **Recommendation: scope MVP to NETCONF-only** and call out CLI features explicitly skipped. (Decision point in Phase 3.)

---

## Part 4 — How to be useful in conversation about this

If you're (re-)joining this project mid-stream and want to be helpful, the high-leverage places to look are:

**To talk about stratoweave shape:**
- `stratoweave/stratoweave/src/stratoweave/ttt.act` — search for `actor Layer`, `actor Session`, `class Transform`
- `stratoweave/stratoweave/src/stratoweave/device.act` — `DeviceRegistry`, `DeviceMgr`
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act` — `DeviceAdapter` base class
- `stratoweave/sorespo/src/sorespo/cfs.act` — small, readable transform examples
- `stratoweave/sorespo/src/sorespo/sysspec.act` — how an app wires layers together

**To talk about what we're porting:**
- `activities/software-install/README.rst` — the user-facing docs (good for vocabulary)
- `activities/software-install/packages/sw-install/src/yang/software-install.yang` — the data model
- `activities/software-install/packages/sw-install/python/software_install/component_plan.py` — the plan abstraction (~245 lines, central)
- `activities/software-install/packages/sw-install/python/software_install/sros.py` — simplest per-OS module; good for understanding the step pattern
- `activities/software-install/packages/sw-install/python/software_install/step_executor.py` — the run loop with retry/backoff

**Useful questions to ask yourself when reviewing my work:**
- Does the YANG model survive intact, or am I quietly dropping leaves?
- Is the request/run/component/step state machine preserved, or am I simplifying it past the point of equivalence?
- Am I using stratoweave's reactive idioms, or am I just transliterating Python into Acton?
- Are there corners of the Python code I'm hand-waving past? (Honest answer for now: `software_install_script.py` is 1424 lines and I have not read it cover-to-cover yet — Phase 2 is where that happens.)

---

## What's next

- **Phase 2** (next): extract the precise logic into `docs/01-software-install-logic.md` — language-agnostic, exhaustive enough that two engineers reading it would implement compatible things.
- **Phase 3**: design the stratoweave module — types, transforms, integration. **Stop for review before coding.**
- **Phase 4+**: skeleton → SROS first → tests → IOS-XR → Junos → polish.

Open questions I'm flagging now (will resurface in Phase 3):
- Action-style RPC vs. config-driven trigger?
- NETCONF-only MVP, or do we tackle CLI?
- How is operational state (run-log, current status) modeled — gdata oper layer, or actor-owned state with a `TreeProvider`?
