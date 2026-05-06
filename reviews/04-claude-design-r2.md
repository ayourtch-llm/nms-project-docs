# 04 — Claude (pty-1) Round-2 Design Review of `02-sw-install-design.md`

Reviewer: Claude (pty-1).
Scope: independent fresh-reader review of the v2 design doc, sanity-checked
against the live stratoweave codebase. Round-1 reviews
(`01-codex-design.md`, `02-claude-design.md`) and the integration writeup
(`03-integration.md`) were read **after** I formed my own assessment;
`03-codex-design-r2.md` was peeked at the end of the pass for divergence.
Where I converge with prior reviewers I say so explicitly so my voice is
distinguishable from theirs; the bulk of what follows is independently
sourced.

Materials read end-to-end:

- `docs/00-orientation.md`, `docs/01-software-install-logic.md`,
  `docs/02-sw-install-design.md`
- `stratoweave/stratoweave/src/stratoweave/ttt.act` (esp. lines 540–760,
  1640–1700, 1800–2300)
- `stratoweave/stratoweave/src/stratoweave/device.act` (esp. `DeviceMgr`,
  `set_dmc`, the public `rpc_xml`/`fetch_config`/`get_adapter` surface)
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act`
- `stratoweave/stratoweave/src/stratoweave/device_meta_config.act`
  (`Credentials`, `address[]`)
- `stratoweave/netconf/src/ssh_client.act` (`actor Client` over a
  subprocess `process.Process`)
- `stratoweave/sorespo/src/sorespo/{sysspec,rfs}.act`,
  `sorespo/src/sorespo/layers/{base_1,t_2}.act` for the working
  RFS-transform-with-`update_oper`/`update_dynstate` pattern
- `stratoweave/sw-install/src/sw_install/yang.act` (the existing Phase 4
  scaffolding — see §F1 below; it is **pre-v2**)

Tone: Andrew explicitly asked for hard pushback. The design is much
stronger than v1, and several of the v2 changes are clearly correct (the
control/oper split per A5, generation counters per A2, restoration of
the explicit `start-generation` per CL2). What follows is the list of
things I would not start coding around.

---

## Verdict

**Don't start §6/§8 code yet.** Six specific issues are blocking, several
of them correctness-shaped:

1. **§3.2 says the runner updates transform `memory` via `update_dynstate`.
   It can't.** `memory` and `dynstate` are different persisted attributes
   in `ttt.act`; the runner has no path to memory and nothing in
   `update_dynstate` writes it. (§A1)
2. **§8 says the per-device `DeviceRunner` "is the install lease for the
   device". It isn't** — it's a lease against sw-install's *own* code,
   not against `DeviceMgr.configure(...)` from RFS or
   `DeviceMgr.rpc_xml(...)` from anywhere else. (§A2)
3. **The transform wiring topology in §1/§2/§7 is under-specified and
   inconsistent with §8.7.** A `proc(Path, ?Layer) -> Node` factory
   wired *under* `/devices/device/<name>/software-pack/` gives one
   transform per device (sorespo's pattern); §8.7's "SwInstallRunner …
   owns `dict[device_name, DeviceRunner]`" implies a singleton over the
   whole tree. Which is it? (§A3)
4. **Generation counters trigger `update_dynstate` writes that re-fire
   `on_conf` through `Session.recompute()`.** The design specifies stale
   *step* callbacks but not stale *config-trigger* re-entry. Without a
   discipline this can double-start, double-consume generations, or
   churn LMDB. (§A4)
5. **Cancel semantics contradict themselves** (§4.4 vs §8.5 vs the
   user-observable contract); same for `enabled` (§4.6 still has a
   literal "wait, actually let's pick a distinct status" mid-prose).
   (§A5)
6. **Phase 4 `NoopFileTransfer` is internally inconsistent** with the
   `CopyImage.pre_check` path that's supposed to use it. NETCONF-only +
   "files pre-staged on device" is a coherent stance, but it requires
   *device*-side stat, not the absent `FileTransfer.stat`. (§A6)

Plus a handful of medium/low items below. A few converge with what
`03-codex-design-r2.md` raised; I'm flagging those rather than relitigating
them, with one or two places where I'd push harder than codex did.

I'm flagging **fidelity-vs-operability tradeoffs the design has quietly
made** as a separate section (§D), because they aren't bugs but the
design doesn't surface them and a reader going from `01-…-logic.md`
straight into `02-…-design.md` will miss them.

---

## A. Blockers

### A1. `transform_wrapper` memory vs `update_dynstate`: incompatible plumbing

**Where the design says it:** §3.2 ("The runner explicitly updates memory
whenever it materializes a new request (or transitions one), via the
transform's `update_dynstate` path"); §7.1 (`compute_memory_snapshot(cfg,
memory)`).

**What `ttt.act` actually does:** `memory` and `dynstate` are *separate*
persisted attributes:

- `_TransformTransaction.compute(...)` calls
  `self.function.transform_wrapper(merged, linked, self.memory,
  self.dynstate)` and stores the returned `(newout, new_memory)`. The
  `essay.1` (new_memory) is what gets persisted as `ATTR_MEMORY` in
  `db_ops` (`ttt.act:1942–1962`).
- `update_dynstate(dynstate)` only sets `self.dynstate` and triggers
  `Session(...).recompute()` (`ttt.act:1981–1987`). It does **not**
  touch memory.
- `finalize()` calls `function.on_conf(self.get(), self.memory)`
  (`ttt.act:1989–1993`).

The runner runs inside `act`-spawned actor scope; it has handles to
`update_dynstate` and `update_oper` from `TransformActorParams`
(`ttt.act:1897–1904`). There is **no** runner→memory write path. So a
sentence like "runner explicitly updates memory" is not implementable
against the current platform.

The clean fix is to put **everything** the runner needs to persist into
`dynstate` (last-observed pack-data hash, last-observed generations,
request id counter, plan, per-component State, `error_count`,
`next_wake_at`). `memory` either becomes a no-op
(`transform_wrapper` returns `(empty, memory)` unchanged) or is dropped
from the design entirely. The two-bucket split in §3 ("Transform memory
and create-request idempotency") is doing no useful work that I can see
and is creating a consistency surface where there shouldn't be one.

This is the same ground codex R2 covers (§"transform memory vs dynstate
is confused"); I'm independently confirming the diagnosis from a
separate read of `ttt.act` lines 1942–1993, and pushing further: drop
`memory` from sw-install's design, don't try to "use both cleanly."

### A2. "DeviceRunner's existence IS the install lease" — it isn't

**Where the design says it:** §8.2 ("**Why one per device, not per
request:** The actor's *existence* IS the install lease for the
device"); §8.1 motivates this against the Python `MaapiLocker`.

**What `device.act` actually exposes:** `DeviceMgr` has a public
`configure(new_conf, ...)` API used by RFS layer transforms
(`device.act:675`), a public `rpc_xml(cb, xml_rpc)` that any actor
holding a `DeviceMgr` reference can call (`device.act:783`), public
`fetch_config(done)` and `declare_subscriptions(...)`. None of these
gate on a sw-install-owned lease. There is no `DeviceMgr.acquire()` or
`adapter.acquire()` API. `DeviceAdapter` (`adapters/adapter.act:221–256`)
declares `set_dmc`, `configure`, `get_capabilities`, `get_modules`,
`fetch_config`, `get_config`, `rpc_xml`, `declare_subscriptions` —
nothing exclusive.

The Python `MaapiLocker.lock_partial(/devices/device[name=X])` was a
**system-wide** mutex over the device subtree: every other writer
blocked while the install was in flight. The Acton design's per-device
runner mutexes only sw-install's own logic. While the install is
running, nothing prevents:

- An RFS transform from pushing config via `DeviceMgr.configure`.
- A monitoring transform from issuing `rpc_xml` against the same
  adapter.
- A subscription from continuing to deliver oper updates.

For an OS upgrade — which by definition will reboot the device,
invalidate config, and then re-converge — that is a **real** safety
gap, not a fidelity nit. The fix has to live in the platform:

- Add `DeviceMgr.acquire_exclusive(owner_id, timeout, cb)` that gates
  the public `configure` / `rpc_xml` / `fetch_config` /
  `declare_subscriptions` paths until released.
- Or weaken the spec mapping explicitly: "sw-install does not prevent
  RFS or external rpc_xml callers from interleaving with the install,
  unlike the Python `MaapiLocker`. Operators must ensure no RFS layer
  is actively reconciling against a device under upgrade." Document the
  rough edges in the README.

The design needs to pick one. Right now §8.1–§8.2 reads like the gap is
solved when it isn't.

This is the same finding codex R2 lists at "high"; I read this
independently before consulting that review. The convergence is real,
and the fix needs to be a platform-side commitment (or an explicit
downgrade), not glossed.

### A3. Wiring topology — is sw-install one transform or N?

**Where the design says it:** §1's API
`make_sw_install_transform(...) -> proc(ttt.Path, ?ttt.Layer) ->
ttt.Node` matches the rootgen factory shape that `Transform()` accepts.
§2 says "Wire a `Transform` at that layer attached at the per-device
path." §8.7 says "**The `SwInstallRunner` actor (transform's
`act`-spawned root) is a coordinator that owns `dict[device_name,
DeviceRunner]` and dispatches incoming config changes per device.**"

These two pictures are inconsistent. Sorespo's actually-working pattern
is the *first* one: `t_2.act:18` shows
`ttt.List(ttt.RFSTransform(sorespo.rfs.BBInterface, …, BBInterfaceTransformCtor, …), [<key>])` —
the transform factory is wrapped in a `ttt.List`, so one transform
instance is created **per list entry** (per backbone-interface), each
with its own `update_dynstate` / `update_oper` and its own
`BBInterfaceTransform` actor (`rfs.act:189`). That is the stratoweave-
native "per device" shape. There is no `dict[name, ...]` coordinator.

If sw-install follows that pattern, **`SwInstallRunner` should not
exist** — there is just one `DeviceRunner` per per-device transform
instance, each with its own dynstate slice. That cleanly solves the
update_dynstate concurrency question §8.4 hand-waves over.

**But**: trigger detection also needs the global `/software-install/...`
config (the pack library, the `enabled` master switch, error-handling
policy). Per-device transforms wired under `/devices/device/<name>/`
don't see those. So either:

- **Option A — single top-level transform.** Wire one Transform at
  some ancestor that sees both subtrees. This makes `SwInstallRunner`
  real, requires it to dispatch to a `dict[name, DeviceRunner]`, and
  reintroduces the dynstate-concurrency problem (one handle, one tree,
  N writers).
- **Option B — per-device transform + global subscription.** Each
  per-device transform owns its `DeviceRunner`. Global config is
  obtained via `Layer.declare_subscriptions(...)` against the lower
  layer (existing platform mechanism, `ttt.act:735`; `LayerTreeProvider`,
  `ttt.act:960`). This is more code but matches sorespo's grain.
- **Option C — fall back to §7.2.** Top-level actor wired in `sysspec`,
  not as a Transform at all. Already in the design as a fallback.

The design treats Option B as if it's the same as Option A and waves
at C. It isn't, and the right pick determines a lot of downstream
decisions — including whether `dynstate` snapshots are per-device
(simple, parallel, no contention) or global (one tree, requires
serialization). My recommendation: write the wiring out concretely
before writing runner code. I lean B if the platform side is willing
to commit a small documented subscription example; otherwise C, since
"transform that emits no downward config and observes two disjoint
roots" is genuinely the wrong shape for the substrate.

(Codex R2 hits the same shape at its "high: proposed transform
attachment point cannot see the required data". I converge with that
diagnosis. My added contribution: list the three concrete wirings and
note that B is achievable with `declare_subscriptions` already exposed
by `Layer`.)

### A4. `update_dynstate` re-entrancy through `Session.recompute()`

**Where the design says it:** §8.4 ("every state-bearing change …
emits a serialized snapshot via `update_dynstate(...)`"); §8.3 covers
stale **step** callbacks via generation tokens.

**What `ttt.act` actually does:** `_TransformTransaction.update_dynstate`
not only sets `self.dynstate` but immediately constructs
`Session(...).recompute()` (`ttt.act:1981–1987`). Recompute walks the
transform stack; `finalize()` then calls `function.on_conf(self.get(),
self.memory)` whenever there's a running config (`ttt.act:1989–1993`).

So **every dynstate write the runner makes will cause `on_conf` to
re-enter the runner** with the same cfg. Multiple consequences the
design doesn't address:

1. **Spurious trigger consumption.** If the runner naively re-checks
   `request-generation > last_observed_generation` on every `on_conf`,
   the second call sees the generation already advanced, no-ops, and
   moves on. That's actually fine — but only if the runner's
   "last_observed_generation" was bumped *durably* before the runner
   started doing work for that generation. If the bump happens after
   work completes, a runner crash mid-step plus restart re-fires the
   trigger.
2. **Self-triggering loops.** If anything in the runner's reconciliation
   logic writes a value that ends up in `cfg` (via, say, transform
   memory leakage — see A1), `on_conf` repeats forever. Per A1, drop
   `memory` and this is moot — but it should be designed in as an
   invariant.
3. **LMDB churn.** A chatty step (e.g., IOS-XR install op-id polling
   every few seconds) writes dynstate frequently. Each write recomputes.
   Each recompute is `Session(...).recompute()` work. The cost of a
   chatty install is now O(steps × dynstate-writes-per-step ×
   recompute-cost). Some debounce/coalescing discipline is needed.

Required additions to §8 before coding:

- `on_conf` is a pure idempotent reconciliation function over `(cfg,
  dynstate)`. It never starts work as a side effect of being called; it
  decides what work *should* be in flight and dispatches if not already
  in flight.
- Generation observations are durably persisted in dynstate **before**
  any work begins for that generation, so a crash leaves the trigger
  consumed.
- Dynstate writes for high-frequency state (run-log entries, polling
  progress) coalesce — either via a periodic flush actor or via "write
  on state-class transition only, not on every poll".

Codex R2 catches the same issue ("observer Transform has a re-entrancy
risk through update_dynstate"); my framing emphasizes that this is
**not** the same as the §8.3 stale-step-callback problem, which the
design does handle. They are two distinct re-entrancy vectors, and
§8 currently only addresses one.

### A5. Cancel and `enabled` are unfinished

**Cancel.** §4.4 says "the runner sets request status to `cancelled`
immediately and marks all subsequent callbacks for the cancelled
request to no-op via the generation token… the user observes
`request.status = cancelled` once the in-flight RPC returns."

That's contradictory at face value: status either flips immediately
(operator sees `cancelled`) or after the RPC returns. §8.5 has the
same wording. The cleanest contract introduces an intermediate state:

- `request-status: ... | cancelling | cancelled`
- `cancel-generation > last_observed_cancel_generation` →
  status becomes `cancelling`, generation token bumps, in-flight
  callbacks drained.
- When the in-flight RPC returns (or the actor concludes nothing is
  outstanding), status becomes `cancelled`.

That's honest about what an operator sees. The current spec is "status
is `cancelled` but the device may still be mid-RPC for up to 10
minutes" — which is lying to the operator. If you don't want to add
`cancelling` to the enum, then the alternative is "stay in
`processing` until the in-flight RPC returns, then go to `cancelled`"
— but then `cancel-request` looks like it's doing nothing. Pick one.

**`enabled`.** §4.6 line "**In-flight runner reaches a cooperative
stop point** (current step or current RPC completes), then pauses with
`request.status = waiting-confirmation` — wait, actually let's pick a
distinct status." That's a literal mid-edit comment left in the doc.
Either commit to `paused` (the proposal at the end of §4.6) and
rewrite the paragraph, or remove it. If `paused`, a state diagram
showing transitions across `enabled` toggling × `request.status` × the
explicit-start gate clarifies a lot. Specifically I want answers to:

- If `enabled=false` while a request is mid-backoff (`next_wake_at`
  set): does `next_wake_at` carry over, does status become `paused`,
  does re-enable consume the existing `start-generation` intent or
  does it require a fresh increment?
- If `enabled=false` is set before any `start-generation`: stays in
  `unprocessed` or transitions to `paused`?
- Does toggling `enabled` bump the device's generation token (hence
  invalidating in-flight callbacks)?

Codex R2 makes the same call about `enabled`; I'm pushing harder on
cancel by recommending a `cancelling` status in the enum.

### A6. Phase 4 `NoopFileTransfer` does not coexist with `CopyImage.pre_check`

**Where the design says it:** §9.2 ("Phase 4 ships `NoopFileTransfer`:
every method except `caps()` returns `NotImplementedError`; `caps()`
returns all-false. `CopyImage.pre_check` returns `SKIP_STEP` if all
files are already present (verified via `stat`/`list`), or `FAILURE`
with a clear "no FileTransfer configured" message if files are
missing"); §9.6 ("NETCONF-only Phase 4 (NoopFileTransfer) requires the
file to already be on the device at the given path").

The two paragraphs together require `NoopFileTransfer.stat(...)` to
**actually work** in Phase 4 (it's the basis for SKIP_STEP), but §9.2
says it returns `NotImplementedError`. So either:

- `NoopFileTransfer.stat` returns `NotImplementedError` and
  `CopyImage.pre_check` always returns FAILURE in Phase 4 → SROS
  Phase 4 cannot run at all, contradicting §11.4.
- `NoopFileTransfer.stat` is real (somehow remote-stat-via-NETCONF) →
  it's not a Noop.

The honest fix is to split the abstraction codex R2 already proposed:

- `RemoteFileInspector` (`stat`, `list`) — feasible over NETCONF for
  SROS (`file dir` / `oper-file` queries) and probably IOS-XR.
- `FileTransfer` (`put`, `delete`) — Phase 5, needs SCP/SFTP/etc.
- `LocalFileInspector` for `CheckFiles` (controller-side, exists
  trivially via Acton stdlib).

In Phase 4, SROS would have a real `RemoteFileInspector` over NETCONF
and a `NoopFileTransfer` for `put`/`delete`. `CopyImage.execute`
becomes "if any file missing → FAILURE with clear message" — same
result as today, but the SKIP_STEP path actually works for the
already-on-device case, which is the realistic Phase 4 scenario (an
operator pre-staged the image).

Independently arrived at; codex R2 reaches the same conclusion and
proposes the same split. Worth adopting verbatim.

---

## B. Section-by-section interrogation

### §3 — config/oper boundary, where state lives

The control/oper split is the right call (and the v2 fix per A5 is a
clear improvement over v1). My remaining concerns:

- **§3, §3.2 and §3.3 — three different stories about "memory".** §3
  table says "current generation tokens, **next_wake_at**,
  **error_count.backoff**" go to dynstate. §3.2 says memory holds
  `last_pack_data`, last-observed generations. §3.3 says
  `internal-state` is dropped because dynstate replaces it. The
  cleanest version is the §3-table + §3.3 view: **everything operational
  goes in dynstate**, drop §3.2 entirely, treat memory as no-op for this
  module. (See A1 for why.)

- **`internal-state` operability regression.** §3.3 / CL14 drops the
  YANG `internal-state` leaf because dynstate replaces it. That's
  defensible but the design doesn't surface the tradeoff: in the
  Python original, an operator debugging a stuck install can `show
  device X software-pack request 5 component base-25.3.1
  internal-state` over CLI/RESTCONF to see the State JSON. With the
  v2 design, **State is invisible to NETCONF/RESTCONF** and the only
  way to inspect it is to read lmdb on the controller host. That's a
  meaningful operability regression. Recommendation: project
  selected dynstate fields (`destination_volume`, `destination_paths`,
  `boot_time`, `op_id_*`) into the per-component oper subtree as
  read-only diagnostic fields. Make it a documented diagnostic
  surface, not an opaque blob.

- **`error_count.backoff` and `next_wake_at` should be in oper too.**
  Codex R2 raised this; I agree and will repeat: if the next retry is
  90 seconds away, the operator wants to *see* that in the oper
  subtree. Don't hide it.

- **Generation counter overflow on backup/restore.** The design's
  argument for generation counters (§3.1: "they survive backup/
  restore") only holds if backup→restore preserves both the current
  config value and the runner's persisted "last observed" in
  dynstate. If a user restores config from backup, the generation
  counter goes BACKWARDS relative to the runner's persisted last-
  observed. The runner's `current > last_observed` check then
  evaluates false forever — until the user knows to bump the
  generation. Recommendation: document this explicitly, and if the
  platform has a "config restore" event, reset all `last_observed_*`
  counters at restore time.

### §4 — generation-counter substitute for actions

The pattern is the right reactive idiom. Things to nail down:

- **Per-request scoping.** `confirm-all-generation`,
  `start-generation`, `cancel-generation`, `clear-run-log-generation`
  are all keyed only by device. If a new `request-generation` lands
  between an operator reading the request id and writing the
  `start-generation`, the start applies to the new (different)
  request. Add an optional `target-request-id` next to each trigger,
  or define explicitly that "triggers always apply to the device's
  latest request at observation time." For automation safety I'd
  rather have the optional scope leaf. Codex R2 raised this too.

- **`last-create-result` is per-device, not per-call.** Two operators
  writing pack assignments concurrently stomp on each other's
  `last-create-result`. For interactive use that's tolerable; for
  automation it's a regression vs. the original action's per-call
  return. Either accept the limitation and document, or add a
  caller-supplied correlation id leaf to `control/` that gets echoed
  in `last-create-result`.

- **`confirm-steps` per-request override.** The Python YANG had
  `request/confirm-steps?` overriding the global. v2 moves user
  inputs out of `request[]` (correctly) but the override is missing
  from `control/`. Either add `control/request-options/confirm-steps?`
  (or similar) sampled at request materialization, or list the
  per-request override as deferred/dropped in §15.

- **Run-log retention sizing (codex CL/§: 1000).** 1000 entries per
  request sounds generous. Dimensional check: SROS has ~13 steps per
  component × 1 component × ~5 messages per step = ~65 entries per
  successful run, easily 200 in a retry-heavy run. IOS-XR with FPDs
  can produce much more. 1000 covers a few runs of the same request
  before clearing. Fine, but: (a) when the buffer drops oldest, the
  YANG list silently truncates — operators investigating a failure
  may not realize they missed entries. Either expose `dropped-count`
  or `oldest-when` in oper, or document. (b) The bound is per-request,
  not per-device — ten requests carry 10× the budget. Worth a
  sentence.

### §6 — Plan + step semantics

This section is good — the v2 changes (CL8 flush ordering, A8 refresh
discipline, A4 retry counter semantics) all match the spec correctly.
Two additions:

- **Step `pre_check` and `execute` have callback signatures:**

  ```
  proc def execute(self, state, ops, ft, cb: action(StepResult, NewState, ?Exception) -> None)
  ```

  The `cb` is `action(...)`, which means it dispatches on whichever
  actor mailbox owns `cb`. The runner needs to make sure that mailbox
  is the per-device runner, *not* the step itself, otherwise the
  generation-token check (§8.3) doesn't gate against stale state. State
  this invariant in §6.

- **`StepKey` equality / hashing.** Plans use `StepKey` as `OrderedDict`
  keys. `class StepKey(value)` with `(name: str, re_id: ?str)` is
  fine, but the design should explicitly state value equality is
  structural; this is what makes Junos's `re_id` = "0" / "1" /
  `None` (trailing Done) all distinct keys.

### §7 — Transform observer vs top-level actor

See A3 for the wiring topology issue. Once that's resolved, §7.1 is
plausible (sorespo's `BBInterfaceTransform` is the existence proof).
§7.2's "fallback" framing is too soft: if the chosen wiring is "single
top-level transform with multi-root view," the substrate is genuinely
awkward — empty `transform_wrapper` output is fine
(`compute(...)` produces no diff), but treating the transform as a
multi-root observer when sorespo proves the per-list-entry pattern
works would be deliberately picking the harder shape.

§7.2 mentions `gdata.TreeProvider`. That works (it's how
`device.act:207` `DeviceTreeProvider` exposes oper data). The fallback
isn't strictly worse than the Transform path; for sw-install it might
be cleaner. The design should pick *now*, not "start with §7.1, switch
if friction."

### §8 — DeviceRunner, restart, cancel, backoff

Per-device runner is the right unit. Stale-step-callback discipline
via generation tokens is the right mechanism. Persistent `next_wake_at`
is the right backoff fix.

Outstanding issues already covered above:

- Lease scope (A2) — must address.
- `update_dynstate` re-entrancy (A4) — must address.
- Cancel status semantics (A5) — must address.

Additional:

- **Spawn-vs-inject restart story.** §8.4 says: "the platform restores
  the transform's dynstate from lmdb. The transform's `act` callback
  inspects the restored state on first invocation and respawns
  DeviceRunners with their persisted `RequestState` injected at
  construction time." Two questions I'd like answered before coding:

  1. The `act` callback returns an `on_conf` closure. The first
     `on_conf(cfg, memory)` after restore receives the restored
     dynstate too (in `ttt.act:1989` `function.on_conf(self.get(),
     self.memory)` — note `self.dynstate` is restored separately by
     `restore(...)` in `_TransformTransaction:1964`). The runner needs
     to reconcile from cfg + dynstate on first call. Spell that out.
  2. What if the runner observes that a request was mid-`processing`
     when the controller died? The Python's per-step bookkeeping
     (`processing` status, `op_id_*` already recorded) gives a clean
     resume point. The Acton port should call out: on restart,
     `processing` status becomes `failed-transient` (so the scheduler
     re-runs from the mid-step) — *or* the runner re-enters the same
     `pre_check` of the in-flight step, which is supposed to be
     idempotent. State which.

- **Device not in registry.** If `dev_registry.get(name)` returns a
  DeviceMgr that hasn't yet had `set_dmc` called (no DMC, NoAdapter
  active), the runner cannot do anything useful. What's the
  observable behavior? Spec a "waiting-for-device" status, or fail
  the request with a clear message. Right now the design assumes
  every per-device transform has a real DeviceMgr backing it.

- **`history: list[RequestState]` retention.** The design says "for
  the request[] oper projection (bounded)" but doesn't define the
  bound. Codex R2 hits this — I agree the latest non-terminal +
  latest terminal-per-status is the minimum for idempotency; older
  history can be bounded-N or retained indefinitely depending on
  operator expectations. Decide.

### §9 — FileTransfer + CLI

- **§9.4 credential reuse, option (c) is harder than admitted.**
  The design treats option (c) ("piggyback on the netconf adapter's
  existing SSH transport") as the cleanest with caveat "requires
  `netconf/src/ssh_client.act` to expose a 'spawn channel'
  affordance (it currently doesn't)."

  That phrasing under-sells the cost. `ssh_client.act` is **not** an
  SSH library — it's a wrapper around the OpenSSH `ssh` *subprocess*
  (`actor Client(...)` constructs a `process.Process`, sets up
  `_p_on_stdout` / `_p_on_stderr`, line 39–116). One SSH connection ≡
  one subprocess. There is no in-process SSH library to multiplex
  channels over. "Spawn channel" would mean either:

  - Add an OpenSSH `ControlMaster` / `ControlPath` story and have
    the SCP/SFTP path attach to the existing master via
    `-S /path/to/ctl`. Doable, but real platform work, and brittle on
    edge cases (Windows, restricted PAM).
  - Spawn a *separate* `ssh`/`scp`/`sftp` subprocess that reuses the
    *credentials* (private key path or password) but opens a fresh
    TCP connection. That is actually option (b) in disguise — same
    creds, different connection — and dodges nothing structural.
  - Move `netconf/src` to an in-process SSH library. Big undertaking.

  My read: option (c) as currently described will collapse into option
  (b) (DMC at factory). That's fine, but the design should drop
  option (c) language and own the actual platform ask: **expose `dmc`
  on `DeviceMgr`** (one-liner getter). `DeviceAdapter.set_dmc(...)`
  already takes DMC by value (`adapter.act:230`); adding
  `DeviceMgr.get_dmc()` does not increase the credential blast
  radius beyond what already exists. The design's principled
  objection ("(a) leaks credentials out of the adapter") doesn't
  actually apply to a `DeviceMgr.get_dmc()` getter — DMC isn't owned
  by the adapter, it's owned by `DeviceMgr` (line 288: `var dmc:
  DeviceMetaConfig = DeviceMetaConfig(...)`).

  Recommendation: drop the (a)/(c) framing, plumb a `DeviceMgr.get_dmc()`
  getter in the platform, FileTransfer factory takes `(DeviceMgr,
  DeviceMetaConfig)` and reads creds at *use* time (fresh each
  transfer), not at construction time (DMC is mutable —
  `set_dmc(...)` is called repeatedly). That detail is missing from
  the design and will bite.

- **§9.6 filename interpretation is YANG-fragile.** The design says
  filename semantics are "defined by the configured `FileTransfer`
  implementation". That makes the YANG schema's behavior depend on
  *runtime configuration*, which is a form of leaky abstraction
  through the data model. Two ways to fix:

  1. Commit to one semantics per OS / per pack: "the `filename`
     leaf-list is the absolute controller-local path". Implementations
     that need a remote URL or device-pull use a separate
     `remote-url` leaf instead.
  2. Use a YANG `union` of `inet:uri | string` so the leaf is at
     least typed.

  The current design lets a misconfiguration silently change what
  `filename` means. That's especially bad for backups.

- **§9.7 CLI / TextFSMPlus is overcommitted for Phase 4.** Codex R2
  pushes the same point harder; I agree. The DeviceOps facade is
  worth keeping (it's a small surface, decouples Phase 4 from Phase
  5, and matches the Python `NokiaSros{Cli,Netconf}Strategy` pattern).
  Everything else — TextFSMPlus templates, Send/Preset/Done line
  actions, the aycalc-equivalent expression evaluator, the acton-utils
  textfsm extension, the prompt synchronization story, the secrets-
  in-presets-and-logs story — should be in a separate ADR
  (`docs/adr/cli-driver.md` or similar). The current §9.7 is ~110
  lines of design for a Phase 5 dependency that doesn't exist yet,
  inside a doc whose explicit scope is "Phase 4 stop here for round-2
  review before resuming Phase 4 implementation continues" (§16).
  Cut it; refer to the ADR.

---

## C. Smaller items

### C1. yang.act in `stratoweave/sw-install/` is **pre-v2**

`stratoweave/sw-install/src/sw_install/yang.act` (167 lines) reflects
v1: `request-status` enum lacks `paused` (§4.6), no `control/` subtree,
no `last-create-result`, no per-device augmentation, no `scp-port`. Its
revision description still mentions a `cancel-requested` leaf that v2
replaced with `cancel-generation`.

Either the design's §11.1 ("Skeleton + revised YANG (this iteration):
…`yang.act` complete with control subtree + per-device augment") is
the next code task — in which case the design doc and the live
yang.act will drift further until that lands — or the existing yang.act
should be updated as part of accepting v2. The review brief said
"almost no implementation written yet" so I assume the former, but
flagging because anyone reading both will get whiplash.

### C2. Doc has two unfinished editing remnants

- §4.6 line "wait, actually let's pick a distinct status" (covered in
  A5) is mid-edit prose left in place.
- §14 is literally `## 14. (deleted from v2 — was the previous "Ready
  for review" section)`. Either drop the header or leave a note
  explaining the renumber. Right now it's a TOC stub that goes
  nowhere.

Both are small but shouldn't be in a doc that's being asked to "stand
on its own" for a fresh reader.

### C3. `scp-port` placement is a model change, not "preserved"

The original Python YANG had `scp-port` directly on the per-device
augment (`/devices/device/scp-port?`). v2 moves it inside
`/devices/device/software-pack/scp-port?`. This means **removing the
software-pack association also removes the scp port**. Any future
non-sw-install use of an SCP port (e.g., a config-backup tool) cannot
share the leaf. Codex R2 caught this; I'd push: keep `scp-port` as a
direct device augmentation (or move it into stratoweave's device meta
config altogether — it's really an SSH/SCP transport setting, not a
sw-install setting).

### C4. `software-install-matrix` — call out the deletion

§4.7 lists it as "drop" — fine — but `01-…-logic.md:39` notes it as
"unused by core logic". §15 deferred-features lists it. Make it
unambiguous in §4.7: "dropped from the YANG; not modelled in Acton;
re-add only if a real use case surfaces." (As written, the table
entry "drop, log in §15 deferred-features" reads ambiguously — does
"drop" mean drop from YANG, or drop from logic only?)

### C5. `request-id` starts at 1, OK, but where is it persisted?

The design's `RequestState.request_id` is in dynstate. Good. The
**counter** (next id to assign) needs to be in dynstate too, separate
from the per-request state. State explicitly that the next-id counter
is a top-level dynstate field, not derived from `max(request[].id) +
1` (which is wrong if old requests are pruned).

### C6. Run-log key uniqueness

YANG keys `run-log[when]` by `yang:date-and-time` (microsecond). If
two log entries land in the same `actor` tick, they could share a
timestamp. The Python original used Python `datetime.now()` and a
single thread; collision is rare but possible. In Acton with multiple
fan-out sources, collisions become more likely. Either:

- Add a sequence number leaf to the run-log entry and key on `(when,
  seq)`.
- Or keep `when` as the key and document collision behavior (drop?
  overwrite? bump nanosecond?).

Either is fine; pick. (Codex R2 raised this at "low".)

### C7. Step `next_step()` can JUMP forward past the requested target

Logic doc §4.2 says "A step's `next_step(state)` returning `X` causes
the inner loop to **skip** all steps strictly between current and
`X`." `02` §6.1 sketch:

```
def next_step(self, state) -> ?StepKey: ...
```

needs to spell out: the runner's plan loop *advances the cursor* to
`X`, marking all skipped steps as `skipped`. If `X` is not in the
plan (e.g., a typo), what happens? Python raises in
`refresh_steps` ("regression guard"). State the same in Acton with a
`FAILURE` outcome plus a clear log.

### C8. `DeviceOps` strategy selection on capability change (codex R2 also)

If `dev.get_capabilities()` changes between runs (the device upgrades
its OS, capabilities shift) the strategy chosen at SrosOps construction
goes stale. State whether SrosOps is rebuilt per run, or capability
checks are re-evaluated each call. Codex R2 covered this; I agree with
"snapshot per run, fail clean on incompatible drift".

---

## D. Fidelity-vs-operability tradeoffs the design has *quietly* made

These are not bugs. They are design choices the spec doesn't surface
prominently, and a fresh reader going from `01` to `02` will miss
them. List them under a "Conscious deviations from the Python spec"
section in `02`.

1. **`internal-state` is no longer NETCONF/RESTCONF-visible.** Per
   §3.3. Operator debugging is now controller-side (lmdb) only. (See
   §A1/§3 above for the recommended diagnostic-projection fix.)
2. **Run-log is bounded at 1000 entries/request.** Per §6.6. Python
   was unbounded. (See §B/§4 above.)
3. **Cancel takes effect at "next step boundary or RPC return."** Per
   §4.4/§8.5. Python `cancel-request` SIGINT'd the worker. (See §A5.)
4. **Per-device install lease is sw-install-internal only.** Per §A2.
   Python `MaapiLocker` was system-wide.
5. **Generation counters can go backward on backup-restore.** Per §B/§3.
   Python had no equivalent generation concept.
6. **`software-install-matrix` dropped.** Per §4.7. Python had it but
   unused.
7. **`vrp` enum kept; no `vrp` step module.** Per §4.7. (Already
   surfaced in §15; fine.)
8. **`Snabb`/`ONS-TL1`/`HGW` dropped.** Per logic doc §6.4. (Surfaced
   in §15; fine.)
9. **Action-style return values replaced by per-device `last-create-
   result`** (single-slot, last writer wins). Per §4.1.
10. **CLI strategy code paths exist in Phase 4 but raise
    `NotImplementedError`.** Per §9.7.

A reader scanning `02-sw-install-design.md` from top to bottom doesn't
get this consolidated picture. A "fidelity differences" subsection
(maybe §15.5) would do a lot for review velocity and make `02` stand
on its own without forcing readers to cross-reference `01`.

---

## E. Onboarding doc evaluation

### `00-orientation.md`

**As a fresh reader, this is a strong onboarding doc.** It gives the
right mental model in the right order (TL;DR → repo map → stratoweave
big abstractions → software-install vocabulary → mapping). The "where
to look" pointers in Part 4 are useful.

The **one significant gap**, which I felt while reading `02`, is that
`00` introduces transforms as "pure-ish functions from above to
below" (line 48) and doesn't introduce **`update_oper`**,
**`update_dynstate`**, **`memory`**, **`dynstate`**, or the actor-
spawning `act` callback at all. Those four concepts are what `02` is
*built on*. A fresh reader hitting `02` then has to scramble back to
`ttt.act` to figure out what's going on.

Recommendation: add a ~half-page subsection to Part 1, "Operational
state and the observer pattern", referencing `sorespo/rfs.act`
`BBInterfaceTransform` as the worked example. Doesn't need to be deep;
just establish that the platform supports a transform that produces no
downward config but publishes oper state and persists private dynstate
through an `act`-spawned actor.

(Codex R2 makes the same recommendation. Independently arrived at;
the gap is real.)

A smaller polish item: the "key actors / concurrency" table (line 102)
mentions `actor DeviceMgr` but doesn't mention the *adapter pattern*
underneath it. Worth a sentence: "DeviceMgr delegates protocol-specific
work to a `DeviceAdapter` (NETCONF, mock, …); modules that drive
devices typically interact only with `DeviceMgr`'s public surface and
don't reach into the adapter."

### `01-software-install-logic.md`

**This is the strongest doc in the set.** It's implementation-grade.
I'd implement directly from it without reaching back to the Python.
Every behavior I would have wanted to verify (consecutive-counter
semantics, `next_step` jumping, plan refresh monotonicity, Junos
`re_id` parametrization, `internal-state` restart story, `state.reset`
trigger from `CheckVersions`) is documented at the right level of
detail. §3.5's `_execute_step_action` table is exemplary — it's the
kind of "this is exactly what happens, in order" specification that
turns ambiguity into bug prevention.

Two micro-additions would make it even better:

1. **A "Python-vs-port intentional differences" table** once `02`
   decisions are settled, complementing §11. Right now `01` says "out
   of scope of the port"; `02` is then where the *intentional
   semantic deltas* land. A short cross-reference makes both docs
   easier to maintain.
2. The original YANG `enabled` description (per logic spec §2.1
   "Worker activation") is interpreted in `02` §4.6 as "no step
   execution starts" — a stronger meaning than the Python YANG
   description suggests. Worth calling out as one of the conscious
   deviations (see §D).

### `02-sw-install-design.md`

Already covered above. Summarizing the doc-level (not architecture-
level) concerns:

- Two unfinished prose remnants (C2).
- Mixes settled decisions, assumptions, and open questions throughout.
  Round-1 marked some of this with **❓DECISION**, **⚠️ASSUMPTION**,
  **🆕** — those help, but several settled-since-round-1 items still
  read as "I'm starting with X, will switch if friction" (e.g., §7.1
  vs §7.2, §9.4 (a)/(b)/(c)). Decide before round-3.
- The conscious-deviations list (§D) is missing.
- §9.7 should be moved to a separate ADR.

---

## F. Recommended pre-implementation edits, in priority order

Numbered list, blocking items first.

1. **Pick the wiring topology in §1/§2/§7/§8** (one of A/B/C in §A3
   above). Spell out where the transform lives, how it gets both the
   global `/software-install/` and per-device config, whether
   `SwInstallRunner` exists or each per-device transform's actor is
   the runner.
2. **Drop transform `memory` from sw-install's design** (consolidate
   ownership in dynstate). Rewrite §3.2 and §7.1 accordingly. Resolves
   §A1.
3. **Decide on the device-operation lease.** Either commit to a
   platform-side `DeviceMgr.acquire_exclusive(...)` API as a Phase 4
   prerequisite, or explicitly downgrade the fidelity claim in §8 with
   a "what this does *not* protect against" paragraph. Resolves §A2.
4. **Specify `on_conf` re-entry discipline** as an §8 invariant
   (idempotent reconciliation; durable generation observation before
   work; coalesced dynstate writes for high-frequency state).
   Resolves §A4.
5. **Cancel and `enabled` state machines.** Add `cancelling` to the
   request-status enum (or commit to "stay processing until RPC
   returns, then cancelled"); finish the `enabled=false` paragraph;
   add a small state-transition table covering `enabled` × in-flight
   request × backoff. Resolves §A5.
6. **Split `FileTransfer` into `RemoteFileInspector` (Phase 4
   over NETCONF) + `FileTransfer` byte-mover (Phase 5).** State that
   `LocalFileInspector` is Acton-stdlib filesystem. Rewrite §9.2,
   §9.3, §9.6 around the split. Resolves §A6.
7. **Plumb `DeviceMgr.get_dmc()` (one-line platform addition);
   FileTransfer factory takes `(DeviceMgr, DeviceMetaConfig)` and
   reads creds at use time, not construction.** Drop the (a)/(c)
   framing in §9.4. Documents the actual platform ask honestly.
8. **Add per-request scoping (optional `target-request-id` leaf) to
   start/cancel/confirm-all/clear-run-log triggers** in §4.
9. **Move §9.7 to `docs/adr/cli-driver.md`** (or similar). Keep only
   `DeviceOps` in §9 of the design.
10. **Add a "Conscious deviations from spec" subsection** (§D content).
11. **Add the operational-state subsection to `00-orientation.md`** (§E).
12. **Clean up unfinished prose** (§4.6 mid-edit, §14 placeholder).
13. **Update `stratoweave/sw-install/src/sw_install/yang.act` to v2** as
    the first concrete code task once `02` is signed off.

---

## G. What looks right and shouldn't change

Calling out the parts that are clearly correct so they're not
disturbed in the rewrite:

- v2's split between `control/` (config triggers) and `request[]`
  (oper-only) per A5. This is unambiguously the right shape.
- Generation counters as the trigger mechanism (vs edge-triggered
  booleans). Idempotency under config replay is a real property that
  edge-triggers don't have.
- Per-device runner as the unit of step serialization (within sw-
  install) — *with* the lease caveat in A2.
- Per-class retry budgeting (`error_count.transient` /
  `error_count.other` consecutive, with reset rules) — A4 is a clean
  spec match.
- Plan refresh monotonicity + after-every-step trigger (A8/CL8).
- Generation tokens for stale step callbacks (A4 / CL7 / §8.3) — note
  this is distinct from the `update_dynstate` re-entrancy issue in §A4.
- Dropping NSO action nodes for reactive triggers, with the noted
  ergonomic patches (last-create-result, confirm-all-generation,
  clear-run-log-generation).
- Junos `(class_name, ?re_id)` step keying.
- NETCONF-only Phase 4 scope (assuming §A6 is fixed).

The architecture direction is sound; the items in §A and §B are
correctness/clarity gaps inside that direction, not arguments for a
different one.

---

## H. Round-1/Round-2 cross-reference

I avoided reading the prior reviews until I'd written the bulk of
this. After writing, I scanned `01-codex-design.md`,
`02-claude-design.md`, `03-integration.md`, and
`03-codex-design-r2.md`. Convergences with codex R2:

| My item | Codex R2 |
|---------|----------|
| §A1 memory vs dynstate | "high: transform memory vs dynstate is confused" |
| §A2 device lease | "high: a per-device actor is not an actual device lock" |
| §A3 wiring topology | "high: proposed transform attachment point cannot see the required data" |
| §A4 update_dynstate re-entrancy | "high: observer Transform has a re-entrancy risk through update_dynstate" |
| §A5 cancel + enabled | "medium: cancel semantics are contradictory" / "medium: enabled=false semantics are unresolved" |
| §A6 NoopFileTransfer + CopyImage.pre_check | "medium: Phase 4 NETCONF-only filename semantics are not coherent enough" |
| §B/§4 trigger scoping | "medium: confirm-all-generation needs request scoping" |
| §B/§4 last-create-result | "low: action return value replacement still weaker" |
| §B/§4 run-log retention | "low: run-log retention changes fidelity" |
| §B/§9.4 (c) is harder than admitted | not in codex R2 — additional |
| §C1 yang.act is pre-v2 | not in codex R2 — additional |
| §C2 unfinished prose, §14 stub | not in codex R2 — additional |
| §D consolidated fidelity-vs-operability list | not in codex R2 — additional |
| §E `internal-state` operability regression | not in codex R2 — additional |

The convergence on the four high-priority items is, I think, the
strongest signal: independent reads of the design and the platform
code surface the same gaps. They aren't subjective. They need to be
addressed before runner code goes deep.

---

## TL;DR for skim-readers

- Don't start runner code yet.
- Six blockers: §A1–§A6.
- Two of them (memory/dynstate confusion; "DeviceRunner is not a
  real lease") are correctness-shaped and require choices that
  reach into the platform.
- One (wiring topology) is the most consequential clarification:
  per-device transform vs single top-level observer vs sysspec-actor.
- Fix `02-sw-install-design.md`'s unfinished prose remnants
  (§4.6 mid-sentence, §14 stub).
- Add a "conscious deviations from Python spec" subsection so a
  fresh reader can see the operability tradeoffs the v2 design has
  made (internal-state invisibility, bounded run-log, cancel
  latency, lease scope, generation backup-restore semantics).
- Update `00-orientation.md` with one short subsection on
  `update_oper`/`update_dynstate`/the `BBInterfaceTransform` shape.
  This is the single highest-leverage onboarding edit.
- `01-software-install-logic.md` is excellent; treat as the porting
  contract.
- Move §9.7 (TextFSMPlus / CLI driver / acton-utils textfsm extension)
  out of the Phase 4 design doc into a separate ADR. It's
  architecturally relevant only via the `DeviceOps` boundary; the
  rest is forward-looking.
