# Round-3 review — claude (pty-1)

Reviewing: `00-orientation.md`, `01-software-install-logic.md`, `02-sw-install-design.md` v3 (rev 2026-05-07), `docs/adr/cli-driver.md`, `stratoweave/sw-install/src/sw_install/yang.act` v3.

Method: read the doc set as a fresh reader; sanity-checked the design's platform claims against `stratoweave/stratoweave/src/stratoweave/ttt.act`, `device.act`, `adapters/adapter.act`, and the sorespo per-list-entry pattern in `stratoweave/sorespo/src/sorespo/rfs.act` + `layers/t_2.act` + `layers/base_2.act`. Round-1+2 reviews not consulted before forming this assessment; comparison notes appended at the end.

---

## TL;DR

v3 is a clear step up. The state-ownership consolidation (§3) is correct and removes a real round-2 hazard. The control surface (§4) is internally coherent and the conscious-deviations consolidation (§15.5) is genuinely useful. The YANG (`yang.act`) is in good shape and matches the doc.

But three claims in §3/§7/§8 are out of step with what the platform actually offers today, and one claim in §9.5 is *more* favorable than the doc presents (the platform already has what's being asked for). Together they push back on the "Phase 4 ready as-is" framing — there's still substrate alignment work that should land in v4 or be acknowledged as platform-prereq before coding begins.

**Status: do not start Phase 4 yet.** Address H1 + H2 first; H3/H4 are corrections with little blast radius.

---

## HIGH severity

### H1. §7.2 wiring is incompatible with the platform's RFS-shape `init_dynstate` (the `params.lower` you call doesn't exist on RFS-shape transforms)

The §7.2 sketch:

```acton
proc def act(params: ttt.TransformActorParams) -> ?proc(...):
    runner = DeviceRunner(...)
    if params.lower is not None:
        params.lower.declare_subscriptions(...)
```

…assumes `params.lower` is populated for the per-device transform. It isn't, for an RFS-shape (`RFSFunction`) transform.

Evidence:

- `stratoweave/stratoweave/src/stratoweave/ttt.act:1898` — `TransformActorParams.__init__(self, path, update_dynstate, update_oper, dev: ?DeviceMgr, lower: ?Layer=None)`. `lower` defaults to `None`.
- `ttt.act:2184–2185` — `RFSFunction.init_dynstate(...)` constructs `TransformActorParams(path, update_dynstate, update_oper, dev)` — **four args, no `lower`**, so the default `None` wins.
- Cross-check on the live consumer: `stratoweave/sorespo/src/sorespo/layers/t_2.act:21–27` — sorespo's `BBInterfaceTransformCtor` reads `params.dev` and never touches `params.lower`. That's not a coincidence; `params.lower` is genuinely None on RFS-shape.
- `TransformFunction.init_dynstate` (`ttt.act:2011–2012`) **does** pass `lower` — but it doesn't pass `dev`. So you can have one or the other from the existing platform; not both.

The §7.4 escape hatch ("if implementation surfaces a substrate-level incompatibility … we revisit") is meant to cover surprises. This isn't a surprise — it's visible in the code today.

Two cleaner fixes the design could pick:

1. **Drop RFS-shape; use plain `Transform`**. sw-install produces no downward config, so `RFSFunction`'s "receive `dev`, produce per-device config" semantics buy nothing. A regular `TransformFunction` gives you `params.lower` for free; the runner extracts `devname` from `params.path` (same trick `_DeviceTransaction.devname_from_device_path` uses, ttt.act:2206) and calls `dev_registry.get(devname)` itself. Zero platform change, and it lines up better with the observer-transform pattern the design otherwise embraces.

2. **Patch the platform** to thread `self.lower` through `RFSFunction.init_dynstate` (and bump `TransformActorParams` to also accept `lower` on RFS calls). Tiny patch but adds a sw-install-driven platform change. Not zero risk to other RFS consumers (though `params.lower` being None today means no caller could be reading it — so additive only).

Recommendation: option 1. Update §7 to: "sw-install uses `ttt.Transform` (not `RFSTransform`) under a per-device `ttt.List(...)`. The runner pulls its `DeviceMgr` from `dev_registry` by devname-extracted-from-path, and uses `params.lower.declare_subscriptions(...)` for global config." This also dispenses with the §7's "RFS-shape per-list-entry" framing borrowed from sorespo, which was the wrong analogy to begin with — sw-install isn't producing per-device downward config.

### H2. §3 / §8.4 restart story: there is no platform path for the runner actor to receive restored dynstate at startup

The design says (§8.2):

```acton
actor DeviceRunner(...):
    var dynstate: SwInstallDynstate = ...      # restored from the platform on startup
```

…and (§8.4) "On platform startup … runner is constructed with restored dynstate injected."

That's not how the platform delivers dynstate today:

- `_TransformTransaction.restore` (`ttt.act:1964–1975`) sets `self.dynstate` on the **transaction**, not the runner actor.
- `_TransformTransaction.init_dynstate` (`ttt.act:1977–1979`) calls `self.function.init_dynstate(act, update_dynstate, update_oper, path, lower)` — note the **outbound** `update_dynstate` proc (the runner uses it to *write*), but no inbound `dynstate` parameter.
- `TransformFunction.init_dynstate` (`ttt.act:2011–2012`) builds `TransformActorParams(path, update_dynstate, update_oper, None, lower)` — again no `dynstate`.
- The runner's only inbound handle is `_on_conf(conf, memory)` (`ttt.act:2014–2017`). **`memory` is delivered, `dynstate` isn't.** That asymmetry is a real platform-API gap.

Cross-check against sorespo's RFS actor: `stratoweave/sorespo/src/sorespo/rfs.act:189–251` — `BBInterfaceTransform` actor initialises `_dynstate = base.BBInterface.dynstate_type()()` (a fresh empty value) on every spawn. Across restarts, sorespo's runner does **not** see the previously-persisted dynstate inside its actor body. The persisted dynstate is read by the *sync* `BBInterface.transform(self, i, di, dynstate)` (rfs.act:254 onward) — `transform_wrapper` sees it for downward computation. The actor's in-memory copy is rebuilt from observed state.

That works for sorespo because its actor-side dynstate (`pps`, `rx_packets`) is just a derived counter — losing it is a non-event. For sw-install, dynstate **is the source of truth** for every restart-critical field: `next_request_id`, `last_pack_snapshot`, `last_request_generation` (and friends), `current.status`, `current.plan`, `current.error_count`, `next_wake_at`. If the runner starts fresh on each restart:

- `next_request_id` resets to 1 → ID collisions with previously-published `request[]` entries.
- `last_*_generation` resets to 0 → first config observation re-fires every trigger spuriously, since `cfg.start_generation > 0` is true the moment a non-default value exists. (The same generation counter that round-2 was supposed to make safe.)
- `current.status` is lost — the §8.4 "processing → failed-transient" recovery rule has nothing to act on.

This is the same shape of issue as the round-2 finding "memory has no platform-side write path back to the runner," but for dynstate, and it is unaddressed in v3. §8.4's narrative does not survive contact with the platform's `init_dynstate` signature.

Possible resolutions, in increasing invasiveness:

1. **Bridge through `transform_wrapper`.** The function is a class instance shared with the runner actor (the `_on_conf` is set inside `init_dynstate`). `transform_wrapper(cfg, linked, memory, dynstate)` could stash `dynstate` on `self` and the runner reads it on first `on_conf` (or it's pushed via a one-shot `proc` call to the runner). Workable but ugly: makes `transform_wrapper` no-longer-pure, couples timing of "runner sees dynstate" to "first compute fires," and you have to be careful about ordering between the first `update_dynstate` write from the runner and the platform's restored value.

2. **Add `dynstate` to `TransformActorParams`** and have `_TransformTransaction.init_dynstate` pass `self.dynstate` through. One-line platform change in two places (one for Transform, one for RFSTransform), additive (existing callers ignore the new field). Probably the right answer.

3. **Add an `on_dynstate(dynstate)` action callback** to the runner-construction protocol, called once after restore. Cleaner separation but bigger surface change.

Either way, **the design currently bakes in an interface that doesn't exist.** A v4 needs to either pick (1) explicitly (with eyes open about the wart) or to lift "platform passes dynstate at construction" out of "implementation detail" into a named platform prerequisite alongside §14.

Note: this is independent of H1. Even if you switch from RFS-shape to plain Transform (recommended for H1), H2 still applies — `TransformFunction.init_dynstate` doesn't pass dynstate either.

### H3. §9.5 / Q2 is moot — `DeviceMgr.get_dmc()` already exists

The design flags this as `⚠️ASSUMPTION: the platform team will accept the DeviceMgr.get_dmc() addition. **Round-3 question.**` and lists it as `❓Q2`.

It's already in the code: `stratoweave/stratoweave/src/stratoweave/device.act:401`:

```acton
def get_dmc():
    return dmc
```

`set_dmc(...)` is at line 427 and `var dmc` at line 288. The mutability you're concerned about ("DMC is mutable") is real and `set_dmc` is called repeatedly on reconfiguration, but the read API exists.

Action: drop §9.5's ASSUMPTION marker, drop Q2 from §12, drop item 3 from §14 (the "or `DeviceAdapter.get_credentials()` instead" alternative). The "read DMC at use time" guidance still stands — just frame it as exploiting an existing API rather than as a platform ask.

One sub-question that *would* be substantive but isn't raised: `get_dmc()` is defined inside the actor body, so externally it's an actor-method call (mailbox-bound, not free of cost). The factory's "calls `dev.get_dmc()` per transfer to get fresh credentials" idiom assumes a callback-style receive — worth confirming the call shape compiles cleanly. Probably fine but worth a one-line check before Phase 5 codes against it.

### H4. YANG augments `/sw-rfs:rfs`, but the design narrative says `/devices/device[name]/.../software-pack/`. Pick one.

§7.1's wiring sketch:

```
Host layer:
    /devices/device[name]/
        +-- ... config from RFS layer above ...
        +-- /software-pack/                       ◀── per-device transform attaches here
```

§4.8's table:

> 🆕 `/devices/device/scp-port` | **direct device augmentation** (CL_R2_5) — not under `software-pack/` — survives unbinding the pack

§9.6 says the same thing.

The actual YANG (`yang.act:191`):

```yang
augment "/sw-rfs:rfs" {
    leaf scp-port { ... }
    container software-pack { ... }
}
```

`sw-rfs:rfs` is the RFS-layer per-device list (sorespo `t_2.act:18` confirms: `ttt.List(ttt.Container({...}), [...])` keyed by `name`, namespace `http://stratoweave.org/yang/stratoweave-rfs.yang`). It is *not* `/devices/device[name]/...`, which is the per-device meta-config list under `stratoweave-device`.

Two effects:

1. The design doc and the YANG disagree on the path. A reader who reads §7/§9 first then opens `yang.act` will be confused.
2. The "scp-port survives pack unbinding" rationale (§9.6, CL_R2_5) is half-true. The leaf does survive pack unbinding (good — it's a sibling of `software-pack`, not nested). But it does not survive the device being removed from the RFS layer, which a reader of §9.6's narrative might believe it does. Calling it "direct device augmentation" overstates its scope: it's a direct *RFS-list* augmentation.

Pick:

- **(a)** Update the design doc to say "augments `/sw-rfs:rfs[name]/`," fix §7.1's wiring sketch, and reword §9.6 to be precise about the layer. No YANG changes. Recommended — `sw-rfs:rfs` is where the per-device RFS-layer config lives, which is where sw-install belongs.
- **(b)** Move the augment to `/swdev:devices/swdev:device` so the design's narrative becomes accurate. More work; touches the device-meta-config namespace; probably wrong.

(Round-2's "scp-port placement" CL_R2_5 was about whether scp-port nests inside `software-pack/` or sits outside. The right answer to that is "sits outside" — the YANG has it right. The bug is in the design doc's *path narrative*, not its scoping decision.)

---

## MEDIUM severity

### M1. §3.5 generation-counter restore semantics covers only the safe direction

§3.5 addresses the case `restored_config_generation < persisted_dynstate_generation` ("trigger evaluates false until manually bumped"). That's the *safe* direction — false-non-trigger.

The dangerous direction is the opposite: dynstate restored from an older snapshot than config. Then `cfg.start_generation > dynstate.last_start_generation` evaluates *true* on the next reconciliation pass and the runner fires a stale start trigger that the operator never asked for, possibly starting an old request that's already terminal-in-config. Worse, `next_request_id` is older than the highest id visible in the published `request[]` history (because §3.3's history is in dynstate too) → a fresh request ID collides with one already in oper.

§15.5 #5 calls this out as "Generation counters can go backward on backup-restore unless dynstate is restored alongside it" — but only the false-non-trigger direction. The false-trigger direction is more dangerous and isn't called out anywhere.

Recommendation: spell both directions out in §3.5 and §15.5 #5. Add a runner-side defensive check at startup: "if any persisted oper-projection request id ≥ `dynstate.next_request_id`, log a 'restore inconsistency' error and refuse to materialise new requests until operator clears via `request-generation` *and* manual `next_request_id` reset" — or whatever you decide the recovery is. The point is: dynstate-and-config restore inconsistency needs an explicit fail-loud path, not an implicit "operator should restore them together."

### M2. §6.2 / CL_R2_8 callback-mailbox contract has no enforcement mechanism described

§6.2 says: "Every cb passed to a step must dispatch on the per-device DeviceRunner mailbox, not on the step's own actor (if any)."

That's the right contract, but the design doesn't say *how* the runner achieves it. The natural Acton idiom is: the runner closes over an `action def` defined on itself and passes that as `cb`. As long as steps are *classes* (not actors), every `cb` they receive lands on the runner — the contract holds for free.

But §6.1 declares the step protocol as `class Step(object)` — with no actor — so steps can't accidentally violate it. So why is it called out as a contract that must be enforced? Either:

- It's a reminder for step authors to *not* spawn helper actors and dispatch through them — fine, but say that explicitly ("steps must be ordinary classes, not actors; if a step needs a helper actor it must terminate that actor before invoking `cb`").
- Or steps can be actors after all, and the runner actively rebinds `cb` somehow — in which case spell out the rebinding.

The way it reads now sounds like a runtime invariant the runner enforces, but the runner has no mechanism to do so other than "trust the step author."

### M3. §3.3 history retention rule has an internal inconsistency with §3.2

§3.2:
> `current: ?RequestState  # at most one active request`

§3.3:
> `history` retains:
> - the latest non-terminal request unconditionally (drives idempotency comparisons against pack-data); …

If `current` is the only "active" slot, the *non-terminal* request is `current`, not in `history`. So §3.3's first bullet either means:

- (a) "the latest non-terminal request" in `history` (i.e., a previous request that was non-terminal at the time it was superseded — but §3.1 marks superseded requests obsolete, so they become terminal at supersession, so this set is empty), or
- (b) the wording is meant to say `current` lives somewhere, and history's first bullet is actually "the entry holding the most recent `last_pack_snapshot`" (which is what §3.3 says in its very next paragraph).

I think you mean (b) — the rule is "never drop the entry that supplies the idempotency baseline." §3.3's first bullet then is redundant with the third paragraph ("never drop the entry holding the most recent `last_pack_snapshot`"). Recommend collapsing to a single statement.

A related concern: idempotency in §4.1 is defined against `dynstate.last_pack_snapshot`, *not* against an entry in `history`. If you're already storing the snapshot as a top-level `last_pack_snapshot`, the "history retains the entry holding the most recent snapshot" rule is doubly-stored — pick one source of truth.

### M4. §6.6 run-log filter — Acton's `logging` doesn't have ContextVar/Filter equivalents

`acton/base/src/logging.act` (`Logger` at line 252, `info(msg, data: ?dict[str, ?value])` at line 303) takes structured data per-call. There's no thread-local context var, no `logging.Filter` chain, no `_log_filter` analog of the Python's `OperCdbLoggingHandler._log_filter`. The `swi_request_path / swi_component / swi_step / swi_run_id` attributes the design refers to have to be passed *explicitly in the structured-data dict on every emit*.

Two consequences for §6.6:

1. **Step authors will forget.** Without ContextVar-style propagation, every `log.info(msg, {...})` call in step code must remember to add the swi_* keys, or the run-log filter drops it. Plumbing a "`self.log` per-step pre-bound to swi_*" onto the step API would help, but isn't in §6.1's signatures.
2. **Filter at handler level, not at logger level.** §6.6 says "Records flowing past from other modules are dropped at the persistence boundary." That implies a custom `Handler` that inspects structured-data keys. Practical, but worth naming the Handler class and where it lives in §6.6 — otherwise reviewers can't tell whether the filter is real or aspirational.

Recommend §6.1 or §6.6 specify: (a) a per-step `StepLogger` injected into step methods that closes over swi_* and forwards to the runner's logger — so step authors can't forget — and (b) a `RunLogHandler(parent, request_state_lookup)` that the runner installs on the per-device logging chain to capture `swi_*`-tagged records and write them into the bounded ring.

### M5. §4.6 per-request scoping silently no-ops missing target ids

> "errors silently if no such id exists in current+history"

Silent error is dangerous in a control surface where the operator's only feedback is the published oper data. If I `cancel-target-id 7 + cancel-generation+1` and there's no request id 7, my cancel just vanishes — and the request I *thought* was id 7 (via stale RESTCONF state) keeps running.

Cheaper than full per-call correlation: surface a `last-trigger-result {generation, target-id, kind: accepted|rejected, reason}` oper leaf alongside `last-create-result`. Preserves the single-slot last-writer-wins model but at least gives operators a chance to detect their trigger didn't take. Or — and probably easier — log a WARNING entry into the run-log of the targeted-id-if-found, and into a per-device "rejected triggers" log otherwise. (The latter requires another bounded ring buffer, so probably not worth it; the former oper leaf is cheaper.)

### M6. §4.7 enabled `true → false` with `failed-transient + next_wake_at` says backoff doesn't fire while disabled — but `after backoff: _start_run()` is already scheduled

§8.6 schedules `after backoff: _start_run()` at the end of a failed run. §4.7 says when `enabled` flips false during this window, "next_wake_at retained; backoff doesn't fire while disabled."

Mechanism to achieve this isn't stated. The `after` block fires regardless of `enabled` — the runner has to re-check `enabled` at start of `_start_run` and *not start*, instead transitioning to `paused` and persisting `next_wake_at` with the original value. Then `false → true` schedules a fresh `after max(0, next_wake_at - now): _start_run()`. (§8.6's last line addresses this for restart, not for in-flight pause.)

Recommend §8.6 spell out: `_start_run()` is gated on `current_global_config.enabled`. If disabled, transition to `paused` (preserving `next_wake_at`); on `enabled` flip back to true, replay the schedule.

### M7. §3.4 diagnostic projection leaves are always present in the YANG, regardless of OS

`yang.act:481-504` — `destination-volume`, `boot-time`, `rebooted`, `op-id-add/prepare/activate/commit` are all on `request/component/` without `when`. They'll show up as absent leaves on, say, Junos requests — semantically harmless but YANG-tooling-noisy. Consider:

```yang
leaf op-id-add {
    when "../../software-pack-data/os = 'ios-xr'";
    type uint32;
}
```

Same for the SROS-only `rebooted`. Per-Junos-RE projections (§3.4 last bullet) aren't in `yang.act` at all yet — flag for Phase 6, or scope into v3.

### M8. §15.5 #1 phrasing overstates the loss

> "internal-state is no longer NETCONF/RESTCONF-visible. Replaced by typed diagnostic projections (§3.4) — better for typed inspection, worse for 'show me everything via one CLI command'."

The diagnostic projections (§3.4) live on the request/component subtree and *are* NETCONF/RESTCONF-visible — that's the whole point. What's no longer visible is the *opaque jsonpickled blob*. Reword to: "`internal-state` (opaque JSON blob) is dropped; the operationally-useful fields are now typed leaves under `request/component/` (still RESTCONF-visible). Fields not surfaced as named leaves are no longer externally inspectable." That's the honest framing.

### M9. §8.6 `error_count.backoff` reset-on-DONE is ambiguous

§6.5 says "DONE; clear error_count" — clearing should include `error_count.backoff`. §8.6 references `error_count.backoff` as the live mutable backoff state. But if a request transitions DONE-then-eventual-cancel-then-fresh-create-request, the new request starts with what backoff?

The Python original gives each new request a fresh `error_count` (§3.1's "create request with id = last_id+1" implies a fresh oper subtree). The Acton design's `RequestState` is per-request, so its `error_count` is naturally per-request — DONE clearing is fine for that request, and a fresh request gets fresh counters. Just confirm in §3.2 that `error_count` is on `RequestState`, not on the dynstate root. Reading §3.2 again — it's on `RequestState`. Good. But the "(error_count.backoff or 10)" formula in §8.6 then needs to be clear that the "or 10" is only on the *first* failure of a new request, not after a successful run. Looks right; just worth a sentence.

---

## LOW severity / nits

- **L1.** §6.1: "Step methods receive **four** abstractions" but the signature shows five params (`state, ops, lfi, rfi, ft`). Fix the count.
- **L2.** §6.1: `cb: action(StepResult, NewState, ?Exception) -> None`. The runner needs to disambiguate `(FAILURE, _, exc=None)` from `(FAILURE, _, exc=Some)`. Fine, but state the convention: "`exc` is non-None iff the step body raised; runner logs traceback and treats as FAILURE."
- **L3.** §6.1 type `NewState` isn't formalized — should be `?State` (or a per-OS subclass). Minor; trivially fixed.
- **L4.** §8.7 "a one-shot subscription on the device's status (or polling)" — the "subscription on the device's status" path isn't a thing on `DeviceMgr` today (no equivalent of `on_dmc_change` callback). Polling is the only viable mechanism. Either reduce §8.7's wording to just "polling" or add to §14 a v2.0 platform ask for "DeviceMgr `on_status_change` event."
- **L5.** §7.4 ⚠️ASSUMPTION about subscription delivery — given H1 + the rewiring to plain `Transform`, the subscription callback lands on the runner actor (since `cb` is an `action def` on the runner). No re-entry through `transform_wrapper`. So the assumption resolves favorably once the wiring is fixed.
- **L6.** YANG `leaf scp-port { type inet:port-number; }` (yang.act:192) has no default. Python original used "fall back to SSH port" semantics. Either add `default 22;` (wrong if SSH port is non-22) or document "absent = use SSH port" in the YANG description (yang.act has this — fine; ignore L6).
- **L7.** YANG `revision 2026-05-07` — the date matches today's (`currentDate`). If the v3 YANG was authored earlier, please use the actual authorship date; if it's intentional that v3 is dated today, fine.
- **L8.** §11 phasing: item 7 ("Mock-driven full-flow integration test") is the right gate but item 8 ("kill mid-step, verify re-spawn from dynstate") isn't possible until H2 is resolved. Make it explicit that test 8 depends on the dynstate-runtime delivery story.
- **L9.** §13 "Acton stdlib filesystem primitives used by `LocalFileInspector`" — the `file.FileCap` reference is correct for capability-style fs access, but the design doesn't mention how the cap is acquired. The factory needs `file.FileCap` injected from the app (since caps aren't ambient in Acton). Minor design gap.
- **L10.** §6.6 "(when, seq) where `when` is `yang:date-and-time` and `seq` is a per-request monotonic u64" — yang.act:530 keys `(when, seq)`. Good. But "seq resets to 0 on `clear-run-log-generation` increment" + "ring buffer drops oldest when full" interact: if the ring is full, the next entry has `seq` = (last_seq + 1). After clear, `seq` resets to 0. Two clears within one run: nothing prevents the `(when, seq)` pair from colliding with a previously-existing key if `when` happens to be the same after clear. Microsecond `when` is unlikely to collide post-clear but could. Either reset `when`-baseline as well, or clarify the post-clear ring is empty so collision-against-existing isn't possible (which seems to be your intent — yes, clear empties the ring per §4.5). Just confirm in §6.6.
- **L11.** ADR `cli-driver.md` is well-scoped — fine to defer all the listed Phase 5 issues. The acton-utils textfsm extension as "independent workstream" with sw-install as first consumer (§"Dependency") is the right call. No notes.

---

## Onboarding doc evaluation

### `00-orientation.md`

Stands on its own for a fresh reader. Strengths:

- The "Where things live" tree front-and-center sets concrete spatial grounding before any concept-talk.
- "The four big abstractions" table is the highest-leverage paragraph in the whole doc — `Node`/`Layer`/`Transform`/`DeviceAdapter` is the right minimum viable mental model.
- The data-flow diagram (lines 53–82) is doing real work — most stratoweave docs assume it.
- "Operational state and the observer-transform pattern" (lines 113–125) is exactly the right lead-in to sw-install's shape; the BBInterfaceTransform reference is a useful pointer.
- §3 (mapping table) is the highest-leverage paragraph for someone joining the project mid-stream.

Minor improvements:

- **`Layer.declare_subscriptions`** isn't mentioned, despite being load-bearing in §7.2 of the design. Add one bullet under "Operational state and the observer-transform pattern" — "transforms can also subscribe to data outside their local input via `params.lower.declare_subscriptions(...)`; this is how, for example, sw-install's per-device runner reads the global `/software-install/` config." (Adjust per H1.)
- **`gen_adata` runs at build time** — say so explicitly in "YANG-as-types," since the design doc's `gen_adata` references are the second time a reader meets the term.
- The "tricky bits we'll need to decide on in Phase 3" in §3 (lines 230–238) are a closeout from when this doc was written. Worth a short note that the bullets are now resolved in the design doc — point to §3, §4, §7, §9 of `02-sw-install-design.md`.

### `01-software-install-logic.md`

Excellent. Two engineers reading this would implement compatible reimplementations. Sectioning is right; depth is right; the "stop after §2/§3 for high-level" hint at the top is a nice touch.

Minor:

- §3.7 (`confirm-step` action): "When a step's plan loop reads `step.status == 'waiting-confirmation'`, it sets `needs_confirmation = not confirmed.exists()`." This is effectively part of §3.4's component step loop, but §3.4 doesn't mention confirmation-status checking explicitly. Thread the link.
- §10 (idempotency / restart story) is great; consider front-loading the "five things a port must preserve" list higher (§10's current placement near the end is fine for a spec, but a port author will hit §3 first and miss the cross-cutting picture).
- §12 (decision points for Phase 3) — same concern as 00-orientation §3: the bullets are resolved in the design doc; mark them "resolved in 02-sw-install-design v3 §X" so a fresh reader doesn't waste time re-deciding.

These are stale-pointer issues, not content issues. The doc's substantive content is solid.

---

## Specific design areas the team flagged

**§3 (state ownership; dynstate-only with diagnostic projections; transform memory dropped)**: the consolidation is the right call; the v2 split was unworkable for the stated reason. Diagnostic projections are the right tradeoff. **But** H2 is unresolved — the runner has no platform-side path to receive its own restored dynstate. Until that's addressed, "dynstate-only" works for *writes* but not for *reads at startup*.

**§4 (control surface generation counters with target-id scoping; cancelling enum; enabled state machine)**: internally coherent. Per-request scoping is well-modelled. M5 (silent no-op on missing target id) and M1 (one-direction restore semantics) are real but addressable. The `enabled` state-transition table (§4.7) is the kind of artifact a design doc should contain — keep it.

**§7 (Option B wiring — per-device transform + global subscription via `Layer.declare_subscriptions`)**: H1 — wiring is broken under RFS-shape; switch to plain Transform. H4 — the YANG-vs-narrative path mismatch needs reconciling. Once those are fixed, §7 is the right structural answer (no top-level coordinator, per-device fan-out, declare_subscriptions for global config). Don't park the §7.4 fallback as "fallback" — once H1+H4 are fixed, Option B should be the unambiguous answer; document any platform changes explicitly in §14 instead of in a fallback narrative.

**§8 (DeviceRunner re-entrancy invariants, restart story, cancel mechanics, honest device-lease downgrade)**:

- Re-entrancy invariants (§8.3) — A4 invariants 1, 2, 3 are stated cleanly. Good.
- Restart story (§8.4) — depends on H2; reframe once that's resolved.
- Cancel mechanics (§4.4 + §8.5) — `cancelling → cancelled` via generation_token + drain is the right pattern.
- Honest device-lease downgrade (§8.1) — the README/§15.5/runtime-warning trio is the right way to handle a known-incomplete safety contract. The v2.0 lease API entry in §14 captures the followup. This is well-handled.

**§9 (three-way file abstraction split + DeviceMgr.get_dmc())**: the LocalFileInspector / RemoteFileInspector / FileTransfer split is exactly right; resolves the v2 NoopFileTransfer vs CopyImage incoherence cleanly. RemoteFileInspector being separate from FileTransfer is what lets Phase 4 verify pre-staged images without committing to a byte-mover. H3 — `get_dmc()` is already there; drop the ASSUMPTION marker. The "read DMC at use time" pattern is correct.

**§15.5 (consolidated tradeoff list)**: covers most of what's needed. Gaps (suggest adding):

- The H2 dynstate-runtime-delivery story (whatever you decide) — needs a deviation-list entry once it lands.
- M1's bidirectional generation/dynstate restore semantics — both directions in #5, not just one.
- M8 wording on internal-state visibility (the diagnostic projections *are* RESTCONF-visible).
- Optionally, "the per-device install lease is `software-pack`-augment-scoped, not `/devices/device`-scoped" — H4's narrative correction.

**ADR `cli-driver.md`**: commits only to the `DeviceOps` boundary in Phase 4, defers all CLI specifics to Phase 5. The seven Phase-5 issues listed at the end are the right things to flag and the right place to flag them. Good lift; no notes.

**v3 yang.act**: matches the doc with the H4 caveat (the augment target is `/sw-rfs:rfs`, not `/devices/device`). The control subtree shape (lines 217–337) is well-modelled — generation+target pairs, `request-options/confirm-steps` per-request override, presence-based confirmation list. `last-create-result` config-false at line 339–360 is right. `request[]` config-false (line 363) and `(when, seq)` run-log key (line 530) are correct. Diagnostic projections at lines 481–504 — see M7 (consider `when` constraints). Good shape overall.

---

## Approval / blocking status

**Blocking:**
- **H1** (Option B wiring incompatible with platform's RFS-shape `init_dynstate`) — design must specify either RFS→Transform reshape or platform patch.
- **H2** (no platform path for runner to receive restored dynstate) — design must specify the bridge mechanism (transform_wrapper stash, TransformActorParams.dynstate, on_dynstate callback, or other) before Phase 4 begins.

**Trivially fixable, please address in v4:**
- **H3** (`get_dmc()` exists; drop the ASSUMPTION).
- **H4** (YANG vs. design-doc path narrative; pick one).

**Worth addressing but non-blocking:**
- M1, M2, M3, M4, M5, M6, M7, M8, M9 — each is a localized cleanup; none hold up Phase 4 once H1+H2 are resolved.

**Onboarding docs:** good as-is for a fresh reader, with the small stale-pointer cleanups noted.

Once H1 and H2 are addressed (and H3/H4 corrected), I would not block Phase 4 implementation. The architecture is sound and the v3 shape is converging.

---

## Comparison with rounds 1–2 (scanned after forming the above)

Reading `04-claude-design-r2.md` and `05-integration-r2.md` after the fact:

- **Convergent with my round-2 self**: state ownership consolidation (A1/CL_R2_2), per-device wiring (A3, modulo H1), NoopFileTransfer/RemoteFileInspector split (A6/CL_R2_4), conscious-deviations consolidation (§D → §15.5). All landed.
- **My round-2 self also flagged**: "memory has no path back from runner to platform" — addressed via dynstate-only. Good. **But the dynstate analog (H2 here) wasn't called out in round-2** — that's a new finding round-3 caught by going one layer deeper into `init_dynstate`'s actual signature. Worth noting that "memory→dynstate" as a fix solved the *write* path but not the *restore-read* path.
- **Round-2 H1 analog**: codex-r2 §7 raised concerns about declare_subscriptions, which v3 §7.4 acknowledged with ⚠️ASSUMPTION. v3 attempted resolution but hit the params.lower / params.dev mutually-exclusive problem documented above. So v3 made progress on round-2 §7 without fully landing it.
- **`get_dmc()` already exists** — neither round-1 nor round-2 caught this. v3 carried the "platform ask" framing forward unchanged. Easy round-3 win.

Net: H1 and H2 are the round-3 deltas. The other items are either round-2 cleanup that didn't make v3, or fresh nits.
