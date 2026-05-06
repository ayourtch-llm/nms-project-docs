# 03 — Integration of Round-1 Design Reviews

This document consolidates the two parallel reviews of `docs/02-sw-install-design.md`:

- `01-codex-design.md` — Codex (pty-2)
- `02-claude-design.md` — Claude (pty-3)

It records (a) where the reviews agree, (b) where each caught something unique, (c) my decisions on each item, and (d) the action items that flow into the design doc revision and into the round-2 brief.

Tone note up front: both reviews are excellent. Codex landed clean structural critiques and called out the per-device install-lease gap that I genuinely missed. The pty-3 Claude review went further into the code (`device_meta_config.act`, `netconf/ssh_client.act`, `_send_config`'s confq pipeline) and surfaced two things I would have hit during implementation and had to redesign around — the credential-reuse hole and the action-return-values regression. Several of the catches below are specifically thanks to one or the other reviewer; I've credited the unique-source items inline.

---

## Headline takeaway

**Reactive design direction: confirmed.** Both reviewers endorse dropping the NSO action nodes and using stratoweave's transform/actor/`update_oper` mechanism as the substrate. The pushback is uniformly "don't lose specific semantics in the substitution," not "the substitution is wrong."

**Holding-implementation verdict: confirmed.** Both reviewers explicitly say the listed items should be settled before runner code is written. Two are correctness-shaped (per-device lease; retry status mapping); the rest are spec-fidelity items that change the design doc but not the implementation order.

---

## Agreement matrix (where both reviewers converged)

These are the items both reviewers raised independently. High confidence — accepting in full.

| # | Issue | Action |
|---|-------|--------|
| **A1** | **Cancelled-request reactivation is load-bearing** and pure pack-change-only triggers drop it. Need an explicit operator-driven "kick a fresh request" mechanism. | Accept. Add config control surface (see A2). |
| **A2** | **Need a config control subtree** (separate from the oper request subtree) for operator inputs. Codex proposed `request-generation`, `start-generation`, `cancel-generation`, `clear-run-log-generation` as `uint64` increment-to-trigger leaves. Claude (item 4) wants a `clear-run-log-requested` leaf for forensic clearing between runs (separate concern from auto-trim). | Accept the generation-counter pattern (it survives replay; edge booleans don't). Adopt `clear-run-log-generation` for explicit clears in addition to bounded retention. |
| **A3** | **Per-device install lease is missing.** `DeviceMgr`'s actor-mailbox serialization gives only message-level mutex; a `RequestRunner` with an outstanding `rpc_xml` does not block other RFS configures or unrelated rpc_xml from interleaving. The Python `MaapiLocker.lock_partial` was full-run mutex. | Accept. Required design change (see decision D1). |
| **A4** | **Retry status mapping bug** — counters are *consecutive*, not cumulative. WAIT-exhaustion ends as `failed-transient`, FAILURE-exhaustion ends as `failed-other`. My `transient + other > max_retries` check drifts from spec; reset rules (FAILURE → transient=0, WAIT → other=0) are also load-bearing. | Accept. Fix in design §8. |
| **A5** | **Config/oper boundary** — the original `request[]` is `config false`. Putting writeable `confirmed` under it mixes axes; YANG/RESTCONF behaviour gets awkward. | Accept. Move user inputs (control generations, confirmations, optional per-request overrides) into the config control subtree; keep request/plan/run-log purely operational. |
| **A6** | **Transform `memory` plumbing for idempotency** — my `transform_wrapper` returning `(Container, memory)` doesn't update memory. Without writing back, "is this pack-data the same as last time?" loses across restart. | Accept. Spell out: transform memory holds the last-snapshotted pack-data per device; runner triggers transform-memory updates explicitly via the runner→transform path (or transform_wrapper computes the new memory directly from cfg, depending on which is cleaner). |
| **A7** | **YANG scaffold is incomplete** — no per-device association/request surface yet. | Accept. This is the next concrete YANG work; deferred until design is settled to avoid two passes. |
| **A8** | **Plan-refresh monotonicity is load-bearing** — must enforce "may add steps, must not remove" exactly as the Python does. Claude (item 12) adds the trigger-discipline part: refresh runs **after every step**, not only at run start (this is what enables IOS-XR FPDs to surface mid-run). | Accept. Make both the monotonicity invariant and the after-every-step trigger explicit in design §6.2. |
| **A9** | **Capability vs pack OS separation** — the device's NETCONF-capability-derived `DeviceOs` (validates the actual platform via `ValidatePlatform`) is distinct from the pack `os` enum (selects which step module's `get_steps()` to call). Don't conflate. Keep `vrp` in the YANG enum, fail validation cleanly. | Accept. |
| **A10** | **Run-log filter (`swi_component` attribute)** is part of the spec — only records emitted with the package's logger context get persisted. Don't naively persist every record that flows past. | Accept. (Codex hits this implicitly via "preserve the run-log shape"; Claude item 15 makes it explicit.) |

---

## Codex-unique catches

These are catches surfaced only in `01-codex-design.md`. All accepted unless noted.

| # | Issue | Action |
|---|-------|--------|
| **C1** | `enabled` semantics need pinning: allow request materialization, suppress execution, in-flight runner reaches cooperative stop point and pauses, re-enable resumes per `start-generation`/auto policy. | Accept. Document in design §4. |
| **C2** | `FileTransfer` should expose typed `RemoteFileInfo` (path, size, checksum, algo, mtime) instead of `dict[str, int]`. | Accept. |
| **C3** | `FileTransfer` should expose explicit `FileTransferCaps` (put/delete/checksum/device_pull bool flags) so the runner can distinguish "cannot verify" from "verification failed". | Accept. |
| **C4** | Preserve `scp-port` (currently in the original per-device augment) somewhere — sw-install device config or stratoweave device metadata. Don't drop it. | Accept. Park it on the per-device sw-install subtree we add in A7. |

---

## Claude-unique catches

These are catches surfaced only in `02-claude-design.md`. The credential-reuse one (CL3) is the most consequential; it changes the design.

| # | Issue | Action |
|---|-------|--------|
| **CL1** | **Action return values regression.** `create-request` returned `(request-id, status ∈ {new-request, existing-request})`. With reactive triggers, the caller has to *poll the oper subtree* to discover the assigned id and whether their config-write created a new request or coalesced. Real ergonomic regression for any external automation. | Accept. Add a per-device `last-create-result` oper sibling: `{request-id: uint32, status: enum(new-request|existing-request), at: yang:date-and-time}` — runner stamps it on every transform-memory diff. Cheap and removes the polling problem. |
| **CL2** | **Stage-and-review beat is lost** if `unprocessed → processing` is implicit on creation. Reactive config-driven means an operator who wants to inspect the plan before kicking off has to chase the first confirmation gate or set per-step confirms. No equivalent of "pause the whole request without setting per-step confirmations." | Accept. Reinstate `start-generation` as the explicit start gate (it's already in A2). Revise design §4.2: "transition to `processing` requires `start-generation` to advance OR `auto-execute-after-confirm` to be true after all confirms are received." Default policy choice between the two TBD; my lean is preserve "explicit start" to match spec semantics. |
| **CL3** | **Credential reuse for `FileTransfer` doesn't actually work as currently designed.** Walking through the code: `DeviceMetaConfig.credentials` lives in `device_meta_config.act`; DMC is injected into the *adapter* via `DeviceAdapter.set_dmc(...)`; neither `DeviceMgr` nor `DeviceAdapter` exposes credentials publicly. A `FileTransfer(swdev.DeviceMgr)` constructor literally cannot obtain creds without an API change. Three real options: (a) extend `DeviceAdapter` with `get_credentials()`; (b) pass DMC at the factory boundary (and figure out DMC visibility for our non-RFS transform); (c) **piggyback file transfer on the netconf adapter's existing SSH transport** — the cleanest because the transport is already authenticated. | Accept the diagnosis. Decision: prefer (c) where supported, fall back to (b) with explicit DMC plumbing through the transform. Add a §9.4 to the design doc working through this with concrete API sketches. **This is a real design change; codex flagged the symptom (factory should take device meta) but Claude walked the actual code paths to show the gap.** |
| **CL4** | **`confirm-step all` semantics gap.** The original action takes `case all { leaf all empty; }` to confirm every step in one shot. Reactive model requires the caller to enumerate every (component, step) and set `confirmed`. Caller now has to read oper to know what steps exist before it can confirm them all. | Accept. Add a per-request `confirm-all-generation` writable leaf — runner expands into per-step confirmations. Three lines, removes a real ergonomic regression. |
| **CL5** | **Backoff persistence across restarts.** Spec §10 explicitly calls out that backoff state persists in CDB across worker restarts — load-bearing. Acton's `after delay:` is in-memory only; controller restart loses the wakeup. | Accept and elevate this to a design requirement. Persist `error_count.backoff` and a `next_wake_at: yang:date-and-time` in dynstate; on restore, schedule a fresh `after` with `max(0, next_wake_at - now)`. |
| **CL6** | **Cancel latency** — Acton actors aren't kill-able mid-callback. Cancel takes effect at the *next step boundary or completion of the current outstanding RPC*, whichever is sooner. For IOS-XR's `_monitor_operation_log` poll loop with 600s timeout, cancel can take up to 10 minutes. | Accept. Document the semantics explicitly in design §8: silent "cancel may take 10 minutes" is a bug-shaped surprise. |
| **CL7** | **Runner-creation idempotency races.** Two pack-data updates in quick succession could spawn two runners before either marks prior requests obsolete. Make creation idempotent on `(device, request-id)`. | Accept — combines naturally with D1 (one DeviceRunner per device, request generations inside it). |
| **CL8** | **`_execute_step_action` flush ordering** — `state.flush` only if `result != FAILURE`, then refresh, then persist plan. A FAILURE result must NOT persist half-mutated state. | Accept. Document as an invariant in design §6.2. |
| **CL9** | **`State.reset()` semantics** — called from `CheckVersions` when previously-Done version is no longer running on the device. Without it, a drifted device gets stuck. | Accept. Add to design §5 (state types) — each per-OS State must implement `reset()`. |
| **CL10** | **Junos `dual_re` cross-run invariant** — if `state.dual_re` changes between runs of the same request, `ValidatePlatform` returns FAILURE; the request is invalidated. | Accept. Document as a Junos-specific invariant in design §6.3. |
| **CL11** | **IOS-XR archive parsing is controller-side filesystem work** — `state.packages` extraction reads `.iso` (`get_version_from_iso`) and `.tar` (`get_file_packages`). Non-trivial new dependency (iso/tar libs in Acton). | Acknowledge in design §9 alongside FileTransfer. Note: this is for Phase 5 (IOS-XR), but the cost should be visible now so we don't get surprised later. |
| **CL12** | **Junos has both `Done[re_id]` and a trailing unparameterized `Done`.** Don't lose the trailing one when implementing `StepKey`-as-dict-key. | Accept — design already supports it via `re_id: ?str`; just call out the trailing-Done case in §6.3 explicitly. |
| **CL13** | **`request-id` starts at 1.** Trivial nit. | Accept. Note next to the `request_id: u32` field. |
| **CL14** | **`internal-state` is referenced in the design but not in the YANG.** I dropped it from the YANG but still describe it in §3.2. | Accept. Either restore as an oper string leaf or drop the §3.2 reference. Lean: drop the reference; dynstate is the source of truth. |
| **CL15** | **`RequestRunner` restart story is unspecified.** `update_dynstate` is transform-level; `RequestRunner` is a child actor, not a transform. State has to bubble up to the transform's dynstate to be persisted, and runners re-spawn from restored snapshot on boot. | Accept. Material engineering, not implementation detail — call it out in design §8 with a concrete plumbing sketch. |
| **CL16** | **§9.2 SUCCESS vs SKIP_STEP slip.** I wrote "CopyImage.pre_check short-circuits to SUCCESS only if files are already present" — that's wrong. Files-already-present means SKIP_STEP (don't run execute); SUCCESS would call execute() and FAILURE redundantly. | Accept. Fix in design §9. |
| **CL17** | **Filename interpretation needs YANG clarification.** Path semantics are FileTransfer-impl-defined; YANG should say so explicitly or split into two leaves. | Accept. Clarify in YANG comment. |
| **CL18** | **`Cleanup` step is IOS-XR only**, not SROS. Don't accidentally add a SROS Cleanup. | Accept. Fact-check in §6 of the design. |

---

## Conflicts and divergences

There are no actual conflicts between the two reviews — they're additive. The closest to a divergence is on the start-gate mechanism:

- Codex: control subtree with `start-generation` (uint64 increment).
- Claude: writeable boolean `start-requested` that the runner clears.

I'm taking codex's pattern (generation counters) per A2, since it's idempotent under replay and avoids the runner clearing config behind the user's back. Claude's underlying point (that we need the gate at all) is in CL2 — I'm not throwing that away, just picking the mechanism.

---

## Two architectural questions worth a deliberate decision

These both came up in Claude's review and deserve explicit answers before I revise the design.

### Q-A: One DeviceRunner per device, or one RequestRunner per active request?

Codex (Q2 §): "Prefer one `DeviceRunner` actor per device, with request generations inside it, rather than independent request actors racing on the same device."

Claude (item 7): same direction — runner-creation idempotency, `(device, request-id, generation)` token model.

Both push toward **one DeviceRunner per device**, owning per-request child state internally. This is also what the Python code semantically did via `MaapiLocker` (only one install owns the device at a time). I'm convinced — flipping the design from per-request actor to per-device runner with internal request state.

**Decision D1: One `DeviceRunner` actor per device.** Internally it holds `current_request: ?RequestState` (only one active at a time). New pack-data → if a current request is running, mark it obsolete and start a new one; old request's pending callbacks become no-ops via generation-token check. Cancel signal aborts the current one. This also gives us the install-lease semantics for free (the actor owning the device IS the lease).

### Q-B: Is `Transform` actually the right substrate, or should sw-install be an actor wired at app-spec time like `DeviceRegistry`?

Claude (smaller items §): "A transform whose `transform_wrapper` returns `(Container, memory)` (empty downward) and that doesn't push device config is unusual in the codebase — sw-install is a parallel control loop, not part of the layer-stack data path. … Worth asking: does it actually want to be a `Transform`, or something else?"

This is a genuinely good question. The argument for `Transform`:
- Free `update_oper` / `update_dynstate` plumbing.
- Reads upper config naturally.
- gen_adata wiring for the YANG model.

The argument against:
- We don't produce downward config — abusing a transform that returns empty `Container`.
- We're observer-shaped, not transformer-shaped.
- gen_adata may not have a precedent for "transform that's a sink/observer."

**Decision D2: Defer until I've prototyped the Transform path.** Try `Transform` first because the auxiliary services (oper/dynstate/yang wiring) are real value; if `gen_adata` rejects the empty-output shape or the integration is fundamentally awkward, fall back to "sw-install is a top-level actor wired at app-spec time, reading config via `Layer.get_data()` and a subscription, publishing oper via its own `TreeProvider`." Document the fallback path in the design doc so we're not stuck if the Transform substrate doesn't fit. **This requires a brief platform-owner sanity check before Phase 4 implementation continues.** I'll surface it explicitly in the round-2 brief.

---

## Items I'm pushing back on or scoping differently

Very few. Two soft pushbacks:

1. **Sibling repo vs in-tree (Claude smaller items §):** Claude suggests starting in-tree. I'm sticking with sibling repo because the iteration loop is short *enough* (single make build), and the cleanliness benefit (clear API surface, separate Build.act, no contamination of the platform repo) outweighs the marginal iteration speed. Open to revisiting if churn proves higher than expected.

2. **`software-install-matrix` (Codex deferred-features list / Claude smaller items §):** Both ask to capture in a "deferred features" list rather than silently drop. Accepted. Add a §15 to the design doc enumerating what we deliberately drop or defer.

---

## Action items flowing into the design doc revision

In rough order of size:

1. **§4 — control subtree + reactive triggers.** Replace the prose with a concrete YANG sketch: `request-generation`, `start-generation`, `cancel-generation`, `clear-run-log-generation`, `confirm-all-generation` as uint64 leaves; `confirmation` list keyed by `(request-id, component, step)`. Explicit `enabled` semantics. Last-create-result oper sibling for the action-return-values regression.
2. **§8 — DeviceRunner architecture.** Flip from per-request actor to per-device runner with internal request state. Document: install-lease semantics, generation-token callback discipline, cancel latency, restart story (dynstate plumbing, fresh-runner-on-boot from restored snapshot, persistent backoff).
3. **§9 — credential reuse.** Add §9.4 working through the three options for FileTransfer credential acquisition; commit to (c)-piggyback-on-netconf-SSH as the preferred path with (b)-DMC-plumbing as fallback. Typed `RemoteFileInfo` and `FileTransferCaps`. Preserve `scp-port`. Fix SUCCESS/SKIP_STEP confusion.
4. **§7 — transform substrate decision.** Try `Transform`; document the fallback to top-level actor + `TreeProvider` if gen_adata or oper-wiring rejects the observer shape. Surface this question to platform owners in round 2.
5. **§3 — config/oper split.** Revise the table to show clean axes: pack library + control subtree + confirmations are config; request[]+plan+run-log are oper.
6. **§6 — plan/step semantics fidelity.** Add: refresh-after-every-step trigger; flush-only-if-not-FAILURE invariant; `State.reset()` semantics; Junos `dual_re` cross-run invariant; trailing-Done step on Junos.
7. **§5 — state types.** Per-OS `reset()` defined. `request-id` starts at 1. Drop `internal-state` reference (or restore as oper-only summary leaf — leaning drop).
8. **§12 — close out resolved decision points.** Most are now resolved by the items above. Re-list.
9. **New §15 — deferred features.** `software-install-matrix`, IOS-XR archive parsing dependency, anything else explicitly dropped or deferred to Phase 5+.

---

## Round-2 brief plan

After the design doc is revised:

1. `/clear` both pty-2 (codex) and pty-3 (claude).
2. Brief each with: project context, pointer to the three onboarding docs (00, 01, 02), pointer to round-1 reviews if they want to see prior history, and the same four focused questions.
3. **Crucially: do not summarize round-1 feedback in the brief.** Per the iteration-loop convention in memory — fresh context, no complacency carry-over. Their job is to find blind spots from zero.
4. Add one platform-substrate question for both (the Q-B / transform-vs-actor question above) — it's structurally consequential and worth a second-round look.

The expectation: round 2 may surface fewer items if the docs are now solid; if it surfaces a comparable volume, the docs aren't yet self-contained and need another round.
