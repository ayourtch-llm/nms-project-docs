# 05 — Integration of Round-2 Design Reviews

Consolidates the round-2 reviews:

- `docs/reviews/03-codex-design-r2.md` — Codex (pty-2)
- `docs/reviews/04-claude-design-r2.md` — Claude (pty-3)

Records (a) the high-priority items both reviewers reached independently, (b) unique additions from each, (c) my decisions, (d) action items flowing into the design revision.

Tone note: both reviewers delivered substantive, code-grounded critique. Several catches saved real implementation work — particularly Claude r2's discovery that **option (c) "piggyback on netconf SSH transport" is impossible as I described it** (the ssh_client wraps an OpenSSH subprocess, not an in-process SSH library) and Codex r2's clean diagnosis of the **memory-vs-dynstate plumbing impossibility**. Independent convergence on six high-priority structural issues is the strongest signal: those aren't subjective.

---

## Headline takeaway

**Don't start §6/§8 implementation code yet.** Six blocking issues converge across both reviews. Two of them (A1 memory/dynstate, A4 re-entrancy) are correctness-shaped and require design decisions the v2 doc didn't make. Two more (A2 device lease, A3 wiring topology) require choosing among real alternatives, not just clarifying. The remaining two (A5 cancel/enabled, A6 file semantics) need state-machine-level precision.

The architectural direction (reactive triggers + DeviceRunner per device + Transform substrate) remains sound — the items below are gaps inside that direction, not arguments against it. v3 of the design doc tightens around them.

---

## Convergence matrix — six HIGH items both reviewers raised independently

| # | Issue | Source citations | Decision |
|---|-------|-----------------|----------|
| **A1** | Transform `memory` vs `update_dynstate` plumbing is incompatible. The runner has no path to write `memory`; only `transform_wrapper`'s return value updates it. v2 §3.2 / §7.1 say the runner updates memory via `update_dynstate` — that's not implementable against the platform. | Codex r2 "high: transform memory vs dynstate is confused"; Claude r2 §A1 | **Accept fully. Drop transform `memory` from sw-install entirely.** All operational state — last-observed pack data, last-observed generation counters, request id counter, plan, per-component State, error_count, next_wake_at, run-log — lives in dynstate. `transform_wrapper` returns `(empty, memory)` unchanged. Rewrite §3.2, §7.1. |
| **A2** | "DeviceRunner's existence is the install lease" claim is false. DeviceMgr exposes `configure(...)` (used by RFS), `rpc_xml(...)` (anyone), `fetch_config(...)`, `declare_subscriptions(...)` — none gate on a sw-install lease. The Python `MaapiLocker.lock_partial` was system-wide; v2's per-device runner is sw-install-internal only. For an OS upgrade this is a real safety gap. | Codex r2 "high: a per-device actor is not an actual device lock"; Claude r2 §A2 | **Honest downgrade for v1.** Adding `DeviceMgr.acquire_exclusive(owner, timeout)` is a platform commitment that needs its own conversation; sw-install shouldn't presume that decision. Document the gap clearly. Operators must ensure RFS layer isn't actively reconciling against a device under upgrade. **Add a "Platform prerequisites for v2.0" subsection** flagging the lease API as the upgrade path. |
| **A3** | Wiring topology is ambiguous. v2 §1/§2 say "wire a Transform at the per-device path" (= per-device transform per sorespo's pattern); §8.7 says SwInstallRunner owns `dict[device_name, DeviceRunner]` (= singleton over the whole tree). These are inconsistent. Per-device transforms can't naturally see global `/software-install/` config. | Codex r2 "high: proposed transform attachment point cannot see the required data"; Claude r2 §A3 (offers three concrete options A/B/C) | **Pick Option B: per-device transform + global subscription.** Each per-device transform owns its `DeviceRunner` (no `SwInstallRunner` singleton). Global config (`/software-install/...`) reaches each transform via `Layer.declare_subscriptions(...)` against the lower layer — existing platform mechanism (`ttt.act:735`, `LayerTreeProvider`). This matches sorespo's per-list-entry grain *and* gives each device its own isolated dynstate slice (no cross-device contention). Cleanest available shape; rewrite §1, §2, §7.1, §8. |
| **A4** | `update_dynstate` writes trigger `Session.recompute()` and re-entry into `on_conf`. v2 has stale-step-callback discipline (generation tokens — §8.3) but no stale-config-trigger discipline. Naive impl can self-trigger duplicate starts, double-consume generations, churn LMDB. | Codex r2 "high: observer Transform has a re-entrancy risk through update_dynstate"; Claude r2 §A4 | **Accept. Add three invariants to §8:** (1) `on_conf(cfg, dynstate)` is a pure idempotent reconciliation function — never starts work as a side effect; decides what should be in flight and dispatches if not already; (2) generation observations are persisted to dynstate **before** any work begins for that generation, so a crash leaves the trigger consumed (no replay); (3) high-frequency state writes (run-log entries, install-op-id polling) coalesce — write on state-class transition only, or via a periodic flush. |
| **A5** | Cancel and `enabled` semantics contradict themselves. §4.4/§8.5 say status flips to `cancelled` immediately AND the user observes `cancelled` once the in-flight RPC returns. §4.6 contains a literal mid-edit comment "wait, actually let's pick a distinct status." | Codex r2 "medium: cancel semantics are contradictory" / "medium: enabled=false semantics are unresolved"; Claude r2 §A5 (recommends adding `cancelling` to enum) | **Accept both fixes.** Add **`cancelling`** to the request-status enum: cancel-generation increment → status becomes `cancelling` immediately + generation token bumps + in-flight callbacks become no-ops; when last in-flight RPC drains → status becomes `cancelled`. Honest about what the operator sees. For `enabled`: rewrite §4.6 with a state-transition table covering (enabled toggle × current request status × explicit-start gate). Finalize `paused` as the in-flight stop status. |
| **A6** | Phase 4 `NoopFileTransfer` cannot coexist with `CopyImage.pre_check`'s SKIP_STEP-if-files-present path. NoopFileTransfer.stat returns NotImplementedError; pre_check needs working stat. Either Phase 4 SROS doesn't run at all, or NoopFileTransfer isn't a Noop. | Codex r2 "medium: Phase 4 NETCONF-only filename semantics are not coherent enough"; Claude r2 §A6 | **Accept the three-way split:** `LocalFileInspector` (controller-side filesystem; Acton stdlib) for `CheckFiles`; `RemoteFileInspector` (per-OS NETCONF queries: SROS uses `oper-file` / `file dir`) for `CopyImage.pre_check`; `FileTransfer` (Phase 5 byte-mover) for `CopyImage.execute`. Phase 4 SROS gets a **real `RemoteFileInspector`** over NETCONF + `NoopFileTransfer` for byte transfer. `CopyImage.execute` returns `FAILURE` with clear "no FileTransfer configured — pre-stage the image" if files missing; SKIP_STEP if all present (verified via RemoteFileInspector). |

---

## Codex-r2-unique catches

| # | Issue | Decision |
|---|-------|----------|
| **CR1** | Request `history: list[RequestState]` retention is undefined. Last cancelled request drives idempotency; pruning it changes behavior. | Accept. Define: keep latest non-terminal request unconditionally + latest of each terminal status (cancelled/done/failed-other/failed-transient/obsolete). Older history pruned when total > 50 (configurable). Pruning is independent from idempotency state. |
| **CR2** | Backoff formula slip: v2 says `factor * max(error_count.backoff, 10)`; spec says `(error_count.backoff or 10) * factor`. Differ when factor < 1.0 or state is restored. | Accept. Match the spec exactly. |
| **CR3** | `error_count.backoff` and `next_wake_at` should be in oper too — operators need to see why the next retry is delayed. | Accept. Project both into the request oper subtree. |
| **CR4** | `enabled` semantics nuances need state-transition rules (mid-backoff toggle, before-start toggle, generation-token bumping on disable). | Accept; covered by the §4.6 state-transition table fix in A5. |
| **CR5** | Per-request `confirm-steps` override (Python had `request/confirm-steps?`) is missing from v2's control surface. | Accept. Add `control/request-options/confirm-steps?` sampled at request materialization time. |
| **CR6** | DeviceOps strategy selection on stale capabilities. | Accept. Snapshot per run; fail clean on incompatible drift. |
| **CR7** | Generated-YANG/type-boundary details for sibling repo integration (does sw-install own generated classes, or does each app regenerate?). | Accept. Document: sw-install ships its own generated `model.act`; apps that compose sw-install YANG into their own layer get type-compatible accessors via the canonical YANG namespace. |
| **CR8** | `last-create-result` is per-device, last-writer-wins, not per-call. Concurrent clients race. | Accept partial: document the limitation in §4.1. Adding a client correlation id leaf is a future enhancement; not blocking. |
| **CR9** | Run-log retention semantic change should be in fidelity-deviations notes. | Accept; covered by the new conscious-deviations subsection. |
| **CR10** | Run-log key collision risk (microsecond timestamp). | Accept. Add a `seq` leaf; key on `(when, seq)`. |

---

## Claude-r2-unique catches

| # | Issue | Decision |
|---|-------|----------|
| **CL_R2_1** | **Option (c) "piggyback on netconf SSH transport" is impossible as described.** `ssh_client.act` wraps an OpenSSH subprocess; channel multiplexing requires ControlMaster/ControlPath plumbing or in-process SSH lib. The proposed (c) collapses into (b) (DMC at factory) anyway. | **Accept fully.** Drop the (a)/(c) framing. Plumb `DeviceMgr.get_dmc()` as a one-line platform addition (DMC is owned by DeviceMgr, not the adapter — no credential leakage beyond what already exists). FileTransfer factory takes `(DeviceMgr, DeviceMetaConfig)`. **Read DMC at use time, not construction time** — DMC is mutable via `set_dmc(...)` which is called repeatedly. Rewrite §9.4. |
| **CL_R2_2** | **`internal-state` is no longer NETCONF/RESTCONF-visible** (operability regression). Operator debugging now requires controller-side lmdb access. | Accept. **Project selected dynstate fields** (`destination_volume`, `destination_paths`, `boot_time`, `op_id_*`, etc.) into the per-component oper subtree as read-only diagnostic leaves. Documented diagnostic surface, not opaque blob. |
| **CL_R2_3** | yang.act in `stratoweave/sw-install/` is **pre-v2** — lacks `paused` enum, control subtree, last-create-result, per-device augment, scp-port. | Accept. Update yang.act as part of v3 design closure (this isn't "implementation code" — it's the design artifact in YANG form, parallel to the design doc). |
| **CL_R2_4** | §14 placeholder stub (`## 14. (deleted from v2 ...)`) and §4.6 mid-edit prose ("wait, actually let's pick a distinct status") are unfinished doc remnants. | Accept. Clean up both in v3. |
| **CL_R2_5** | `scp-port` placement under `software-pack/` means removing the pack association also removes the scp-port. | Accept. Move `scp-port` to direct device augmentation (or to stratoweave device meta config, since it's really an SSH transport setting). Rewrite §9.5. |
| **CL_R2_6** | Generation counter overflow on backup/restore — restore makes counter go backward relative to runner's persisted last-observed; trigger then evaluates false forever. | Accept. Document explicitly. If platform exposes a "config restore" event, reset all `last_observed_*` counters at restore time; otherwise document as a known gotcha. |
| **CL_R2_7** | `next_step()` jump target not in plan — Python raises in `refresh_steps` (regression guard). Acton should match with FAILURE + clear log. | Accept. Document in §6.1. |
| **CL_R2_8** | Step `pre_check`/`execute` callback discipline — `cb` mailbox must be the per-device runner, not the step itself, for generation-token check to gate stale state. | Accept. Document as an §6 invariant. |
| **CL_R2_9** | Spawn-vs-inject restart story: on restart, what if a request was mid-`processing`? Python's per-step bookkeeping gives a clean resume; Acton port should specify. | Accept. **Decision:** on restart, any request in `processing` becomes `failed-transient` so the scheduler triggers a re-run; the re-run's `pre_check` is responsible for idempotent resume (already a spec invariant — file-already-copied skip, op_id_* check). |
| **CL_R2_10** | Device-not-in-registry case (DeviceMgr without DMC, NoAdapter active) — what happens? | Accept. Add a `waiting-for-device` request-status; the runner pauses until DMC+adapter become available. |
| **CL_R2_11** | request-id counter persistence — must be a separate dynstate field, not derived from `max(request[].id) + 1` (wrong if old requests are pruned per CR1). | Accept. Make explicit in §8. |
| **CL_R2_12** | `software-install-matrix` table entry "drop, log in §15" reads ambiguously. | Accept. Disambiguate: dropped from YANG; not modelled in Acton; re-add only if a real use case surfaces. |

---

## Where both reviewers push back on me, and I partially accept

### §9.7 CLI / TextFSMPlus — both reviewers say it's overcommitted for Phase 4

**Codex r2:** "It is too specific, too large, and has unresolved platform ownership. Keep it as a deferred ADR with enough detail to preserve the idea."

**Claude r2:** "The DeviceOps facade is worth keeping. Everything else — TextFSMPlus templates, Send/Preset/Done line actions, the aycalc-equivalent expression evaluator, the acton-utils textfsm extension, the prompt synchronization story, the secrets-in-presets-and-logs story — should be in a separate ADR. The current §9.7 is ~110 lines of design for a Phase 5 dependency that doesn't exist yet."

**Context:** Andrew explicitly authorized including CLI design now, with the rationale that "we're already iterating; the marginal review cost is small compared to a separate Phase 5 review cycle."

**Decision (partial accept):** the substance both reviewers and Andrew want is real. The architectural touchpoint (`DeviceOps` as the strategy boundary) **stays in §9** of the design doc — that's the part that actually constrains Phase 4 step signatures. Everything else (TextFSMPlus templates, Send/Preset/Done semantics, aycalc-equivalent expression eval, Phase-5 implementation details, the acton-utils textfsm extension dependency) **moves to `docs/adr/cli-driver.md`** — a sibling decision-record that captures the intent without bloating the Phase 4 design.

This satisfies all three voices: Andrew gets the CLI architecture committed to before Phase 4 implementation; reviewers get a clean Phase-4-shaped design doc; the Phase 5 design lives in its own document and can iterate independently.

---

## Decisions on architectural questions left open in v2

### Q1 (was: Transform substrate vs top-level actor)

**Resolved by A3.** Use `Transform`, but per-device (Option B from Claude r2) — **not** as a top-level multi-root observer. Each per-device transform owns its `DeviceRunner` and its dynstate slice. Global config arrives via `Layer.declare_subscriptions`. No `SwInstallRunner` singleton.

The §7.2 fallback ("top-level actor + TreeProvider") is no longer "fallback if Transform proves awkward" — it's parked as an alternative if Option B's subscription mechanism turns out incompatible with how the per-device transform's actor sees `Layer` references. Likely fine; verify during implementation but don't pre-design around it.

### Q2 (was: update_dynstate from a child actor of the transform's act-spawned root)

**Resolved by A3.** No more child-actor-of-coordinator hierarchy. The per-device transform's own `act`-spawned actor *is* the DeviceRunner. `update_dynstate` is called directly from the runner against the same transform whose dynstate plumbing it has via `TransformActorParams`. No bubbling.

### Q3 (was: default value of `auto-execute-after-confirm`)

Unchanged: `false` (operator-explicit start, matches Python).

### Q4 (was: credential reuse path — a/b/c)

**Resolved by CL_R2_1.** Drop (a)/(c). Plumb `DeviceMgr.get_dmc()`. FileTransfer factory takes `(DeviceMgr, DeviceMetaConfig)`. Read DMC at use time.

### Q5 (was: run-log default bound)

Unchanged: 1000 entries/request. Add `dropped-count` and `oldest-when` oper fields per CR9.

### Q6 (was: who owns the acton-utils textfsm extension)

**Deferred to the ADR.** Sw-install consumes it; ownership of the extension itself is platform-side, captured in `docs/adr/cli-driver.md`.

---

## Action items flowing into v3 design revision

In rough priority order (matches both reviewers' priority lists):

1. **Drop transform `memory` from sw-install.** Rewrite §3.2 and §7.1. All state in dynstate.
2. **Pick wiring topology: Option B (per-device transform + global subscription).** Rewrite §1, §2, §7.1, §8 — including removing `SwInstallRunner` as a coordinator.
3. **Honest device-lease downgrade.** Rewrite §8.1–§8.2. Add "Platform prerequisites for v2.0" subsection flagging `DeviceMgr.acquire_exclusive(...)` as the upgrade path.
4. **`on_conf` re-entry discipline.** Add three invariants to §8 (idempotent reconciliation; durable generation observation before work; coalesced high-frequency writes).
5. **Cancel state machine** — add `cancelling` to enum; rewrite §4.4 and §8.5 with a state-transition table.
6. **`enabled` state machine** — rewrite §4.6 with a transition table covering all the corner cases.
7. **Three-way file abstraction split** — rewrite §9.2/§9.3/§9.6 around `LocalFileInspector` + `RemoteFileInspector` + `FileTransfer`.
8. **`DeviceMgr.get_dmc()`** — rewrite §9.4. Drop (a)/(c) framing.
9. **Per-request scoping** — add optional `target-request-id` leaf next to start/cancel/confirm-all/clear-run-log triggers.
10. **Move §9.7 details to `docs/adr/cli-driver.md`.** Keep `DeviceOps` facade in §9. Reference the ADR.
11. **Add §15.5 "Conscious deviations from the Python spec"** — list all 10 fidelity-vs-operability tradeoffs (internal-state visibility, bounded run-log, cancel latency, lease scope, generation backup-restore, …).
12. **Add `00-orientation.md` subsection on `update_oper`/`update_dynstate`/the BBInterfaceTransform observer pattern.** Both reviewers agree this is the highest-leverage onboarding edit.
13. **Diagnostic projection of dynstate** — surface `destination_volume`, `destination_paths`, `boot_time`, `op_id_*`, `error_count.backoff`, `next_wake_at` in oper for operator debugging.
14. **Clean unfinished prose** — §4.6 mid-edit, §14 stub.
15. **Update `stratoweave/sw-install/src/sw_install/yang.act`** to v3 (control subtree, per-device augment, paused enum, last-create-result, target-request-id leaves, scp-port location fix, etc.).

Plus the smaller items: backoff formula fix (CR2), error_count.backoff/next_wake_at in oper (CR3), per-request confirm-steps override (CR5), DeviceOps capability snapshot per run (CR6), generated-YANG ownership (CR7), run-log key uniqueness (CR10), generation counter restore semantics (CL_R2_6), next_step jump target not in plan (CL_R2_7), step callback discipline (CL_R2_8), processing-mid-restart behavior (CL_R2_9), waiting-for-device status (CL_R2_10), request-id counter persistence (CL_R2_11), software-install-matrix language (CL_R2_12), scp-port placement (CL_R2_5), CL_R2_2 (internal-state diagnostic projection — already in #13).

---

## What stays unchanged from v2 (both reviewers explicitly endorsed)

Per Claude r2 §G:

- The `control/` config / `request[]` oper split (A5 in round-1 integration).
- Generation counters as the trigger mechanism.
- Per-device runner as the unit of step serialization (within sw-install) — *with* the lease caveat.
- Per-class retry budgeting (`error_count.transient` / `error_count.other` consecutive, with reset rules).
- Plan refresh monotonicity + after-every-step trigger.
- Generation tokens for stale step callbacks.
- Dropping NSO action nodes for reactive triggers, with the noted ergonomic patches (last-create-result, confirm-all-generation, clear-run-log-generation).
- Junos `(name, ?re_id)` step keying.
- NETCONF-only Phase 4 scope (assuming A6 is fixed).

These survive v3 unmodified.

---

## Round-3 expectation

After v3 lands:

1. New ADR at `docs/adr/cli-driver.md` containing the §9.7 content.
2. Revised `docs/02-sw-install-design.md` addressing the 15-item list above.
3. Revised `docs/00-orientation.md` with the operational-state subsection.
4. Updated `stratoweave/sw-install/src/sw_install/yang.act` reflecting v3.
5. `/clear` both reviewers; brief them against the full revised doc set; expect a much shorter round 3 (probably 5–10 items each, mostly clarity polish if v3 is solid).

If round 3 surfaces another batch of HIGH items, the docs aren't yet self-contained and another round follows. Quality bar is "both reviewers no longer have HIGH or MEDIUM items," not "round 3 is the last round."
