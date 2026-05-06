# Round-4 Claude review (`02-sw-install-design.md` v4 + supporting docs)

Reviewer: Claude (pty-1), reviewing as a fresh reader against the full v4 doc set.
Date: 2026-05-07.
Scope: `00-orientation.md`, `01-software-install-logic.md`, `02-sw-install-design.md` v4, `adr/cli-driver.md`, `stratoweave/sw-install/src/sw_install/yang.act` v4, plus sanity-check against live `stratoweave/stratoweave/src/stratoweave/{ttt.act,device.act,adapters/adapter.act}` and `sorespo/src/sorespo/{rfs.act,sysspec.act}`.

I assembled my own view of the platform first; only afterwards did I scan filenames in `docs/reviews/` for divergence. I have **not** read the round-4 codex review (`09-codex-design-r4.md`) so as not to contaminate this assessment.

---

## TL;DR

v4 has integrated round-3 review feedback cleanly. The four convergent r3 HIGH items (RFSTransform → plain Transform; dynstate restore path; `get_dmc()` already exists; `scp-port` placement) are resolved correctly. The new v4 additions (§3.5 restore consistency, §3.7 three-tier writes, §4.4 cancellation two-lane, §6.6 step-logger plumbing, last-trigger-result oper feedback) are mostly sound but have a handful of concrete issues that would bite during Phase 4 implementation. **Five HIGH-priority items below need a v5 pass before code starts.**

The single most important to land is **H1**: §8.2 declares the runner's `update_oper`/`update_dynstate` fields as `action(...)` but the platform actually exports them as `proc(...)` (`ttt.act:1898`). v4 will not typecheck as written.

The platform-prerequisite ask in §14 (`TransformActorParams.dynstate`) is a small, additive, well-motivated change — endorsed.

---

## What v4 got right (carry forward unchanged)

- Switching from `RFSTransform` to plain `ttt.Transform` (§7) is correct. Verified: `_RFSTransaction.finalize` (`ttt.act:2151-2158`) suppresses `on_conf` when output is empty Container; `_TransformTransaction.finalize` (`ttt.act:1989-1993`) does not. `RFSFunction.init_dynstate` (`ttt.act:2184-2185`) does not pass `lower`; `TransformFunction.init_dynstate` (`ttt.act:2011-2012`) does. Plain Transform is the only substrate that gives sw-install both the `on_conf` callback for empty downward output and `params.lower` for global subscriptions.
- Collapsing transform `memory` and treating dynstate as the sole runner state (§3.1) — confirmed sound; both r3 reviewers endorsed.
- Three-tier write classification (§3.7) is the right framing in principle. (See H2 for an implementability concern with Tier A.)
- `last-trigger-result` (§4.6) replaces a worse "silent fail on missing target-id" — good fail-loud move.
- CLI ADR is well-scoped; the §9.7 cleanup (no per-method NotImplementedError stubs) is the right Phase 4 obligation.
- Round-3 H3 framing fix (`get_dmc()` exists at `device.act:401-402`) is correct and the v4 wording is honest about the prior doc drift.
- YANG model (`yang.act`) reads cleanly; the `when` constraints on OS-specific diagnostic leaves are correct (path `'../../software-pack-data/os'` from `request[id]/component[name]/op-id-add` resolves to `request[id]/software-pack-data/os`).

---

## HIGH

### H1 — `update_oper` / `update_dynstate` typed as `action`, but platform exports them as `proc`

`02-sw-install-design.md` §8.2 declares:

```
update_oper: action(?gdata.Node) -> None,
update_dynstate: action(?gdata.Node) -> None,
```

But the platform constructs `TransformActorParams` (`ttt.act:1898`) with:

```
update_dynstate: proc(?gdata.Node) -> None
update_oper:     proc(?gdata.Node) -> None
```

These are not interchangeable in Acton's type system. The `act` callback in §7.2 passes `params.update_oper` and `params.update_dynstate` directly into the `DeviceRunner` constructor — that won't compile against an `action(...)` field.

**Fix:** change the DeviceRunner field declarations in §8.2 (and the same types where they appear in §7.2 sketch) to `proc(?gdata.Node) -> None`. Verify by trial-compiling a stub Phase 4 skeleton before deeper work begins.

This is a one-letter design bug, not an architectural problem — but it must be corrected because the §8.2 sketch is the single most-quoted block in the design and copies into Phase 4 code verbatim.

### H2 — Tier A "synchronous, await ack" is not straightforwardly implementable

§3.7 says Tier A writes are "synchronous (await ack)" — the runner blocks until the LMDB write completes before performing the side effect.

But the platform's `update_dynstate` is `proc(?gdata.Node) -> None`. It returns void. Looking at the call chain: `update_dynstate` lands in `_TransformTransaction.update_dynstate` (`ttt.act:1981-1987`), which sets `self.dynstate` and triggers `Session(...).recompute()`. `db_ops` (`ttt.act:1961-1962`) emits `BufferOp`s into the per-Session buffer; the actual LMDB transaction commit happens at Session.commit time, which is asynchronous to `update_dynstate`'s return.

There is **no per-call ack** that the runner can await. As written, "Tier A: block until LMDB write completes" is aspirational, not implementable against the existing API.

**Why it matters:** the "trigger consumed but side-effect un-issued" failure mode is recoverable (the design notes this — reconciliation cleans up); but the *opposite* failure — **side effect issued, persist still pending in buffer, crash before commit** — would re-fire the trigger on restart. The op-id case (§3.7 Tier A, IOS-XR Phase 6) is the dangerous one: `op_id_add` issued to the device, persist still in buffer, crash → restart → no op-id known → re-issue `install add` → device sees a duplicate / interleaved request. (This is the IOS-XR async-RPC trap the Python `_monitor_operation_log` was carefully written around.)

**Possible fixes (pick one, document in v5):**
- (a) Platform addition (v2.0 ask): `update_dynstate(node, cb: action() -> None)` variant that fires `cb` after the BufferOp lands in a committed transaction.
- (b) Accept that Tier A persist happens-before is not guaranteed at the platform level, and design the runner so every Tier-A-protected side effect is **idempotent against re-fire**. For request-generation/start-generation this is already true (re-running create-request with the same pack returns same id; re-running start with `unprocessed` is no-op). For IOS-XR `op_id_*`, this requires the per-step pre_check to recognize "an in-progress install op exists for this device" via `o.get_current_install_request()` before issuing a new one — which the Python original *does* do (§3.4 `SoftwareCommit.pre_check` notes "or `o.get_current_install_request()` non-None → WAIT"). Carry that pattern forward.
- (c) Document this concretely in §3.7 as a Phase-6 hazard rather than letting "Tier A is synchronous" sit unchallenged.

I'd lean (b)+(c) for v1; (a) becomes a v2.0 platform ask.

### H3 — §3.5 restore-consistency check description doesn't match what the check actually catches

§3.5 prose says the check defends against "config newer than dynstate after a partial backup restore." But the actual check is:

```
max(history.request_id, current.request_id) ≥ dynstate.next_request_id
```

That's a **dynstate-internal** invariant — `next_request_id` should always exceed all known request ids. It catches dynstate corruption or partial dynstate-blob restore, but it does **not** catch a config-side restore that races dynstate. There's no separate "published request id" record outside dynstate to compare against:

- Verified: the platform persists only `ATTR_MEMORY` and `ATTR_DYNSTATE` in the transform's lmdb namespace (`ttt.act:1962`); oper data is not persistent.
- Therefore "max published request id" reduces to "max id in dynstate.history" — same blob as `next_request_id`. They co-vary on any single-blob restore.

The genuinely dangerous direction described (`cfg.request-generation > dynstate.last_request_generation`, dynstate restored older) **is not detected by this check at all**. It just fires a fresh request with `dynstate.next_request_id` from the older snapshot — likely small but consistent within dynstate.

**Fix:** Either rewrite the prose to match what the check actually does (defense against dynstate-blob tampering / restore-mismatch within the dynstate blob itself), or design a check that uses an independent record. A reasonable independent record is the *config tree* itself — if the config carries a `last-request-id-hint: uint32` that the runner periodically writes back via a regular config edit (or recommended-by-runtime pattern), config-only restore would carry it; comparing config's hint to dynstate's `next_request_id` would catch the dangerous direction.

**Lower-effort alternative:** demote §3.5 to "an internal-consistency invariant guard; full backup-restore safety is a v2.0 follow-up that requires a separate fence" and remove the misleading "config newer than dynstate" framing.

### H4 — §7.2 subscription wiring constraint is too quiet for how easy it is to misconfigure

§7.2 says: "Apps integrating sw-install must ensure `/software-install/...` is in the layer below the sw-install layer". This is a non-local cross-layer wiring requirement that:

- isn't checked at compile time (Acton's type system can't see layer composition);
- isn't checked at runtime in the design as written (the runner declares the subscription; if the lower layer doesn't carry that subtree, the subscription's callback just never fires with relevant data, and the runner sits idle reading defaults);
- is exactly the kind of constraint that's right at app integration time and silently broken six months later when someone restructures the layer stack.

**Fix:** the runner should declare the subscription and **also** publish a clear `last-trigger-result` (or a dedicated oper leaf, e.g., `runner-status: enum {ok, missing-global-config, restore-inconsistent, ...}`) when the subscription returns nothing within a startup window (e.g., 5s after first `on_local_config`). Operators get a fail-loud signal instead of silence. Without this, the v1 wiring constraint is a footgun.

The "v2.0: subscribe-to-current-layer API" platform ask in §14 is fine, but for v1 we still need the runtime fail-loud guard.

### H5 — Plain `Transform` placed *inside* `/sw-rfs:rfs` is mechanically OK but breaks platform convention; needs an explicit note

Verified by code inspection: there is **no precedent** in the platform for a plain `Transform` (not `RFSTransform`) attached as a per-list-entry transform inside `/sw-rfs:rfs`. Sorespo's per-device pattern (`sorespo/src/sorespo/rfs.act`) goes through `RFSTransform`. The platform's `RFSTransform` is the documented per-device-transform API: it populates `params.dev`, has a `RFSFunction` shape with `transform_wrapper(cfg, di, memory, dynstate)`, etc.

The design sidesteps both `_RFSTransaction.finalize`'s empty-output suppression and `RFSFunction.init_dynstate`'s missing `lower` by using plain Transform — workable, but a future reader looking at the host layer composition will see `swi_factory` wired alongside `BBInterfaceTransform` (or whatever sorespo-style RFS transforms the app has) and reasonably expect it to also get `params.dev`. It won't.

**Fix:** §7 should add an explicit one-paragraph note: "We are using plain `Transform` *inside the RFS list*. This is a deliberate departure from the platform convention that per-device transforms go through `RFSTransform`. We do this because (a) `_RFSTransaction.finalize` suppresses `on_conf` for empty output and (b) `RFSFunction.init_dynstate` does not thread `params.lower`. The trade-off: the runner does not receive `params.dev` and must call `dev_registry.get(devname_from_path(params.path))` itself. Apps wiring this transform should not assume RFS-style semantics." Explicit > implicit here.

A v2.0 platform ask should be added to §14: either (1) parameterize `_RFSTransaction.finalize`'s empty-output suppression so RFS transforms can opt out, or (2) thread `lower` through `RFSFunction.init_dynstate`. With either, sw-install could move to `RFSTransform` and the convention realigns.

---

## MED

### M1 — `devname_from_path(params.path)` is referenced but not implemented

§7.2 / §3.6 sketches use `devname_from_path(params.path)`. The platform's existing `devname_from_device_path` (`ttt.act:2206-2209`) requires `path: PathKey` directly — it's used by `_DeviceTransaction` whose path is just `device[name=R1]`. sw-install's path is `/sw-rfs:rfs[name=R1]/software-pack` — multiple components, ending in `PathContainer("software-pack")`. The existing helper won't work.

**Fix:** specify in §7 the actual extraction: walk `params.path` looking for the `PathKey` whose parent is `/sw-rfs:rfs`, take its `name`. Or: take `params.path.parent` (one level up from the `software-pack` container) and apply `devname_from_device_path` to that. Either way, write the helper into the design as a few lines so Phase 4 doesn't reinvent it.

### M2 — `enabled` mid-step gating not shown in §8

§4.7 says "current step's RPC may complete; no further steps execute" when `enabled: true → false` mid-processing. The §8 DeviceRunner sketch doesn't show the inter-step `if not global_config_cache.enabled: transition to paused; return` check that this requires. §8.6 shows the check at `_start_run()` start, but step-to-step transitions in the run loop are not depicted.

**Fix:** §8.3 (re-entrancy invariants) should add a fourth invariant: "Between steps, the runner re-checks `enabled` and `dynstate.current.status` (cancelling, terminal); if either disqualifies continuation, transitions cooperatively." And the §8 run-loop sketch should show it.

### M3 — `confirm-all` `by-user` value is unspecified

§4.3 says `confirm-all-generation` doesn't create persistent `control/confirmation[]` entries; instead the runner stamps `confirmed.{by-user, when}` in oper. But the YANG `confirmed` container has `by-user: string` (no mandatory in oper — it's `config false`); the runner needs to put *something* there. Should it be:

- `"<confirm-all>"` (sentinel)
- `""` (empty)
- The most recent operator ID known to the runner (we may not have that)
- Omit `by-user` entirely from runner-stamped projections (`confirmed` is a presence container; leaving `by-user` absent would be valid YANG since `mandatory true` on the *config* `control/confirmation/by-user` doesn't apply to the oper projection).

**Fix:** pick one and document. I'd lean `"<confirm-all>"` (operationally legible) over absent (RESTCONF clients may serialize awkwardly).

### M4 — Single-slot `last-trigger-result` is racy for fast successive triggers

§4.6 / §4.4 / §4.7 all converge on a single-slot oper container. Operators automating sw-install will poll this slot to confirm their trigger landed. Cancel-then-clear-run-log in fast succession overwrites the cancel result before the operator has read it. This is acknowledged ("single-slot last writer wins") but the operational consequence isn't called out.

**Fix:** either (a) widen to a small ring (last 5 results) — small YANG change, big operability win — or (b) explicitly note in §4.6 and the YANG description that this slot is **not** suitable for automation polling, and recommend trigger-result correlation via run-log entries (which the runner emits at trigger consumption).

### M5 — `_drain_notify` token comparison logic in §8.2 is suspect

§8.2 sketch:

```
if dynstate.current is not None and dynstate.current.status == CANCELLING and stale_token + 1 == dynstate.current.generation_token:
    dynstate.current.status = CANCELLED
```

§8.5 cancel implementation bumps `dynstate.current.generation_token += 1` at cancel-time. So at the moment a stale callback for the OLD token (= `current.generation_token - 1`) arrives, the comparison `stale_token + 1 == current.generation_token` holds → drain notify transitions to CANCELLED. So far so good.

But: what if the runner has already started a *new* run after cancel-then-restart-then-execute? Then `current.generation_token` has been bumped again (at run start). The stale callback's `stale_token + 1` is two behind, the comparison fails, the runner just drops the callback. **What does that leave behind?** A `cancelling`-state request that no longer matches anything in flight. The watchdog (`CANCEL_DRAIN_TIMEOUT`) covers this, but only after 600s — that's a long stuck state for what is a recoverable case.

**Fix:** the drain check should be `stale_token < current.generation_token` rather than `stale_token + 1 == current.generation_token`. (The drain *is* still a drain even if multiple generations have passed.) Keep the watchdog as a backstop.

### M6 — Oper data is not platform-persisted; document the startup window

Verified: `db_ops` (`ttt.act:1961-1962`) persists `ATTR_MEMORY` and `ATTR_DYNSTATE` only. There is no oper persistence. After a platform restart, oper is empty until the runner runs first reconciliation and calls `update_oper`. External readers (RESTCONF clients) polling `/sw-rfs:rfs[name=R1]/software-pack/request` will see no requests during this window, then they'll appear as the runner publishes. For monitoring/automation this matters.

**Fix:** §8.4 (restart story) should explicitly say "step 6 first `on_local_config` is also when oper is first populated; clients polling during the gap see empty data and should retry." And §15.5 should add it as a conscious deviation from the Python original (CDB oper persisted across NSO restarts).

### M7 — `dev_registry.get(devname)` call shape

§7.2 + §8.2 use `dev_registry.get(devname)` synchronously. `swdev.DeviceRegistry` is an actor (per `00-orientation.md`); `get` is presumably mailbox-bound. Need to verify whether this returns a value synchronously (Acton actor calls can be `def` returning value vs `proc def` returning future) before Phase 4 codes the `act()` callback as if it were synchronous. If it's async, the runner needs a "waiting-for-device-mgr" pre-state.

**Fix:** one-line check during Phase 4 skeleton; add to §8 a note: "if `dev_registry.get` is async, `act()` must defer construction to a callback chain."

### M8 — Stale callback drops *all* mutations including `op_id_*`

Per §4.4 two-lane callback rule: the cancelled run's callback is "no-op'd" — its `(StepResult, NewState, ?Exception)` is dropped. For SROS Phase 4, fine. For IOS-XR Phase 6, the in-flight RPC may have **successfully started** an `install add` and returned an `op_id_add` that the next run needs to know about (so its pre_check sees the op-log and SKIP_STEP's). Dropping `NewState` discards the op-id.

If §3.7 Tier A persisted op-id-add **before** issuing the RPC, the dynstate has it. But §3.7 Tier A says the op-id is persisted "before issuing the device RPC that returns the op-id" — there's a chicken-and-egg: you can't persist before issuing if you don't know the value yet. The Python pattern is the alternative: issue, get id, then write — and rely on `o.get_current_install_request()` to observe in-flight ops on the device after restart.

**Fix:** §3.7 Tier A item 4 (IOS-XR `op_id_*`) needs reformulation — they are NOT persistable before the side effect. Their recovery story is "device-side observation via `get_current_install_request()` on next reconciliation". Move op_id_* out of Tier A into Tier B with an explicit note. (Phase 6 concern; flag now so Phase 6 doesn't take the Tier A claim at face value.)

---

## LOW

### L1 — `00-orientation.md` actor description is misleading

`00-orientation.md` "Key actors / concurrency" says: "`actor Layer(...)` — owns a `ttt.Layer` instance". `Layer` *is* the actor (`ttt.act:604` — `actor Layer(name: str, rootgen: ..., lower: ...)`). It doesn't own a separate `ttt.Layer` instance.

**Fix:** "`actor Layer(...)` (defined in `ttt.act`) — the per-layer actor that owns a tree node, accepts edit_config, and runs transforms."

### L2 — `00-orientation.md` could use a one-paragraph signpost on `gen_adata`

§"YANG-as-types" mentions `gen_adata` but doesn't say where the build step lives or how a reader runs it. A two-line "you run `gen_adata` via the per-app `gen_adata/` build target; outputs land in the app's `model.act` and `device_meta_config.act`" would help the fresh-reader case.

### L3 — `01-software-install-logic.md` §3.4 — `next_step` after SKIP_STEP

§3.4 says "Always run `next_step()` after execute (even if step skipped)." But the prose right above is clear that `next_step` runs after SUCCESS or SKIP_STEP from execute (or pre_check pathways), and the `next_step` is "a step class to JUMP TO". Worth tightening: when `next_step` itself raises, the *current* step degrades to FAILURE — the prose says this in §3.4 last paragraph, but the table in §3.5 doesn't reflect it. Worth a one-line clarification in §3.5 mapping table: "If `next_step()` raises after SUCCESS/SKIP_STEP of execute, the step transitions to FAILURE."

### L4 — `01-software-install-logic.md` §6.2 IOS-XR Cleanup deletes `state.destination_paths.values()` — preserve in port

The Python's `Cleanup.execute` SCP-deletes copied images. v4 Phase 4 ships `NoopFileTransfer` and §15.5 #15 notes `CheckFiles` becomes a no-op. **Cleanup** isn't called out as also being a no-op in Phase 4. For IOS-XR Phase 6 it'll matter; flag in §15.5 #15: "Cleanup is also a no-op when FileTransfer.caps().delete is false."

### L5 — `02-sw-install-design.md` §15.5 #5 wording

§15.5 #5 talks about generation counters surviving partial backup-restore. As noted in H3 above, the actual check doesn't catch the dangerous direction described. Either rewrite per H3 or note explicitly: "v1 design detects only dynstate-internal inconsistency; cross-cutting backup-restore safety is a v2.0 follow-up."

### L6 — `02-sw-install-design.md` §3.7 Tier B includes `current.run_id_count` "at run start"

`run_id_count` is bumped before any step runs (per `01-...` §3.3). If the bump is "at run start" Tier B, but the persist is at "step boundary," there's a window where run_id_count has bumped in-memory but the first step's persist captures both. Fine in practice — but the prose makes it sound like a separate persist event. Tighten: "Tier B writes flush at step-completion boundaries; `current.run_id_count` is included in that flush rather than as a separate Tier-A side-effect-coupled write."

### L7 — `adr/cli-driver.md` — Phase 5 secrets-in-runlog hazard

ADR Issue 6 (secrets in presets/logs) is a future Phase 5 concern. **But**: the Phase 4 `RunLogHandler` (design §6.6) installs on the runner's logger and accepts any record bearing `swi_component`. When Phase 5 lands and a `Send "${Pass}"` template runs in CliSession, the transcript is logged via the same logger chain and gets persisted to the run-log ring. Phase 4 should leave a hook: a per-record "do not persist to run-log" flag (e.g., a structured-data key `swi_redacted: true`) that the handler honors. Add a one-line obligation to §6.6: "`RunLogHandler` skips records with `swi_redacted=True` in their structured-data dict — Phase 5 transcript redaction will use this."

### L8 — Onboarding doc read-order

`00-orientation.md` reads well as a fresh-reader doc. One small ergonomic improvement: the "Useful questions to ask yourself when reviewing my work" section is good but appears mid-doc; consider moving it to the end (after "What's next") so it reads as the closing prompt rather than mid-flow.

### L9 — `01-software-install-logic.md` is the strongest doc

This doc is genuinely useful as a hand-off artifact for a Python-unfamiliar Acton implementer. The §10 idempotency invariants table is the single highest-leverage paragraph in the entire doc set. Carry forward unchanged.

---

## Onboarding doc evaluation (does the v4 doc set stand alone for a fresh reader?)

**Largely yes.** A reader who works through `00-orientation.md → 01-software-install-logic.md → 02-sw-install-design.md → adr/cli-driver.md → yang.act` in order will get sufficient context. Issues:

- `00-orientation.md` could pre-empt the "what is dynstate vs memory vs oper" question that the reader will hit in §3 of the design (the orientation §"Operational state and the observer-transform pattern" hints at it but doesn't lay out the full triple). A 4-row table (config / memory / dynstate / oper — what they're for, where they live, who writes them) would save readers a cross-doc trip.
- The `02` design references `docs/reviews/08-integration-r3.md` for change anchoring (§"Status" line). A fresh reader without the review history will not find that helpful — it's a working artifact, not an onboarding target. Either inline the relevant change-list or move the reference to a footnote.
- `adr/cli-driver.md` reads cleanly but is positioned as Phase 5, leaving readers wondering what Phase 4 actually does for CLI. A one-paragraph "Phase 4 obligations" recap at the *top* (instead of buried at the end) would help.
- The doc set does not currently contain a reading-order index. Consider adding a `docs/README.md` (or pinning the order at the top of `00-orientation.md`).

---

## Endorsements (to balance the criticism)

- **The ask in §14 (TransformActorParams.dynstate)** is right-sized. Five lines, additive, doesn't break existing callers, unblocks a real architectural need. Strongly endorse over D3b stash.
- **The §15.5 consolidated deviations list** is the right pattern. Future maintainers will read this list before reading the Python original; it directly preempts "wait, why doesn't this match Python?" debates.
- **Phasing (Phase 4 NETCONF-only / SROS-first; Phase 5 CLI + FileTransfer; Phase 6 IOS-XR + Junos)** is well-scoped and matches what the code actually requires.
- **Dropping `internal-state` JSON in favor of typed projections** is the right modernization.
- **Generation-counter trigger pattern** is a clean reactive replacement for NSO actions and should age well.

---

## Recommended v5 changes (priority-ordered)

1. **H1**: fix `action(...)` → `proc(...)` in §8.2 and §7.2 sketches. (Trivial.)
2. **H2**: rewrite §3.7 Tier A semantics — either specify a concrete "happens-before" mechanism against the existing platform API, or accept idempotent-side-effect-on-restart as the contract and update wording. Move IOS-XR `op_id_*` out of Tier A into Tier B. (Substantive.)
3. **H3**: rewrite §3.5 prose to match what the check actually catches; demote the "config newer than dynstate" framing to a v2.0 follow-up. (Substantive.)
4. **H4**: add a runtime fail-loud check for missing `/software-install/...` in `params.lower` — a startup-window oper-published `runner-status` enum. (Small addition.)
5. **H5**: add explicit §7 note about plain-Transform-inside-RFS-list as deliberate departure; add v2.0 platform ask to §14. (Documentation.)
6. **M1–M8**: address per the above.
7. **L1–L9**: doc cleanups, ergonomic.

After v5 lands, this is implementation-ready for Phase 4. The architectural shape is right; the gaps are concrete and bounded.

---

## Out-of-scope items I deliberately did not raise

- Per-OS Junos parametrized step typing — accepted v3 plan (`(name, re_id)` tuple).
- ComponentPlan refresh / monotonic invariant — well-handled in v3 / unchanged.
- File abstraction split (`LocalFileInspector` / `RemoteFileInspector` / `FileTransfer`) — the three-layer split is correct.
- `DeviceOps` facade — endorsed in r3, no concerns added.
- CLI ADR contents (TextFSMPlus selection) — the architectural choice is fine; Phase 5 design work will surface implementation issues that the ADR already itemizes.

---

## Cross-reviewer divergence

I deliberately did not read `09-codex-design-r4.md` before forming this assessment. Differences/overlaps with codex r4 will emerge at integration; I leave that to the integrator rather than pre-aligning here.
