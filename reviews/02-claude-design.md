# 02 — Independent Review of `02-sw-install-design.md`

Reviewer: Claude (pty-1), invoked alongside a parallel Codex review (pty-2).
Scope: design review of `02-sw-install-design.md` against `00-orientation.md`,
`01-software-install-logic.md`, and the live stratoweave codebase.

Materials read end-to-end:
- `docs/00-orientation.md`, `docs/01-software-install-logic.md`,
  `docs/02-sw-install-design.md`
- `stratoweave/stratoweave/src/stratoweave/ttt.act` — transform / RFSTransform /
  `TransformActorParams` / `update_oper` machinery
- `stratoweave/stratoweave/src/stratoweave/device.act` — `DeviceMgr`,
  `_send_config`, the `confq`/approval pipeline, `rpc_xml` forwarding
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act` —
  `DeviceAdapter` ABI (incl. `set_dmc`)
- `stratoweave/stratoweave/src/stratoweave/device_meta_config.act` — where
  `credentials/{username,password,private-key}` actually live
- `stratoweave/netconf/src/ssh_client.act` — how the netconf transport consumes
  those credentials
- `stratoweave/sorespo/src/sorespo/{sysspec,cfs,rfs}.act` — a real working
  example of the actor/transform pattern (incl. `BBInterfaceTransform` with
  `update_oper` / `update_dynstate`)
- Phase 4 scaffolding: `stratoweave/sw-install/src/sw_install/yang.act` and
  the rest of the `sw-install/` tree

The review answers Andrew's four focused questions, then lists smaller items.
Tone is intentionally direct: the brief was "push back on anything that smells
off, don't be diplomatic, the goal is to find blind spots."

---

## Q1. Dropping the YANG action nodes (§4)

**Mostly the right call, but two real losses you're under-stating, plus one
factual issue.**

What the reactive rewrite gets right: `create-request` idempotency falls out
for free from the "memory == pack-data" diff; `confirm-step` as a writeable
presence container is the correct stratoweave idiom; `cancel-requested` as a
writeable leaf is fine.

What you're throwing away that you don't acknowledge:

1. **Action return values.** `create-request` returns
   `(request-id, status ∈ {new-request, existing-request})`. With reactive
   triggers, the *caller* (a NETCONF/RESTCONF client, a higher-layer
   orchestrator, an operator script) writes config and then has to **poll the
   oper subtree** to find out what id was assigned and whether their write
   created a new request or coalesced into an existing one. That's a real
   ergonomic regression for any external automation. Not a blocker — but you
   should have a story for "how does the caller learn the request-id without
   polling?". Possibilities: write the assigned id back into a config-feedback
   leaf, or expose a "last-result" oper sibling per device. This needs to be
   in the design, not deferred.

2. **`execute-request` ≠ "transition to unprocessed".** Your §4.2 collapses
   *materialise* and *execute* into the same edge. The Python flow lets an
   operator stage a request — pack assignment changes → request materialised
   in `unprocessed` → operator inspects the plan and the per-step confirmation
   gates → operator triggers `execute-request`. You're losing the "stage and
   review before kicking off" beat. Your reply ("`confirm-steps` is `default
   true`, so the first step blocks anyway") only partly covers it — the
   operator still has to chase the first confirmation gate to inspect the
   plan, and there's no equivalent of "pause the whole request without setting
   per-step confirmations." Counter-proposal worth considering: keep
   `unprocessed → processing` gated on a writeable `start-requested` leaf (you
   mention this as a "trivial to add later" but I'd add it now — once you ship
   without it you'll regret retrofitting it).

3. **Factual issue with §4.3.** The original `confirm-step` action takes a
   `case all` (`leaf all empty`) — confirm every step in every component in
   one shot. Your reactive model requires the caller to enumerate every
   (component, step) and write `confirmed` on each. That's not a 1:1
   replacement; the caller now has to read oper to know what steps exist
   before it can confirm them all. Either accept this and document, or expose
   a `confirm-all` writeable leaf per request that the runner expands into
   per-step confirmations. The latter is cheap and worth it.

4. **`clear-run-log` → bounded log.** Mostly fine, but the Python intent is
   *forensic clearing between runs*: an operator triages a failure, captures
   the log, then clears so the next retry's log isn't polluted. A bounded
   ring buffer doesn't do that. Add a `clear-run-log-requested` leaf — it's
   three lines of code. The marginal-value argument in the design ("bound
   makes it self-managing") misses the use case.

---

## Q2. Per-request `RequestRunner`, relying on DeviceMgr for serialization (§8)

**This is the section I'm most worried about. The serialization claim is wrong
as stated.**

You write: *"Per-device locking: stratoweave's `DeviceMgr` is one actor per
device — operations against it are serialized for free."*

That is true at the **actor mailbox** level (one message at a time enters the
actor body) but it is **not true for in-flight NETCONF transactions**.
Concretely:

- `DeviceMgr.rpc_xml` (`device.act:783-784`) forwards directly to
  `adapter.rpc_xml(cb, xml_rpc)`. The adapter starts an async NETCONF RPC; the
  actor immediately accepts the next message.
- `DeviceMgr.configure` enqueues into `confq` and `_send_config()` calls
  `adapter.configure(...)` whose completion comes via the `on_configured`
  callback.
- In between **the call** and **the callback**, the actor is happy to dispatch
  other messages — including another `rpc_xml`, or another `configure`, or
  anything else.

So while a `RequestRunner` has an `install activate` RPC outstanding
(potentially minutes), the *normal* RFS pipeline can fire a `configure(...)`
with an unrelated diff (interface change, BGP tweak, anything from upper
layers) and the adapter will start another NETCONF lock+edit+commit sequence.
The NETCONF server will probably reject due to its own datastore lock, but the
order of completion is undefined and you can absolutely interleave RPC
sequences in ways the Python code's
`MaapiLocker.lock_partial(/ncs:devices/ncs:device[name=X])` is *explicitly*
designed to prevent. The Python lock holds for the **whole run**, not just one
RPC.

**Bottom line:** "DeviceMgr is one actor per device" gives you only
message-level mutual exclusion; the Python `MaapiLocker` gives you
transaction-level (run-level) mutual exclusion across the entire NSO data
plane. You don't have an equivalent and you're handwaving it.

Three options, ordered by my preference:

(a) **Route install RPCs through `DeviceMgr.configure` semantics, not
`rpc_xml`.** Hard, because the install RPCs aren't `edit-config` shaped.
Probably impractical.
(b) **Introduce a "device lease" concept on `DeviceMgr`.**
`dev.acquire_lease(owner, cb)` / `dev.release_lease(owner)` that gates the
confq during a lease. Other configures queue; rpc_xml from the leaseholder
pass through; rpc_xml from non-leaseholders error out. This is the closest
analog to MaapiLocker.
(c) **Accept the race and document it loudly.** "Sw-install requires the
operator to freeze upstream config changes for the device while a run is in
flight." Acceptable for an MVP if and only if explicit.

You're currently on (c) without realising it. Pick deliberately.

Other things lost from the Python scheduler that the design doesn't
acknowledge:

5. **Backoff persistence across restarts.** `01-software-install-logic.md` §10
   explicitly calls out "backoff state persists across worker restarts (it
   lives in CDB)" as load-bearing. Your `_schedule_retry` uses
   `after backoff: ...`. Acton's `after` is in-memory — controller restart
   loses the wakeup. The §10 idempotency invariant *requires* this to be
   persisted. You defer "exact lmdb persistence layout" to implementation in
   §13, but this isn't layout — it's whether you persist at all. The design
   should commit: `error_count.backoff` and the next-wake timestamp are
   persisted via `update_dynstate`, and on restore the runner schedules a
   fresh `after` with the remaining delay.

6. **Cancel latency / abort semantics.** Python sends SIGINT to the worker
   process — kills the in-flight run mid-RPC. Acton actors aren't kill-able
   mid-callback; you wait for the outstanding `rpc_xml` to return before the
   cancel takes effect. For IOS-XR's `_monitor_operation_log` polling at 600s
   timeout, cancel can take up to 10 minutes. Probably acceptable, but **state
   your cancel semantics** ("cancel takes effect at the next step boundary or
   completion of the current outstanding RPC, whichever is first") — silent
   "cancel may take 10 minutes" is a bug-shaped surprise.

7. **Multiple-RequestRunner-per-device guard.** You say "one `RequestRunner`
   per active request per device". What enforces that? `SwInstallRunner.on_conf`
   runs whenever upstream config changes. If two pack-data updates arrive in
   quick succession (or `on_conf` is re-entered concurrently — it's a
   `proc def` returning a closure, the actor model serialises it but the
   design doesn't say), you can spawn two runners before either marks prior
   requests obsolete. Make the runner-creation idempotent on
   `(device, request-id)` and write the rule down.

---

## Q3. `FileTransfer` interface (§9)

**Shape is broadly right, but the credential reuse story you're asking about
is a real gap. Right now it doesn't work.**

What the design says: `FileTransfer(local_path, remote_path)` and a
`file_transfer_factory: ?proc(swdev.DeviceMgr) -> FileTransfer`.

The credential reality from `device_meta_config.act` and
`netconf/src/ssh_client.act`:

- `DeviceMetaConfig.credentials` carries `username`, `password`,
  `private-key`. (Plus `initial-credentials` as a list for fallbacks.)
- DMC is injected into the *adapter* via `DeviceAdapter.set_dmc(...)`
  (`adapter.act:230`). The adapter holds it privately. Neither `DeviceMgr` nor
  `DeviceAdapter` exposes credentials on its public ABI.
- `netconf/ssh_client.act` consumes username + password OR private key
  directly.

Therefore: a `FileTransfer` constructed with only a `swdev.DeviceMgr` **cannot
get the credentials**. It would have to:

- Add a `get_credentials() -> Credentials` method on `DeviceAdapter` /
  `DeviceMgr` — viable but a real API extension you should commit to in this
  design, not implementation-time.
- Or accept DMC at the factory boundary —
  `file_transfer_factory: proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer`.
  Then the *transform* needs DMC visibility. RFS transforms get it; this
  isn't an RFS transform; how does it get it? You haven't worked this out.
- Or run file transfer through the netconf-adapter's *existing* SSH transport
  (paramiko-style scp-over-ssh on the same connection). That's option (D) you
  didn't list — and it's actually the cleanest because the netconf transport
  is already authenticated; you piggyback. Look at `netconf/src/ssh_client.act`
  and see whether it exposes a "spawn an scp channel on this transport"
  affordance — if not, that's the one stratoweave-internal change that
  unlocks credential reuse without leaking credentials out of the adapter.

Other §9 issues:

8. **§9.2 SUCCESS vs SKIP_STEP confusion.** You write "`CopyImage.pre_check`
   will then short-circuit to SUCCESS only if files are already present". Per
   the spec §6.1 / §3.4, "files already present" → `SKIP_STEP` (don't run
   execute). `SUCCESS` from pre_check means "continue to execute()". With
   `NoopFileTransfer`, the right behaviour is pre_check returns `SKIP_STEP`
   when files are already there (no upload needed) and `FAILURE` when they
   aren't. You said SUCCESS, which would call execute(), which would then
   `FAILURE` redundantly. Small but it shows you slipped the StepResult
   semantics here.

9. **Filename interpretation needs clarification in YANG.** You changed the
   filename description to "controller-local for SCP push; device-reachable
   URL for device-pull" (`yang.act:65-67`). With the FileTransfer abstraction,
   *who* interprets the path is the FileTransfer impl. Document that path
   semantics are defined by the configured FileTransfer, otherwise the YANG
   is ambiguous and a user can't predict what to write. Or split into two
   leaves and let YANG enforce.

10. **`Cleanup` step.** You list it as a `FileTransfer` consumer
    (`Cleanup.execute`: SCP-deletes destination_paths). True for IOS-XR, but
    for SROS the original code has no `Cleanup` step (it's IOS-XR only — see
    spec §6.2). Minor — make sure you don't accidentally add a Cleanup step
    on SROS.

---

## Q4. Things in the spec that you missed or got wrong

11. **`error_count.transient` / `error_count.other` cross-reset.** Spec §3.2:
    a FAILURE result resets `error_count.transient = 0` (and increments
    `other`); a WAIT result resets `other = 0`. Your `error_count: ErrorCount`
    field has no behaviour spec. Implementing this as "just two counters that
    both increment" silently loses the spec.

12. **Plan `refresh_steps` trigger discipline.** Spec §3.5 step 7: refresh
    runs **after every step** (not just at run start). This is what enables
    IOS-XR FPDs to surface mid-run after `SoftwarePrepare`. Your
    `ComponentPlan.refresh_steps` exists but there's no statement of when the
    runner calls it. Make the trigger explicit: "after each
    `_execute_step_action`, plan is refreshed against current state."
    Otherwise an implementer will only refresh at run start and break IOS-XR.

13. **`_execute_step_action` flush ordering.** Spec §3.5 step 6:
    `state.flush(t_write)` *only* if `result != FAILURE`. Then step 7:
    `refresh_steps`. Then step 8: persist plan again. That ordering is
    load-bearing — a FAILURE result must NOT persist the half-mutated state.
    Your design's "value-typed return-new-state" pattern makes this easier
    (you just don't accept the new state on FAILURE), but you should write it
    as an invariant. Otherwise an early implementation will accidentally
    persist failed state.

14. **`State.reset()`.** Spec §5.1: called from `CheckVersions` when the
    previously-Done version is no longer running on the device. This is the
    "device drifted, restart from scratch" lever. Per-OS State subclasses
    extend `reset()`. Your `state.act` plan doesn't mention `reset()`
    semantics at all. Without it, a drifted device on a previously-Done
    request gets stuck.

15. **`OperCdbLoggingHandler` filter semantics.** Spec §7 / §9: the run-log
    only captures records emitted in the package's logger and tagged with
    `swi_component`. Your `runlog.act` plan is "appends to oper data". If you
    append every log record that flows through, run-log gets polluted by
    everything else in the runner. The filter is part of the spec —
    log-record `swi_component` attribute decides whether it's persisted.

16. **Junos `dual_re` mid-request invalidation.** Spec §6.3: if
    `state.dual_re` changes between runs of the same request,
    ValidatePlatform returns FAILURE — request is invalidated. Your
    `state.act:StateJunos` plan has `dual_re: ?bool` but doesn't mention the
    cross-run invariant. Worth a line.

17. **IOS-XR `state.packages` extraction.** Spec §6.2 CheckVersions: parses
    package list from `.iso` (`get_version_from_iso`) or `.tar`
    (`get_file_packages`). This is **controller-side filesystem work** —
    opening archive files. Your `CheckFiles` is "no device interaction;
    trivial" and `CheckVersions` is described as "needs `dev.rpc_xml` +
    parsing version", but the IOS-XR variant *also* needs an archive parser
    on the controller. That's a non-trivial new dependency (iso/tar libs).
    Either acknowledge it in §9 alongside FileTransfer (it's controller-local
    I/O, not device-side), or scope IOS-XR base packages to "version is
    provided as a static leaf, no archive inspection" which would make IOS-XR
    not 1:1 with the Python.

18. **Junos parametrized step keys.** Your `StepKey(name, re_id)` is fine, but
    spec §6.3 has *both* `Done[re_id]` and a final unparameterized `Done`.
    Watch out: the dict keys must distinguish them; `re_id=None` for the
    trailing `Done` and `re_id="0"` (or whatever master id is) for the per-RE
    Done. You've got the right shape, just don't lose the trailing-no-re_id
    Done.

19. **`request-id` starting value.** Spec §1: "key starts at 1". Trivial nit;
    just record it next to `request_id: u32`.

20. **`internal-state` as oper-leaf.** Your §3.2 says "the Acton port emits
    `internal-state` as a debugging/inspection oper-data leaf only (a
    stringified summary)". The current YANG (`yang.act`) doesn't have an
    `internal-state` leaf at all (you dropped the YANG with the rest of the
    rewrite). Either restore it as an oper string, or remove it from the
    spec-mapping discussion. Right now §3.2 references something that doesn't
    exist in your YANG.

21. **Restart story for the `RequestRunner` actor.** The design defers "exact
    lmdb persistence layout" but the *who* is unspecified. `RequestRunner` is
    a child actor of `SwInstallRunner`, not a transform. Stratoweave's
    `update_dynstate` plumbing is *transform-level*
    (`_TransformTransaction.update_dynstate` → `swdb` → `restore` on next
    boot). You'll need to bubble RequestRunner state up to `SwInstallRunner`
    and persist it through the transform's dynstate, then re-spawn fresh
    runners on boot from the restored snapshot. This is real engineering and
    the design says nothing about it, in violation of spec §10 invariant 2
    ("step_executor is restartable").

---

## Smaller / softer

- **§7 transform shape.** A transform whose `transform_wrapper` returns
  `(Container, memory)` (empty downward) and that doesn't push device config
  is unusual in the codebase — sw-install is a parallel control loop, not
  part of the layer-stack data path. Worth asking: does it actually want to
  be a `Transform`, or something else (e.g., an actor at app-spec wiring
  time, like `DeviceRegistry` is)? The transform machinery gets you
  `update_oper`/`update_dynstate` for free, which is the appeal — but you
  don't get downward output (you don't want it) and you don't get the
  input-merge semantics (you only need to *read* the upper config). This may
  bite you on `gen_adata` wiring; sorespo's pattern doesn't have a precedent
  for "transform that's an observer". Worth a sanity-check with the platform
  owners.

- **Sibling repo vs. in-tree (❓1).** Sibling repo wins long-term but in-tree
  dramatically shortens the iteration loop while you're churning the
  interface. I'd start in-tree and extract once the API surface is stable.
  You currently have the scaffolding at `stratoweave/sw-install/` (sibling)
  — that's fine, just don't rewrite reflexively if you discover the design
  wants more from the platform than you thought.

- **`software-install-matrix`.** You're dropping it (§4.6). Spec §2.1
  confirms it's currently unused by the core logic. Fine — but capture it in
  a one-liner deferred-features list rather than silently dropping.

---

## Verdict on holding implementation

Items 1–7 (Q1+Q2) are design-level and need decisions before you write the
runner. Items 8–11 (Q3+credential reuse) need a decision before you write
`step.act`. Items 12–17 are spec-fidelity items that mostly just need to be
added to the design doc — they don't change the code shape but they will
change correctness. Items 18–21 are corrections.

Most worried about: **#5 (DeviceMgr serialization is not what you think)**
and **#1+#2 (lost stage-and-review semantics with no replacement)**. If those
hold up implementation, the time saved later more than pays for the wait.
