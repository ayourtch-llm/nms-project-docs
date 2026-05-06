# 03 - Codex Design Review R2

Fresh-reader review of:

1. `docs/00-orientation.md`
2. `docs/01-software-install-logic.md`
3. `docs/02-sw-install-design.md`

Then sanity-checked against:

- `stratoweave/stratoweave/src/stratoweave/ttt.act`
- `stratoweave/stratoweave/src/stratoweave/device.act`
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act`
- `stratoweave/sorespo/src/sorespo/`

I read the docs first before looking at prior round-1 review material. Round-1 integration mostly confirms why v2 moved in the directions it did; this review focuses on remaining blockers and design clarity gaps.

## Summary verdict

The doc set is much stronger than a typical pre-implementation design: the logic extraction is detailed enough to implement from, and v2 fixes several important semantic losses from the action-to-reactive mapping. The main architecture is directionally viable, but I would not start deep runner/device code until the following are resolved:

1. The transform-memory/dynstate split is internally inconsistent and likely wrong for the current `ttt.Transform` API.
2. The "one DeviceRunner is the install lease" claim does not actually protect the device from other `DeviceMgr.configure` or `rpc_xml` callers.
3. The proposed transform observer substrate is plausible, but the design places it at a path where it cannot naturally see both global pack config and all device assignments.
4. Cancel semantics contradict themselves and need a single observable contract.
5. Phase 4 NETCONF-only plus `NoopFileTransfer` is under-specified for SROS: `CheckFiles` and `CopyImage` still require a coherent local-vs-remote filename story.
6. The TextFSMPlus CLI section is too large and too committal for a deferred dependency; it reads like a second design doc embedded in this one.

## Findings

### High: transform `memory` vs `dynstate` is confused

`02` says transform `memory` holds `last_pack_data` and last-observed generations, then says the runner updates memory "via the transform's `update_dynstate` path" (`02` lines 125-140). In live `ttt.act`, memory and dynstate are separate persisted attributes:

- `transform_wrapper(...)` returns `(newout, new_memory)`, and `db_ops` stores `essay.1` as `ATTR_MEMORY`.
- `update_dynstate(...)` only updates `self.dynstate` and recomputes; it does not write `memory`.
- `finalize()` calls `on_conf(self.get(), self.memory)` after compute.

Reference: `ttt.act` lines 1942-1962, 1981-1993.

So a child actor cannot "update memory" through `update_dynstate` unless the transform's next `transform_wrapper` derives a new memory value from dynstate. The design needs one source of truth:

- Option A: put last observed generations and last materialized pack data in dynstate, not memory. Let `transform_wrapper` return stable/no-op memory.
- Option B: keep idempotency inputs in memory, but only mutate them in `transform_wrapper` from current config. Then the runner must not own those fields.

I recommend Option A. Request lifecycle, generation observations, request counters, backoff, current/history, and run-log are all operational state. Splitting them between transform memory and runner dynstate creates consistency bugs and restart ambiguity.

### High: a per-device actor is not an actual device lock

The design correctly notices that `DeviceMgr` mailbox serialization is not enough (`02` lines 448-450), but then claims the `DeviceRunner` actor's existence is the install lease (`02` lines 496-499). It is only a lease among sw-install's own code. It does not stop:

- RFS layer transforms from sending normal config through `DeviceMgr.configure`.
- Other platform code from calling `DeviceMgr.rpc_xml`.
- Subscription callbacks or other actors sharing the same adapter.

Live `DeviceMgr` exposes `rpc_xml` directly to the adapter (`device.act` lines 783-784) and has a normal config queue independent of sw-install. There is no lease or mutex API in `DeviceMgr` or `DeviceAdapter`.

The Python lock was stronger: `MaapiLocker.lock_partial(/devices/device[name=X])` serialized against all writers touching the device subtree. If sw-install needs equivalent safety, the design needs a platform-level device operation lease, for example:

- Add a `DeviceMgr.acquire_exclusive(owner, timeout, cb)` API that gates `configure`, `rpc_xml`, file transfer, and maybe `fetch_config`.
- Or explicitly downgrade the fidelity claim: sw-install serializes its own requests only, and external device operations may interleave.

Given software install is disruptive, I would not accept the downgrade silently. The design should make the platform-lock API a Phase 4 prerequisite or at least a clearly owned platform task.

### High: the proposed transform attachment point cannot see the required data

The module API says apps add YANG to the lowest config layer and wire a `Transform` at the per-device path (`02` lines 71-72). But trigger detection requires both:

- Global config: `/software-install/software-pack[]`, `/software-install/enabled`, policy, error handling.
- Per-device config: `/devices/device/software-pack/name/control/...`.

A transform attached under each device subtree will naturally receive only that subtree as `cfg`. A transform attached under `/software-install` sees global packs but not per-device assignments. A top-level observer would need to be wired at a common ancestor or use layer subscriptions/querying.

Sorespo's generated transforms are local to the list entry they are attached to; e.g. `t_2.act` wires `RFSTransform` under `/rfs[name]/backbone-interface[name]`, and the actor receives that entry's config (`t_2.act` lines 21-27). There is no precedent there for a transform that sees multiple disjoint roots.

The design needs an explicit wiring shape:

- A single top-level `software-install` transform that is given the whole layer root or can query the layer root.
- Or one per-device transform plus a separate global-pack cache/subscription.
- Or the fallback top-level actor becomes the primary architecture.

Right now the pseudocode under `make_sw_install_transform` is too abstract to prove the dataflow.

### High: observer `Transform` has a re-entrancy risk through `update_dynstate`

The current transform API supports `update_dynstate` from actor code, and sorespo proves a child actor can call typed wrappers that eventually call `params.update_dynstate` and `params.update_oper` (`sorespo/rfs.act` lines 189-213, `sorespo/layers/t_2.act` lines 21-27). So the assumption in `02` line 520 is probably feasible.

But live `ttt.act` implements `update_dynstate` by setting `self.dynstate` and immediately doing `Session(...).recompute()` (`ttt.act` lines 1981-1987). `finalize()` then calls `function.on_conf(...)` whenever `self.running` exists (`ttt.act` lines 1989-1993).

For sw-install, every runner state change may call `update_dynstate` (`02` lines 516-518). That can cause recompute and re-delivery of config into `on_conf` while the runner is already processing a run. The design has generation tokens for stale device callbacks, but it does not specify idempotency/re-entrancy discipline for repeated `on_conf` calls caused by the runner's own persistence writes.

At minimum, the design should state:

- `on_conf` must be a pure reconciliation pass against `(cfg, dynstate)` and never blindly starts a run.
- `update_dynstate` writes must not advance last-observed config generations unless the config trigger has actually been consumed.
- The runner should debounce/coalesce oper/dynstate publishing if every plan/log change causes an expensive recompute.

Without that, the observer-transform plan may self-trigger enough to duplicate starts, consume generations early, or churn LMDB.

### High: request history cannot be just `history: list[RequestState]` without retention semantics

The original request list is the durable audit/state surface. It is not described as bounded in `01`; only run-log clearing is explicit. In v2, `DeviceRunner` has `current` plus `history: list[RequestState]` "bounded" (`02` line 480), but the bound and archival semantics are not specified.

This matters because `create-request` obsoletes all previous requests and `last_request` drives idempotency/cancelled reactivation. If history is bounded, dropping the last cancelled request or the last materialized pack snapshot can change behavior. If history is persisted only through dynstate, the oper `request[]` list is only as durable as that dynstate snapshot.

Decide:

- Is `request[]` a full durable history, like the Python oper CDB model?
- Or is it intentionally bounded? If bounded, by count, age, terminal-only, or latest-N? What happens to `last-create-result` and idempotency when old entries are evicted?

I would keep at least the latest request plus all non-terminal/current entries unconditionally, and make any older-history pruning explicit and separate from idempotency state.

### Medium: cancel semantics are contradictory

`02` line 193 says the runner sets request status to `cancelled` immediately, but the user observes `cancelled` once the in-flight RPC returns. `02` lines 524-529 again say status flips immediately, while logs may continue later.

Pick one observable contract:

- Preferred: status becomes `cancelling` immediately, then `cancelled` at a step boundary/RPC return. This requires adding `cancelling` to the enum.
- Or status becomes `cancelled` immediately, and all late callbacks/logs are suppressed or tagged as ignored.

The current text combines both. For operators, this is not cosmetic: `cancelled` usually means the workflow is no longer active. If the device can still be mid-reboot or mid-install RPC for 600 seconds, the oper state should show that honestly.

### Medium: `enabled=false` semantics are unresolved in the prose

The `enabled` section literally contains "wait, actually let's pick a distinct status" (`02` line 209). The proposed `paused` enum is reasonable, but the transition rules are incomplete:

- If `enabled=false` while a run is waiting on backoff, is `next_wake_at` retained and status changed to `paused`?
- On re-enable with explicit-start mode, does an already-started paused request require another `start-generation`, or does the existing start intent remain valid?
- If disabled before any start, is the status `unprocessed` or `paused`?
- Does disabling bump the generation token and stale out in-flight callbacks?

This should be tightened before implementation. I would separate `execution_allowed` from `request.status`: disabled before start stays `unprocessed`; disabled during processing moves to `paused`; re-enable resumes only if a prior start intent was already consumed for that request.

### Medium: `confirm-steps` override disappeared from the writable surface

The Python `request.confirm-steps` is oper/request-local but can override the global policy according to the logic doc. V2 moves all user inputs out of `request[]`, which is good, but it does not add a replacement for per-request `confirm-steps` override under `control/`.

`RequestState.confirm_steps` exists (`02` line 460), yet there is no config input that populates it except global policy. If per-request override is intentionally dropped, list it in the YANG diff/deferred section. If preserved, add something like:

- `control/confirm-steps?` sampled when materializing a request, or
- `control/request-options/confirm-steps?` with clear snapshot timing.

### Medium: `confirm-all-generation` needs request scoping

The design says incrementing `confirm-all-generation` confirms every step of every component in the latest request (`02` lines 187, 118). Because generation leaves are not keyed by request, a delayed or replayed config write could apply to a newer request than the operator inspected.

Generation counters avoid replay edges, but they do not bind intent to a specific request id. Same concern applies to `start-generation`, `cancel-generation`, and `clear-run-log-generation` if a new request is created between the read and write.

Consider adding optional target leaves next to each trigger:

- `control/target-request-id`
- or per-trigger containers `{ generation, request-id }`

At minimum, define that triggers always apply to the latest request at observation time and warn external automation that read-modify-write races are possible. For a package porting explicit actions on a request node, I would preserve request scoping.

### Medium: backoff formula is subtly incomplete

`01` says Python factor mode uses `(request.error_count.backoff or 10) * factor`; constant mode uses configured constant. `02` says `factor * max(error_count.backoff, 10)` (`02` line 539).

Those are equivalent only if backoff is never below 10 after being set. If factor can be less than 1.0 or if state is manually restored/edited, they differ. More importantly, v2 no longer models `error_count.backoff` in oper but still calls it actor-private. If the port intentionally hides it from YANG, then troubleshooting retry behavior becomes worse than Python.

Recommendation:

- Match the exact formula: `(backoff if present else 10) * factor`.
- Keep `error-count/backoff` in the oper projection even if the source of truth is dynstate. Operators need to see why the next retry is delayed.
- Add `next-wake-at` to oper. It is too useful to keep internal-only.

### Medium: Phase 4 NETCONF-only filename semantics are not coherent enough

`02` says Phase 4 ships `NoopFileTransfer`; `CopyImage.pre_check` can return `SKIP_STEP` if all files are already present via `stat`/`list`, or `FAILURE` if missing (`02` lines 593-598). But `NoopFileTransfer` methods return `NotImplementedError`, so there is no way to verify "already present" through that interface.

`02` line 632 says NETCONF-only Phase 4 requires the file already be on the device at the given path. That is a major semantic shift from Python, where `filename` is a controller/NSO-host path and SROS `CheckFiles` validates local filesystem presence (`01` SROS CheckFiles). If Phase 4 treats filenames as remote paths, then:

- What does `CheckFiles` check?
- How are `destination_paths` populated for later steps?
- Does SROS `CopyImage` become a pure "verify remote files" step using NETCONF state, not `FileTransfer`?
- How do size/checksum checks work without a real `FileTransfer.stat`?

This can be fixed by splitting the interfaces:

- `ControllerFileStore` for local path existence/archive parsing.
- `DeviceFileStore` or `DeviceOps` methods for remote file stat/checksum/free-space.
- `FileTransfer` only for moving bytes.

For Phase 4, make `CopyImage` a remote verification/no-op step using NETCONF file state, and state that local `CheckFiles` is disabled or reinterpreted. Do not hide this behind `NoopFileTransfer`.

### Medium: preserving `scp-port` under the software-pack subtree changes the model

Original YANG augments `/devices/device/scp-port`, not `/devices/device/software-pack/scp-port` (`software-install.yang` around the per-device augment). V2 moves it into the `software-pack` presence container (`02` lines 616-628).

That means removing the software-pack association also removes the scp port setting, and any future non-sw-install SCP use cannot share it. Maybe that is fine, but it is not "preserve it" in the strict sense.

Recommendation: either keep it as a direct device augmentation if the platform schema allows it, or explicitly document that the port narrows it to sw-install-only metadata.

### Medium: TextFSMPlus is over-designed for the current phase and under-owned

The CLI section (`02` lines 634-748) is detailed and compelling as a future direction, but it creates several design liabilities:

- It commits to TextFSMPlus plus an aycalc-equivalent expression evaluator before any Acton port exists.
- It says CLI strategy code paths are typed and present in Phase 4 but raise `NotImplementedError`. That is surface area without executable value.
- It treats interactive CLI as a template problem, but does not address prompt synchronization, paging, privilege modes, terminal width, command echo, or secrets in presets/logs.
- It puts a substantial acton-utils platform dependency inside the sw-install design while also saying the workstream is separate.

For Phase 4, I would keep only `DeviceOps` as the abstraction boundary and move TextFSMPlus to a short deferred ADR. The step signatures do not need to know whether Phase 5 uses TextFSMPlus, Expect-like primitives, or netcli-native helpers. The current section risks distracting implementation and review from the NETCONF path.

### Medium: `DeviceOps` strategy selection depends on capabilities that may be too stale

The logic doc separates pack OS from device capability-derived OS. V2 says `SrosOps` decides strategy based on `dev.get_capabilities()` / `dev.get_modules()`. In live `DeviceMgr`, `get_modules()` returns the last known module set and id, and `get_capabilities()` delegates to adapter state. If the device reconnects or schema changes mid-run, strategy selection and `ValidatePlatform` behavior could drift.

The design should state whether platform validation is:

- Snapshotted at request/run start and stable for the run, or
- Re-evaluated before every step, with a capability change failing the request.

Python `ValidatePlatform` effectively validates during the plan. For a long-running actor with persistent state, I would snapshot per run but fail if the actual capability set changes incompatibly across retries.

### Medium: generated YANG integration is underspecified for a sibling library

The orientation explains per-app `gen_adata`; the design says `sw-install` exports raw YANG and apps add it to their lowest layer. But the design also proposes a sibling repo with its own `gen_adata` and generated `model.act`.

There are two possible generated-type worlds:

- The library owns generated classes for its YANG and apps import those.
- Each app regenerates a combined layer schema, producing app-local generated classes for the same YANG.

Sorespo's `layers/t_*.act` and `sysspec.act` are generated per app. If `sw-install` is a library transform embedded in another app's generated layer, type identity and import paths matter. The design should specify how `make_sw_install_transform` receives typed config:

- Raw `gdata.Node` parsed by sw-install's own generated `model.act`?
- App-generated accessors passed through wrapper code?
- A shared `sw_install.yang` module compiled once and reused?

This is not just packaging; it decides the public API and test harness shape.

### Low: action return value replacement is still weaker than actions

`last-create-result` is useful, but it is per-device and "last" only. If two clients write pack/control config concurrently, one can observe the other's result. Existing action output was per call.

If this matters, include a client-supplied correlation id in `control/request-correlation-id` and echo it in `last-create-result`. If not, document that `last-create-result` is a convenience/status leaf, not a call-correlated result.

### Low: run-log retention changes fidelity

The Python run-log has no automatic retention; v2 caps it at 1000 entries per request. I agree with adding a cap for operational sanity, but it is a semantic change and should be in the YANG diff/deferred/fidelity notes, including whether dropped count/oldest timestamp is exposed.

### Low: exact time/key uniqueness needs a note

Python uses `yang:date-and-time` as the `run-log` key with microsecond precision. If Acton timestamps collide within one actor tick or after restore, what happens? The design can avoid this by adding a sequence number to internal `RunLogEntry` and deriving unique keys, or by keying run-log on `(run-id, seq)`.

### Low: orientation is good, but it hides the hardest integration question

`00` gives a useful conceptual map. As a fresh reader, I had enough context to understand the project. The blind spot is that it presents transforms as "pure-ish functions from above to below" and only later says operational state is a separate axis. Since sw-install is mostly an operational control loop, the orientation should include one paragraph on `update_oper` / `update_dynstate` and the sorespo `BBInterfaceTransform` pattern. That would make `02` less surprising.

### Low: logic spec is strong, with two useful additions

`01` is the strongest doc. It is implementation-grade. I would add:

- A compact table of "Python behavior intentionally changed by Acton port" once `02` decisions settle.
- A clearer statement around `enabled`: the original YANG description says disabled means every step needs confirmation, but v2 gives it a stronger "no execution starts" meaning. That difference should be explicit.

## Focused answers to requested areas

### Section 3: config/oper boundary and where state lives

The config/oper split is directionally right: user inputs under a writable `control/` sibling, request/plan/run-log as oper. The remaining issue is not config vs oper; it is persistence source of truth. Do not split lifecycle state between transform `memory`, transform `dynstate`, actor private fields, and oper projection without an explicit ownership rule.

Recommended ownership:

- Config: desired pack assignment and control generations/confirmations only.
- Dynstate: all durable lifecycle state and consumed generation counters.
- Oper: pure projection of dynstate plus current computed view.
- Transform memory: avoid for sw-install unless used for a narrow, transform-local optimization.

### Section 4: generation-counter replacement for actions

Generation counters are a good reactive substitute, but the design needs request scoping. Explicit actions were invoked on a specific request node; global per-device counters apply to "latest at observation time." That is a race-prone semantic regression for automation.

Add target request ids or define the race explicitly. Also consider a client correlation id for `last-create-result`.

### Section 7: Transform observer vs top-level actor

Based on live code, `Transform` can provide `update_oper` and `update_dynstate`, and sorespo proves child actors can use those callbacks. So the transform path is technically plausible.

However, because sw-install needs a multi-root config view and does not emit downward config, the fallback actor is not just a fallback; it may be architecturally cleaner. If using `Transform`, wire it as a single top-level observer with a documented full config view, not as an opaque per-device transform.

### Section 8: DeviceRunner, generation tokens, restart, cancel, backoff

The per-device runner is the right unit for sw-install's own serialization. Generation tokens are the right stale-callback mechanism. Persistent `next_wake_at` is the right restart fix.

But it is not a platform-level device lease. Add a real `DeviceMgr` exclusive operation API or explicitly accept interleaving with normal configuration. Also fix cancel status (`cancelling` vs immediate `cancelled`) and re-entrancy from `update_dynstate`.

### Section 9: FileTransfer, credential reuse, NETCONF-only Phase 4

Rejecting `DeviceAdapter.get_credentials()` is the right instinct. Piggybacking on the NETCONF SSH transport is probably the cleanest Phase 5 path if the netconf package can expose it cleanly.

Phase 4 still needs a coherent file model. `NoopFileTransfer` cannot both return `NotImplementedError` and verify pre-staged files. Split remote file inspection from byte transfer, or make the SROS Phase 4 copy step explicitly fail unless a mock/real remote stat provider exists.

### Section 9.7: CLI via TextFSMPlus

The strategy boundary (`DeviceOps`) is worth keeping now. The TextFSMPlus commitment should be moved out of the main Phase 4 design. It is too specific, too large, and has unresolved platform ownership. Keep it as a deferred ADR with enough detail to preserve the idea.

## Onboarding doc quality

`00-orientation.md` succeeds as a fresh-reader entry point. It gives vocabulary, repo map, StratoWeave concepts, and the NSO package shape without requiring immediate source spelunking. The most useful improvement would be a short "oper/dynstate patterns" subsection, because sw-install relies more on those than on normal downward transforms.

`01-software-install-logic.md` is detailed enough to be the porting contract. It clearly preserves the odd but important behavior: consecutive retry counters, refresh-after-step, `next_step` jumps, Junos parametrized steps, and state reset. I would treat it as the source-of-truth spec.

`02-sw-install-design.md` is readable and much improved, but it still mixes settled design, assumptions, and open questions. Before coding deep runner logic, convert the open assumptions around transform substrate, memory/dynstate ownership, real device locking, cancel status, and Phase 4 file semantics into explicit decisions.

## Recommended next edits before implementation

1. Rewrite §3.2 and §8.4 around a single dynstate ownership model; remove the claim that `update_dynstate` updates transform memory.
2. Add a real device-operation lease requirement or downgrade the fidelity claim in §8.1-§8.2.
3. Replace the integration sketch in §2/§7 with an exact wiring topology showing how global config and per-device config reach `SwInstallRunner`.
4. Fix cancel and enabled status semantics with state transition tables.
5. Add request-id scoping for start/cancel/confirm-all/clear-log triggers.
6. Clarify Phase 4 file semantics by separating local file checks, remote file inspection, and transfer.
7. Move most of §9.7 into a deferred ADR; keep only `DeviceOps` in the Phase 4 design.
8. Add generated-YANG/type-boundary details for sibling repo integration.

After those changes, I would be comfortable starting Phase 4 skeleton/types/plan tests while keeping platform-lock and FileTransfer implementation behind explicit interfaces.
