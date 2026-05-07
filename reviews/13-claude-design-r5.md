# Round-5 Claude review — `02-sw-install-design.md` v5

Reviewer: pty-1 / Claude. Re-briefed against the full doc set
(`00-orientation.md`, `01-software-install-logic.md`,
`02-sw-install-design.md` v5, `docs/adr/cli-driver.md`,
`stratoweave/sw-install/src/sw_install/yang.act` v5) plus a fresh read of
the live platform code (`ttt.act`, `device.act`, `adapters/adapter.act`,
`sorespo/`). Prior round-1..4 review threads were not consulted before
forming the assessment below.

---

## TL;DR

v5 is materially better than v4 — the three most painful round-4 findings
(D3a-broken construction-time restore, unimplementable Tier-A "await
ack", invisible sibling `scp-port`) are all closed in a way the live
platform code actually supports. I traced the §3.6 stash-path against
`_TransformTransaction` and the `Layer.load_from_db → recompute(force=True)`
boot sequence in `app.act:127-154` and the path holds.

But three issues are big enough to call out before Phase 4 starts coding,
and a handful of smaller ones want clean-up:

1. **HIGH — Cross-actor read of `transform_fn.stashed_dynstate`
   is the only mechanism in the design where actor isolation is broken.**
   Section §3.6 has the runner reach into the function instance owned by
   `_TransformTransaction` to read a mutable field. Everywhere else the
   design carefully uses `update_dynstate` / `update_oper` / `act` callback
   wiring — actor message-passing. The stash field bypasses that. Either
   convert it to a proc on the function object that the runner *calls*
   (and which then forwards via an action on the runner), or have the
   transform push the dynstate at the runner via an action ref captured at
   `init_dynstate` time. Discussed below.

2. **HIGH — The §3.7 "publish-before-side-effect + idempotent-on-re-fire"
   wording is *almost* right but in two cases doesn't actually withstand
   a crash window.** Specifically `auto_started_after_confirm` (§4.2 v5
   fix per CR4_5) and any field where Tier-A persisting the
   "trigger-was-consumed" marker happens *before* the Tier-B
   persisting of the work it consumed. The model only works if the
   trigger-marker and its consequences are persisted in the *same*
   `update_dynstate(...)` snapshot. Detail and concrete failure window
   below.

3. **MEDIUM — The §7.2 5-second `runner-status=missing-global-config`
   guard interacts badly with the `period=...` parameter on the
   `SubscriptionSpec` and the boot lifecycle.** The first
   `_sub_tick` fires synchronously from `declare_subscriptions` —
   *before* `load_from_db` has populated the lower layer — so the cb
   delivers `None` once at boot regardless of topology. Subsequent ticks
   only fire after `period`. If `period > 5s`, every well-wired startup
   transiently publishes `missing-global-config` until the second tick
   lands. The guard is not actually wrong but the doc/design needs to
   either pin `period ≤ 5s` or define the window relative to "first
   non-`None` cb" not "first `on_local_config` fires + 5s".

Plus six smaller items (medium / low) collected in §B.

---

## A. Verifications I did against the live code

I read these to confirm v5's claims rather than take them on trust.

### A1. `transform_wrapper` post-restore timing (Q1, §3.6) — **HOLDS**

Trace through `app.act:138-152` (the `StartupBootstrap._run`):

```
load_from_db()                       # cascades down via load_from_read_txn
  → for each transform: restore(ATTR_DYNSTATE, ...)   # ttt.act:1964-1975
       → self.dynstate = swdb.decode_node_bytes(raw)  # ttt.act:1973
recompute(force=True) on top layer
  → apply(tid, force=True)                            # ttt.act:876
    → root.configure(tid, {}, lower, force=True)      # cascades through Container
      → _Container.configure: empty diff + force → propagate {} to children with force=True   # ttt.act:1078-1080
        → eventually _TransformTransaction.configure(tid, {}, ..., force=True)  # ttt.act:1671
          → newconf = patch(self.running, accdiff={}) = self.running  # restored running
          → newconf is non-empty → compute(tid, ...)  # ttt.act:1942
            → self.function.transform_wrapper(merged, linked, self.memory, self.dynstate)
                                                       # self.dynstate is the restored value here
```

So at the time `transform_wrapper` is called from the forced post-restore
recompute, `self.dynstate` *is* the restored LMDB value — confirmed. Then
the `commit → finalize` path on `_TransformTransaction.finalize` (line
1989) calls `self.function.on_conf(self.get(), self.memory)` which is the
runner's `on_local_config`. So by the time `on_local_config` first fires,
the SwInstallTransform instance has had `transform_wrapper` invoked at
least once with the restored dynstate. The stash mechanism works.

**Caveat the design should call out:** if `self.running` happens to be
empty at startup (no `software-pack` has ever been authored for this
device, or it was deleted before the snapshot), the configure path goes
through the `if newconf == {}` branch at `ttt.act:1686-1693` — `compute`
is *not* called, and `finalize` skips `on_conf` because `self.running` is
falsy (line 1992). The runner stays parked in `STARTING` forever, never
reads its stash, and `update_dynstate` never gets called from it (so the
persisted blob from the previous run is also not overwritten — that part
is fine).

This is benign in steady state (no config = nothing to do) but
intersects the §7.2 runner-status guard: a freshly-installed app where
the operator hasn't authored *any* `/sw-rfs:rfs/.../software-pack`
container yet will park at `runner-status=starting` indefinitely (never
even reaching the 5s window because the runner never gets its first
local-config callback). The status leaf says `starting` forever. The
design should explicitly note this as a *fourth* failure mode the guard
doesn't cover, distinct from the three it does cover.

### A2. `_RFSTransaction.finalize` empty-output suppression (§7.1) — **HOLDS**

`ttt.act:2151-2158`:

```acton
def finalize(self, tid):
    if self.running:
        output = self.output
        if output is not None:
            if not (isinstance(output, gdata.Container) and not output.presence and len(output.children) == 0):
                self.function.on_conf(self.get(), self.memory)
```

Confirms — empty `gdata.Container()` output suppresses `on_conf`.

Compare to plain `_TransformTransaction.finalize` at `ttt.act:1989-1993`:

```acton
def finalize(self, tid):
    if self.running:
        self.function.on_conf(self.get(), self.memory)
```

No empty-output check. So the deliberate-departure note is correct: an
RFS-shaped transform that outputs `gdata.Container()` would silently
lose every `on_conf`.

### A3. `RFSFunction.init_dynstate` does not pass `lower` (§7.1) — **HOLDS**

`ttt.act:2184-2185`:

```acton
proc def init_dynstate(self, act, update_dynstate, update_oper, path: Path, dev: swdev.DeviceMgr):
    self._on_conf = act(TransformActorParams(path, update_dynstate, update_oper, dev))
```

vs the plain `TransformFunction.init_dynstate` at `ttt.act:2011-2012`:

```acton
proc def init_dynstate(self, act, update_dynstate, update_oper, path, lower: ?Layer=None):
    self._on_conf = act(TransformActorParams(path, update_dynstate, update_oper, None, lower))
```

Confirmed. `RFSTransform`-shaped transforms get `params.dev`; plain
`Transform`-shaped transforms get `params.lower`. Either-or, not both.
The departure note is correct that sw-install needs `params.lower` (for
the `/software-install/...` subscription) more than it needs
`params.dev` (which it can recover via `dev_registry.get(devname)`).

### A4. `devname_from_swi_path` shape (§7.1) — **MOSTLY CORRECT**

The wiring topology in §7.1 (`ttt.List(ttt.Container({"name", "software-pack": swi_factory}))`) does
mean `params.path` is a `PathElem` whose `.parent` is a `PathKey`, and
`PathKey.name` is the keystr (line 187: `self.name = keystr`). For a
single-key list keyed by `name`, the keystr equals the `name` value.
`p.name == "dev1"`. ✓

**Doc bug:** the comment in §7.1 says *"our `params.path` is a
`PathContainer`"*. There is no `PathContainer` class in `ttt.act`. The
relevant class is `PathElem`. Single-line fix.

### A5. `dev_registry.get(devname)` shape (Q2) — **SYNCHRONOUS-LIKE**

`device.act:77-81`:

```acton
def get(name: str) -> DeviceMgr:
    if name not in devices:
        devices[name] = DeviceMgr(...)
    return devices[name]
```

It's a regular `def` on the `DeviceRegistry` actor. Existing call sites
including `_RFSTransaction.__init__` (`ttt.act:2071`,
`self.dev = dev_registry.get(self.devname)`) and `testing.act:38` use
the result synchronously in expression context. So the design's pattern
in the §7.2 act-callback works as written. Q2 should be marked
*resolved-by-existing-precedent* in §12; not a verify-in-Phase-4 risk.

### A6. `dev.get_modules()` shape and `NoAdapter` detection (§8.7, CR4_2) — **HOLDS**

`device.act:760`: `def get_modules() -> (dict[str, ModCap], ?str): return modset, modset_id` — confirmed 2-tuple.
`adapters/adapter.act:273-277`: `NoAdapter.get_capabilities() -> []`,
`NoAdapter.get_modules() -> {}` — confirmed empty for both. So §8.7's
`if len(modules) > 0 and len(caps) > 0` cleanly distinguishes
NoAdapter from any real adapter. ✓

---

## B. Issues, by severity

### HIGH 1 — `transform_fn.stashed_dynstate` is a cross-actor mutable read

**§3.6, §7.3.** The runner does:

```acton
stashed = self.transform_fn.stashed_dynstate
```

inside `on_local_config`. `transform_fn` is the `SwInstallTransform`
instance — a class instance, not an actor. It's owned by
`_TransformTransaction` (which is the impl of an actor — `Transaction`).
The field is *mutated* by `_TransformTransaction`'s actor (during
`compute` → `transform_wrapper`). It is *read* from the `DeviceRunner`
actor.

Everywhere else in the design — `update_oper`, `update_dynstate`,
`declare_subscriptions(cb=...)`, `dev_registry.get`, the `act` callback —
the design uses Acton's actor message-passing. This single field is the
only crack.

I am not 100% sure of Acton's exact memory-safety guarantees on shared
class instances across actor boundaries (no formal spec to hand), but
the platform itself never relies on this pattern, and `mut` annotations
on `transform_wrapper` (`ttt.act:1996`) suggest mutation is tracked. At
minimum this is a smell; at worst it's a data race or a violation of the
language's actor-isolation invariant.

**Fix options, in order of preference:**

1. **Push, don't pull.** During the `act` callback, capture an action ref
   on the runner (e.g. `runner._stash_dynstate`) and store it on the
   `SwInstallTransform` instance via an init-time setter. Then
   `transform_wrapper` calls `self._stash_action(dynstate)` instead of
   writing the field. The action lands inside the runner's actor context
   safely.

   ```acton
   class SwInstallTransform(ttt.TransformFunction):
       var stash_cb: ?action(?gdata.Node) -> None = None
       var stash_done: bool = False

       def transform_wrapper(self, cfg, linked, memory, dynstate):
           cb = self.stash_cb
           if cb is not None and not self.stash_done:
               cb(dynstate)
               self.stash_done = True
           return (gdata.Container(), memory)
   ```

   The runner installs `stash_cb` at construction, before
   `act` returns the `on_local_config` proc.

2. **Or, gate the consume on `dynstate_initialized` only and skip the
   shared field altogether by relying on the *first* `on_local_config`
   call's `cfg`/`mem` reaching the runner with restored state already
   reflected in `mem` (if the runner's state can be reconstructed from
   `cfg`+`mem` alone)** — but the whole point of dynstate is precisely
   that it can't, so option 1 is the realistic path.

3. **Worst-case fallback:** if Acton actually does guarantee this works
   (single-writer, single-reader-after-writer-finishes), the design
   needs a paragraph somewhere stating that explicitly with a citation
   to the language docs, because reviewers can't tell otherwise.

This is the single biggest hole in v5, and it's structural — the same
hole would reopen if any future iteration also needs to thread state
"sideways" between the function and its actor.

### HIGH 2 — §3.7 "publish-before-side-effect + idempotent-on-re-fire" is fragile for `auto_started_after_confirm`

Re-reading §4.2 + §3.7 together carefully:

- Tier A says: persist `auto_started_after_confirm = True` *before* the
  auto-start side effect.
- Tier B says: `current.status` is persisted at step boundary (not at
  the unprocessed → processing transition).

If `update_dynstate` writes the *whole* dynstate snapshot atomically (and
the code path for "publish Tier A field" always serializes the whole
in-memory `self.dynstate` including the in-memory-but-not-yet-Tier-B
status), then writing the flag and any in-memory status update happen
together — and the framing works.

But §3.7 explicitly *splits* them: flag is Tier A (persist now), status
is Tier B (persist at next step boundary). That means:

```
t0:  detect confirm-all done + auto-execute=true + flag==False
t1:  set self.dynstate.auto_started_after_confirm = True
t2:  call update_dynstate(self.dynstate.to_gdata())   # Tier A — includes status==unprocessed
t3:  set self.dynstate.current.status = processing    # in-memory only
t4:  schedule first step run
t5:  step completes, write Tier B (status now persisted)
```

If crash at t2.5 (between dynstate write and the in-memory status flip),
restart sees:

- `last_confirm_all_generation == config.confirm_all_generation` (no re-trigger)
- `auto_started_after_confirm == True` (auto-start gate is closed)
- `current.status == unprocessed`
- `start-generation == last_start_generation` (no manual-start gate either)

The runner sits in unprocessed forever. Operator has to manually bump
`start-generation` to recover. That's the "more brittle than expected"
window.

Since this *only* affects auto-execute, and the operator does have a
manual recovery lever (bump `start-generation`), it's not a showstopper.
But the design's claim that v5 makes auto-execute "safe under re-fire"
is too strong. Either:

- **(A)** Co-publish the status with the flag: Tier-A write happens
  *with* `current.status = processing` already set in-memory. Then
  either both land or neither lands. Re-fire after crash sees flag=False
  (because Tier-A write didn't happen), reattempts auto-start.
- **(B)** Document the brittle window: §15.5 entry along the lines of
  "v1 auto-execute can deadlock at unprocessed if a crash lands between
  Tier-A flag persistence and Tier-B first-step persistence; recovery is
  manual `start-generation` bump."

Option (A) is cleaner and probably what the design *means* — but the
text doesn't say so.

The same observation applies to `last_request_generation` /
`next_request_id` if their write is meaningfully separated from the
write that creates `current = NewRequest(...)`. If they're co-published
in one snapshot, fine. If not, the same gap exists. The design should
explicitly say: "Tier A field updates are always batched into the same
`update_dynstate(...)` snapshot as the consequences they trigger; the
'Tier' classification governs *when* `update_dynstate` is called, not
*what* it contains."

### HIGH 3 — 5-second `missing-global-config` guard vs subscription period

§7.2 says:

```acton
params.lower.declare_subscriptions(
    owner_id="sw_install:" + devname,
    cb=runner.on_global_config,
    want={SubscriptionSpec(filt=SOFTWARE_INSTALL_FILTER, period=...)},
)
```

`period=...` is unspecified. Two facts about the platform's
subscription mechanism (`ttt.act:713-733`):

1. `_add_owner_spec` calls `_sub_tick(spec)` *immediately* on first
   declaration (line 722).
2. `_sub_tick` calls `get_data(filt)` against the layer's current root
   (line 704) and publishes the result to owners.

If the runner's `act` callback fires during `Layer` construction (which
is the design's intent — the act callback is invoked from `init_dynstate`
during `_create_transform_node`), and `Layer` construction completes
before `load_from_db()` is called (per the boot sequence in
`app.act:138-152`), then the *first* tick of the subscription samples
the lower layer **before its DB has been restored** and therefore
delivers `None` to `runner.on_global_config`. The next sample fires
`after layer_subscription_delay(st.spec)` (line 687) — i.e. after one
full `period`.

So even a perfectly-wired host topology produces this sequence:

```
t = 0    : declare_subscriptions called from act()
           _sub_tick fires synchronously, on_global_config(None, None)
t = T0   : load_from_db completes (T0 << period typically)
t = T1   : recompute fires; transform_wrapper stashes; on_local_config fires
t = T1+5s: §7.2 guard fires runner-status = missing-global-config (FALSE POSITIVE)
t = period: next _sub_tick → on_global_config(restored_global_config, None)
           runner-status = ok
```

The window is silent in the doc. Operators monitoring `runner-status`
will see flapping unless `period <= 5s`. The fix is one of:

- Pin `period` to ≤5s in the design (and in the §7.2 example), accepting
  the polling cost.
- Reframe the guard to "first non-`None` cb" not "5s after first
  on_local_config". I.e. start the 5s timer when the cb has fired at
  least once with `merged is None` and we're still waiting for a real
  value. (Slightly fiddly but correct.)
- Rephrase as "missing-global-config-or-still-syncing" so operators
  don't panic.

Tangent: the design's §14 v2.0 ask #4 ("Layer 'subscribe to current
layer root' API") would obviate this entirely — the host could subscribe
to the same layer the transform sits in, and the data is *guaranteed*
present after recompute. Worth strengthening that ask: it's not just an
ergonomic improvement, it would close this race entirely.

---

### MEDIUM 4 — `make_sw_install_transform` factory is hand-waved

The public-API signature is correct, but how the factory threads
the `SwInstallTransform` instance through *both* the function-factory arg
and the `act` callback isn't shown. Given that:

- `ttt.Transform(function, act, ...)` calls `function(log_handler)` to
  construct the transform-fn and then calls `act(params)` on it (via
  `init_dynstate`); both happen during a single `_create_transform_node`
  invocation but at different lexical points.
- The runner needs a *reference* to the SwInstallTransform instance for
  the stash read (or the action-ref push, per HIGH-1).

The Phase 4 implementer needs concrete code shape, not just type
signatures. A 10-line skeleton in §7.3 would prevent a wrong choice
(e.g. global mutable holder, or worse — different instance for the
function vs the runner).

Suggested addition for §7.3:

```acton
def make_sw_install_transform(dev_registry, file_cap, ...):
    def factory(path, lower):
        # one transform-fn instance per device; the closure captures it
        # for both the function-factory and the act callback
        fn_holder: list[SwInstallTransform] = []

        def function_factory(log_handler):
            fn = SwInstallTransform()
            fn_holder.append(fn)
            return fn

        proc def act(params):
            fn = fn_holder[0]
            devname = devname_from_swi_path(params.path)
            runner = DeviceRunner(
                params.path, params.update_oper, params.update_dynstate,
                dev_registry.get(devname), fn, ...
            )
            # install the stash callback (HIGH-1 fix)
            fn.stash_cb = runner._stash_dynstate
            if params.lower is not None:
                params.lower.declare_subscriptions(
                    owner_id="sw_install:" + devname,
                    cb=runner.on_global_config,
                    want={SubscriptionSpec(filt=SOFTWARE_INSTALL_FILTER, period=5.0)},
                )
            return lambda cfg, mem: runner.on_local_config(cfg, mem)

        return ttt.Transform(function_factory, act=act)(path, lower)

    return factory
```

Note also: `ttt.Transform`'s `lower` keyword arg
(`ttt.act:1907`) is **dead** — it's shadowed by the inner `_create_transform_node(path, lower)`
parameter and never used. Don't bother passing it (the design's example
in §2 currently does pass `lower=lower` in spirit). This is a platform
bug worth flagging upstream but doesn't affect sw-install correctness.

### MEDIUM 5 — `transform_wrapper` returning `memory` unchanged keeps old memory across calls

§3.6's stash code:

```acton
def transform_wrapper(self, cfg, linked, memory, dynstate):
    if not self.stashed_dynstate_consumed:
        self.stashed_dynstate = dynstate
    return (gdata.Container(), memory)
```

returns `memory` unchanged. The §3.1 ownership table notes "Transform
`memory` — unused". Returning `memory` (rather than `None`) means the
platform's `_TransformTransaction.update_memory(newmem)` keeps writing
the same `memory` blob back to LMDB on every commit
(`ttt.act:1958-1959`, then `ttt.act:1962` writes via
`swdb.opt_ops(..., ATTR_MEMORY, essay.1)`). If `memory` is always `None`,
this is a no-op. If anything ever flowed through `memory` during early
boot (before stash consumed?), it would persist. Recommend returning
`None` to make the "memory unused" claim airtight and avoid surprising
LMDB writes:

```acton
return (gdata.Container(), None)
```

Costs nothing; removes a class of "wait, what's in our memory blob?"
debugging surprises.

### MEDIUM 6 — §15.5 missing two deviations the design actually makes

Cross-checking §15.5's 19 items against §3-§9:

- **`runner-status` oper enum** is brand-new (no Python equivalent).
  It's introduced in §4.8 and is in the YANG model, but §15.5 doesn't
  list it. Add: *"§N. Per-device `runner-status` oper enum surfaces
  startup / topology / restore / pause / device-readiness conditions
  with no Python analogue."*
- **Trigger mechanism: NSO actions → generation-counter leaves +
  optional `*-target-id`**. §15.5 #9 mentions "action-style return
  values replaced by per-device `last-create-result` and single-slot
  `last-trigger-result`" but elides the input side — the *trigger*
  itself moves from imperative action callback to monotonic config
  counter. This is the single biggest API-shape deviation operators
  porting from NSO will trip over. Promote it to its own §15.5 entry.

Also worth tightening:

- **§15.5 #5** ("Generation counters: dynstate-internal restore
  inconsistency is detected; cross-cutting backup-restore safety is a
  v2.0 follow-up") — fine as written, but the body wording in §3.5 still
  uses "config newer than dynstate" once (clarification: the *direction*
  the check doesn't catch is dynstate-restored-from-stale-backup against
  newer-config; either way the §3.5 prose is correct, the §15.5 entry
  could just point at it).

### MEDIUM 7 — Phase 4 starts with a runner that never gets `on_local_config` if no software-pack is bound (§3.6 corollary)

Per A1's caveat: a fresh-install app where no
`/sw-rfs:rfs[name=X]/software-pack` has been authored will park the
runner at `runner-status=starting` indefinitely (no `running` config →
no `compute` → no `on_conf` → no `on_local_config`). The 5s
`missing-global-config` window never opens.

This isn't broken (there's nothing for the runner to do) but it's a
*fourth* terminal state the runner-status enum doesn't model. Either:

- Document explicitly in §7.2: "`starting` is the steady-state value
  when no `software-pack` is bound to this device. Operators should
  treat `starting` plus an empty `request` list as 'no work
  configured' rather than 'failure to initialize'."
- Or add a separate enum value (`unbound`?) for the no-pack case so
  operators don't have to ambiguate `starting`.

### LOW 8 — Doc bug: "PathContainer" doesn't exist (§7.1)

The §7.1 helper comment claims `_DeviceTransaction.devname_from_device_path`
expects `path: PathKey` but sw-install's `params.path` is a `PathContainer`.
There's no `PathContainer` in `ttt.act` — the actual class is `PathElem`.
Single-word fix.

### LOW 9 — Q2 should be marked resolved (§12)

`dev_registry.get(devname)` is precedented at `_RFSTransaction.__init__`
(`ttt.act:2071`) and `testing.act:38`, both calling it synchronously and
using the result directly. The design can just close Q2 — no Phase-4
verify step needed.

### LOW 10 — `ttt.Transform`'s `lower` kwarg is dead code (platform observation)

Noted in MEDIUM 4 above; logging here as a separate platform-side
observation. The user-supplied `lower` to `ttt.Transform(...)` is
shadowed by the `_create_transform_node(path, lower)` parameter
(`ttt.act:1908`). All three places `lower` is consumed downstream
(`_Transform.__init__`, `_TransformTransaction.__init__`,
`_TransformTransactionBase` runtime usage) read the runtime-passed
parameter, not the user's keyword. If sw-install or any consumer relies
on passing `lower=` for any reason, it silently doesn't apply. Worth a
mention to the platform team — not an sw-install issue per se.

---

## C. What's clearly right and shouldn't change

- §3.6 stash mechanism (modulo HIGH-1's transport question): the
  *temporal* claim is right; the verification trace through `app.act`
  matches. Do not regress to v4's "platform addition" path even if
  HIGH-1 forces a different transport.
- §7.1 plain-`Transform`-not-`RFSTransform` decision: the trade-off
  analysis is correct. Both `_RFSTransaction.finalize`'s empty-output
  guard and `RFSFunction.init_dynstate`'s missing `lower` are real, and
  matter for sw-install. The v2.0 platform ask (parameterize
  finalize *or* thread lower through RFS) is the right way to realign
  later.
- §4.4 cancel drain comparison `<` instead of `==` (CL4_6): correct;
  the multi-generation-stale token case is real after
  cancel→restart→execute.
- §4.2 `auto_started_after_confirm` flag (CR4_5): the *intent* is
  right; HIGH-2 is about wording, not deletion.
- §9.6 nested `scp-port` revert: correct; the round-4 flag was real and
  the speculative non-sw-install consumer never materialized.
- The phasing decision (Phase 4 = NETCONF-only SROS, Phase 5 = CLI +
  FileTransfer + IOS-XR + Junos) and the ADR's `DeviceOps` boundary
  commitment are clean. Phase 4 delivers a useful subset without
  pre-committing to Phase 5's TextFSMPlus details.

---

## D. Suggested concrete edits before Phase 4 coding starts

In rough priority:

1. **§3.6 / §7.3** — replace the stash-field read with an action-ref
   push (HIGH-1). Updates §7.3's `SwInstallTransform` skeleton + §8.2's
   `DeviceRunner` skeleton + adds one line to §1's file list (or a
   sub-section in §7.3).
2. **§3.7** — add an explicit invariant: "Tier A field updates are
   always batched into the same `update_dynstate(...)` snapshot as
   the consequence they trigger." Then revisit the
   `auto_started_after_confirm` paragraph to either (a) co-publish
   status (preferred) or (b) document the brittle window in §15.5
   (HIGH-2).
3. **§7.2** — pin `period` (recommend 5.0) in the example, *or* rephrase
   the guard timer in terms of "first non-None cb" rather than "first
   `on_local_config` + 5s" (HIGH-3). Strengthen §14 v2.0 ask #4 to call
   this out as a correctness benefit, not just ergonomic.
4. **§7.3** — flesh out the `make_sw_install_transform` factory body
   with concrete closure-sharing code so the Phase 4 implementer can't
   pick the wrong shape (MEDIUM 4).
5. **§3.6 transform_wrapper** — return `(gdata.Container(), None)` not
   `(gdata.Container(), memory)` (MEDIUM 5).
6. **§15.5** — add two deviations: `runner-status` oper enum (no Python
   analogue), and "trigger mechanism: actions → generation counters" as
   a standalone item (MEDIUM 6). Cross-link §15.5 #9.
7. **§7.2** — document the no-software-pack-bound steady-state as a
   `starting` outcome operators should expect (MEDIUM 7), or add a
   distinct enum value.
8. **§7.1** — `PathContainer` → `PathElem` doc fix (LOW 8).
9. **§12** — mark Q2 resolved by existing precedent (LOW 9).
10. **§14** — note `ttt.Transform`'s `lower=` kwarg is dead code; either
    make the platform consume it or remove the kwarg (LOW 10).

None of these are big enough to push Phase 4 back significantly; HIGH-1
through HIGH-3 are doc + small skeleton-code changes (≤ a few hundred
lines of design diff). HIGH-1 has implementation impact (the stash
mechanism touches both `SwInstallTransform` and `DeviceRunner` shape)
but the impact is local and the fix is small.

---

## E. Round-5 questions answered

| Q | v5 status | This review's view |
|---|-----------|--------------------|
| Q1 | "verify in Phase 4" | **Resolved**. Trace through `_TransformTransaction` + `app.act` shows the stash *timing* is correct. Modulo HIGH-1's *transport* concern. |
| Q2 | "verify in Phase 4" | **Resolved by existing precedent** (`testing.act:38`, `ttt.act:2071`). |
| Q3 | "resolved by D5" | Resolved. ✓ |
| Q4 | "resolved by §2 + §7.2" | **Partially.** §7.2's 5s guard works, but interacts badly with subscription `period` (HIGH-3). |
| Q5 | "1000/req" | Reasonable. Not exercised by my review. |
| Q6 | "confirmed" | Phasing is clean (per §C). |
| Q7 | "600s" | Reasonable. Not exercised. |

---

## F. Overall

I'd green-light Phase 4 implementation immediately after the HIGH items
are addressed in the design (which is doc-only or doc-plus-skeleton work
on the order of a few hours). The medium and low items can be folded in
during Phase 4 if it's faster to do them once the implementer hits the
relevant sections.

The architectural shape is sound. The control surface (generation
counters + targeted target-ids + last-result single-slot) is a clean
mapping of NSO actions onto a reactive substrate, and the lifecycle is
well-traced. The biggest open question — HIGH-1 — is about *transport*
of the stash, not *whether the stash works*; that question's been
soundly answered.
