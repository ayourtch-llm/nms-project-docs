# 19 — Claude Round-7 Design Review (lock-in)

**Reviewer:** Claude (pty-1)
**Subject:** `docs/02-sw-install-design.md` v5.2 + `00-orientation.md` + `01-software-install-logic.md` + `docs/adr/cli-driver.md` + `stratoweave/sw-install/src/sw_install/yang.act`
**Method:** read the doc set top-to-bottom as a fresh reader (no carry-over from rounds 1-6); verified design claims against live `stratoweave/stratoweave/src/stratoweave/{ttt.act, app.act, device.act, adapters/adapter.act}`, `acton-yang/src/yang/gdata.act`, and `sorespo/src/sorespo/rfs.act`; consulted `docs/reviews/17-integration-r6.md` and the round-6 reviews only **after** forming this assessment, to spot-check whether items I found had already been raised.

---

## Verdict

**Phase 4 is ready to start once one HIGH item is fixed.** The HIGH item is a doc-only oversight from the v5.2 integration pass — §7.2 didn't get the same v5.2 update that §3.6 / §7.3 / §8.2 received for round-6 A1, so two parts of the doc now contradict each other on the DeviceRunner constructor signature. ~10-line fix.

Architecture is sound. The lifecycle, FIFO, and reconciliation stories all check out against the live platform code. v5.2's round-6 fixes (§4.1 pseudocode, §7.2 watchdog algorithm, §3.6.1 empty-restore signal, §10 topology-misconfigured test, §3.3 prose, §8.7 cleanup) all landed correctly **except** that §7.2's act() sketch wasn't refreshed to match.

Convergence trajectory: HIGH count `6 → 5 → 3+3 → 2+2 → 1` (single reviewer this round). Within the predicted "0-1 items" envelope.

---

## Findings

### HIGH 1 — §7.2 act() sketch still uses v5.1 `transform_fn` parameter; contradicts §7.3 and §8.2 (v5.2 oversight)

**Location:** `02-sw-install-design.md` §7.2 lines 552-561.

The §7.2 act() sketch reads:

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
    ...
```

But:

- **§8.2 DeviceRunner constructor (lines 728-738)** explicitly **drops** `transform_fn` per round-6 A1: `dev: swdev.DeviceMgr,  # 🆕 v5.2 per A1: no transform_fn parameter`. The 10-arg constructor signature has no slot for `transform_fn`.
- **§7.3 factory body (lines 687-710)** is the v5.2-corrected shape: it installs `fn.stash_cb = runner._stash_dynstate` **after** `runner = DeviceRunner(params.path, params.update_oper, params.update_dynstate, dev_registry.get(devname), ...)` (no `transform_fn` arg), and notes "v5.2 per A1: single-owner stash_cb install."

So §7.2 contradicts both §8.2 (constructor signature) and §7.3 (call site). A reader following §7.2 verbatim would write code that fails to compile. The integration doc (`17-integration-r6.md` item A1) explicitly called for "Drop §3.6's actor-body install. Remove `transform_fn` from `DeviceRunner` constructor signature... §7.3 factory is single source of truth"; v5.2 applied this to §3.6, §7.3, and §8.2 but missed §7.2.

A secondary issue compounds it: §7.2's sketch uses the variable name `transform_fn` (no closure-captured `fn_holder` shown), while §7.3's variant uses `fn = fn_holder[0]`. Reconciling §7.2 to match §7.3 makes the closure shape consistent across both sections.

**Fix.** Replace the §7.2 sketch with one that mirrors §7.3:

```acton
proc def act(params: ttt.TransformActorParams) -> ?proc(gdata.Node, ?gdata.Node) -> None:
    fn = fn_holder[0]                   # the SwInstallTransform constructed by function_factory
    devname = devname_from_swi_path(params.path)
    runner = DeviceRunner(
        params.path,
        params.update_oper,
        params.update_dynstate,
        dev_registry.get(devname),
        ...                             # remaining args per §8.2 constructor
    )
    fn.stash_cb = runner._stash_dynstate                # v5.2 A1: same-actor write inside Transaction actor
    if params.lower is not None:
        params.lower.declare_subscriptions(
            owner_id="sw_install:" + devname,
            cb=runner.on_global_config,
            want={SubscriptionSpec(filt=SOFTWARE_INSTALL_FILTER, period=5.0)},
        )
    return lambda cfg, mem: runner.on_local_config(cfg, mem)
```

Either keep §7.2 as a focused "global-config subscription + watchdog" sketch and forward the construction details to §7.3, or unify the two by making §7.2 reference "see §7.3 for the act callback body" without re-printing the runner construction. The current state — two side-by-side, contradictory transcriptions — is the worst option.

---

### MEDIUM 1 — §3.6 step 1 / §8.4 step 1 lifecycle prose is wrong about where `SwInstallTransform` is constructed (pre-existing, masked by every prior round)

**Location:** `02-sw-install-design.md` §3.6 lines 226-235; §8.4 lines 770-776.

Both sections claim:

> 1. `Layer(...)` rootgen constructs `SwInstallTransform` and calls `act(...)`. The runner is constructed **here**, before any restore.

This is incorrect for the per-list-entry transform topology that §7.1 prescribes. Verified against the live code:

- `Layer(...)` (`ttt.act:604`) calls `rootgen(PathRoot(name), lower)` once at construction time. For a `List` (`ttt.act:1232-1242`), construction creates a `ListState` with the swi `template` **stored**, but no list entries — so `swi_factory` (and therefore `SwInstallTransform.__init__` and the act callback) is **not** invoked at Layer rootgen.
- The actual construction sites are:
  1. **Restored case:** during `Layer.load_from_db()` (`ttt.act:610-643`), the LMDB scan delivers `ATTR_KEYS` records first (lexicographic ordering, per the comment at `ttt.act:1289-1292`). Each `ATTR_KEYS` triggers `_List.restore` → `ListState.recreate(key, leaves)` → `template(PathKey(...), lower)` → `_create_transform_node` → `_TransformTransaction.__init__` (which calls `function_factory(log_handler)`, constructing `SwInstallTransform` and appending to `fn_holder`) → `init_dynstate` queued to the Transaction actor → eventually `act(params)` runs and constructs the `DeviceRunner`. **This is where the runner is born for restored entries**, *during* `load_from_db`, not before it.
  2. **Fresh case:** when the user's first `edit_config` creates the list entry via `ListState.acquire` (`ttt.act:1316-1331`), `template(...)` is invoked then. The runner is constructed at that moment, with `self.dynstate=None` because nothing was ever persisted.

The design's correctness conclusion holds — by the time `transform_wrapper` fires (during compute), `init_dynstate` has been processed (FIFO into the Transaction actor's mailbox), so `stash_cb` is set and the FIFO `_stash_dynstate` → `on_local_config` ordering still holds. But the prose-level claim "Layer rootgen constructs SwInstallTransform" misleads anyone trying to instrument the lifecycle, debug a startup race, or reason about why restored vs fresh entries take different paths.

**Fix.** Reword §3.6 step 1 (and the parallel §8.4 step 1) along the lines of:

> 1. `SwInstallTransform` is constructed **per list-entry**, when the entry is materialized:
>    - For **restored** entries, this happens during `Layer.load_from_db()` while replaying `ATTR_KEYS` records (`ttt.act:1285-1297`). `ListState.recreate(key, leaves)` invokes `swi_factory(path, lower)` → `_TransformTransaction.__init__` calls `function_factory(...)` (constructing `SwInstallTransform`) → `init_dynstate` is queued to the Transaction actor → `act(params)` runs, constructing the `DeviceRunner` and installing `stash_cb` on the function instance.
>    - For **fresh** entries (user `edit_config` creates the list entry), the same chain runs synchronously inside `ListState.acquire` during configure.
>
>    In both cases the runner exists with empty in-memory dynstate before `transform_wrapper` first fires for that entry.

This correctly anchors steps 2-7 of the lifecycle without changing any of the timing claims that follow.

---

### LOW 1 — §3.6.1's "always called" empty-restore signal silently substitutes for the integration's `_signal_no_restore()` recommendation

**Location:** `02-sw-install-design.md` §3.6.1 lines 290-298.

Round-6 integration A4 said "commit explicitly to FIFO; drop §3.6 fallback; **add explicit `_signal_no_restore()` for the empty-restore case**." v5.2 commits to FIFO ✓ and drops the fallback ✓, but instead of adding a separate `_signal_no_restore()` action, it overloads `_stash_dynstate(stashed=None)`:

```acton
action def _stash_dynstate(stashed: ?gdata.Node):
    if stashed is not None:
        self.dynstate = SwInstallDynstate.from_gdata(stashed)
    self.dynstate_initialized = True
    self._check_restore_consistency()
```

This is **functionally equivalent** to a separate `_signal_no_restore()` and arguably simpler (one entry point, one mailbox slot, no risk that the two signals could race past each other). It does change the contract subtly: the runner can no longer distinguish "empty dynstate was restored" from "no dynstate was ever persisted" — both arrive as `_stash_dynstate(None)`. For the v5.2 use case (the runner just sets `dynstate_initialized = True` either way and proceeds with `SwInstallDynstate.empty()`), this distinction doesn't matter.

**Fix (optional, ≤2 lines).** Add a one-line note in §3.6.1 explicitly acknowledging the substitution: "(The integration A4 recommendation was a separate `_signal_no_restore()` action; we use `_stash_dynstate(None)` instead because both the FIFO guarantee and the indistinguishability of "empty restore" vs "no restore" make the simpler form correct.)" Otherwise a future reader cross-referencing the integration doc may flag this as a missed action item.

---

### LOW 2 — §7.3 factory sketch elides the public-API factory args that DeviceRunner actually consumes

**Location:** `02-sw-install-design.md` §7.3 lines 687-710 vs the public API in §1 lines 62-73.

`make_sw_install_transform`'s signature carries `local_file_inspector`, `remote_file_inspector_factory`, `file_transfer_factory`, `cli_session_factory`, `log_handler`, `file_cap`. The §7.3 act callback constructs `DeviceRunner(params.path, params.update_oper, params.update_dynstate, dev_registry.get(devname), ...)` — the `...` elides exactly these factories, even though §8.2's DeviceRunner constructor lists them as required positional args (`local_fi`, `remote_fi_factory`, `file_transfer`, `ops_factory`, `cli_session_factory`, `log_handler`).

This is a **doc completeness** issue, not a correctness one — §8.2 fully defines what needs to be passed, and the §7.3 sketch is admittedly a "skeleton." But the round-7 reader has to splice §7.3 + §8.2 + §1 together to reconstruct the wiring, and the splice has one minor decision point: how does the `act` closure capture the factories? Presumably `make_sw_install_transform` closes over them and the inner `factory`/`act` proc-defs read them via closure (the same way `dev_registry`, `log_handler`, `file_cap` are already captured).

**Fix (optional).** Add one line of comment in §7.3 like `# remote_fi_factory, file_transfer, ops_factory, cli_session_factory closed over from make_sw_install_transform's outer scope; passed positionally per §8.2`.

---

### LOW 3 — §7.2 prose understates when `_owner_publish` fires the cb

**Location:** `02-sw-install-design.md` §7.2 lines 583-585.

The §7.2 prose says:

> Layer.declare_subscriptions only fires the cb when merge_subscription_tree returns non-None — never with a None merged tree (per ttt.act _owner_publish).

Verified against `ttt.act:663-672`:

```acton
def _owner_publish(owner_id: str, err: ?Exception=None):
    ...
    merged = merge_subscription_tree(owner.want, subs)
    if err is not None or merged != owner.last_merged:
        owner.last_merged = merged
        sub_owners[owner_id] = owner
        owner.cb(merged, err)
```

The cb fires when **either** `err is not None` **or** `merged != owner.last_merged`. So:
- On error during `_sub_tick` (`ttt.act:708-711`), the cb fires regardless of merged.
- On the first non-None merged value (last_merged was None), the cb fires.
- On steady "merged stays None forever" (topology wrong, no /software-install/ visible), the cb stays silent — **this** is the case that motivates the watchdog.

The design's understanding of why the watchdog is required is correct (the "missing forever" case is real and undetectable via cb alone). But the prose oversimplifies — errors do fire the cb. The §7.2 `on_global_config` handler ignores the err arg, which is fine for the watchdog but should be acknowledged.

**Fix (optional).** Tighten the prose to "only fires the cb when `err is not None` OR `merged != last_merged`; the latter cannot transition out of `None == None`, which is why a topology-wrong runner gets no cb until something independent (the watchdog) intervenes."

---

## Verifications that passed

I checked the v5.2 round-6 fix list against both docs and the live code; below is what I confirmed actually landed and is consistent with the platform.

| # | Round-6 item | Status |
|---|---|---|
| A1 | Single-owner `stash_cb` install in factory | **Partially landed.** §3.6, §7.3, §8.2 all carry the v5.2 shape. §7.2 missed (HIGH 1 above). |
| A2 | Concrete `missing-global-config` watchdog algorithm | **Landed.** §7.2 lines 570-606 has one algorithm, anchored at runner construction with `MISSING_GLOBAL_CONFIG_GRACE = 15.0`. YANG description (`yang.act:263-309`) mirrors prose. Watchdog "fires unconditionally" — addresses the "missing forever" case correctly. |
| A3 | §4.1 reconciliation pseudocode | **Landed.** §4.1 lines 341-422 covers all three triggers (request-generation / pack-change / cancelled-forces-new), precedence (A > B > C, with cancelled-forces-new requiring an explicit request-generation bump), `materialized_by_request_generation` semantics per path, and the crash-recovery example. The "idempotency check 1" short-circuit at the top is the right shape for re-fire-after-crash. |
| A4 | FIFO commitment + empty-restore signal | **Landed with a deviation** (LOW 1 above). §3.6 commits to FIFO, drops the fallback, and uses `_stash_dynstate(None)` instead of a separate `_signal_no_restore()`. Functionally equivalent. |
| A5 | `test_topology_misconfigured.act` required | **Landed.** §10 line 837 calls it out as a Phase 4 requirement and frames it as a "copy-paste template for app integrators." |
| L1 | Stale "v5" / "round-5" tail text | **Landed.** §16, `00-orientation.md` reading-order, `yang.act` revision text all read v5.2 / round-7. |
| L2 | §3.3 IOS-XR projection prose | **Landed.** Section now correctly enumerates which projections are common, SROS-only, IOS-XR-only-modeled, IOS-XR-only-deferred, and Junos-deferred. |
| L3 | §8.7 stale Q2 warning | **Landed.** §8.7 lines 811-812 reframes the resolution as a confirmation citing `_RFSTransaction.__init__` / `testing.act:38`. |
| CL6_L4 | §7.1 helper citation → `devname_from_rfs_path` | **Landed.** §7.1 line 545 cites `devname_from_rfs_path` at `ttt.act:2042-2053` and explains why we don't reuse it directly (no namespace check, not public). |
| CL6_L5 | `fn_holder: list[T] = []` explained | **Landed.** §7.3 lines 678-680 has the inline comment about no first-class `Cell[T]`/`mut[T]` and tuples being immutable. |

I also re-verified the architectural claims that previous rounds locked in:

- **Lifecycle ordering against `app.act:138-152` and `ttt.act:1942-1993`.** `cfs.load_from_db()` runs first, then `cfs.newsession().recompute(force=True)`. `recompute(force=True)` triggers `compute()` (which calls `transform_wrapper(merged, linked, memory, dynstate)` with the restored dynstate — `ttt.act:1942-1948`), then `finalize()` (which calls `function.on_conf(self.get(), self.memory)` if `self.running` is non-empty — `ttt.act:1989-1993`). Order matches §3.6 / §8.4. ✓
- **`_TransformTransaction.finalize` does NOT suppress on empty output.** Confirmed: `ttt.act:1989-1993` only checks `if self.running` (no output check). The empty-output suppression is `_RFSTransaction.finalize`-specific (`ttt.act:2151-2158`). The §7.1 "deliberate departure" rationale is correct. ✓
- **`RFSFunction.init_dynstate` does not pass `lower` through.** Confirmed: `ttt.act:2184-2185` constructs `TransformActorParams(path, update_dynstate, update_oper, dev)` with no `lower` arg (the 5-positional ctor at `ttt.act:1898` accepts `lower` keyword, but `RFSFunction.init_dynstate` doesn't pass one). The plain `TransformFunction.init_dynstate` (`ttt.act:2011-2012`) DOES pass `lower`. The §7.1 deliberate-departure rationale is correct on both points. ✓
- **`Transform()`'s `lower=` kwarg is dead.** Confirmed: `ttt.act:1907` accepts `lower: ?Layer=None` but the inner `_create_transform_node(path, lower)` at line 1908 shadows it. The kwarg passed at `Transform(lower=...)` time is silently dropped. §14.1's CL5_6 platform observation is correct. ✓
- **`dev_registry.get(name)` is synchronous.** Confirmed: `device.act:77-81` is a `def` (not `proc def`) in the `DeviceRegistry` actor; the lazy `DeviceMgr` construction returns the instance directly. `_RFSTransaction.__init__` (`ttt.act:2071`) and `testing.act:38` both use the result inline. ✓
- **`SubscriptionSpec(period=5.0)` is 5 seconds.** Confirmed: `gdata.act:3137-3150` `_normalize_period` accepts float as seconds and converts to `u64` nanoseconds (`u64(period * 1e9)`). `ttt.act:429-433` `layer_subscription_delay` divides by `1e9` to get seconds back. So `period=5.0` is 5 s, and the §7.2 grace (15 s) ≥ subscription period (5 s) inequality holds. ✓
- **`merge_subscription_tree` returns `None` when no spec has data.** Confirmed: `ttt.act:436-445` returns `None` if no `sub.latest is not None` for any spec in `want`. The cb-fire condition in `_owner_publish` (line 663-672) means a runner subscribed to `/software-install/` that's never visible in the lower layer never gets a cb invocation — exactly why the watchdog is required. ✓
- **YANG runner-status enum + precedence + description.** Confirmed: `yang.act:263-309` lists six enum values (`starting`, `ok`, `missing-global-config`, `restore-inconsistent`, `paused-by-enabled`, `waiting-for-device`) with the precedence ordering in the description (`restore-inconsistent > missing-global-config > waiting-for-device > paused-by-enabled > starting > ok`) matching §7.2's algorithm. ✓
- **YANG control surface + run-log keying.** Confirmed: all five generation counters (`request-generation`, `start-generation`, `cancel-generation`, `confirm-all-generation`, `clear-run-log-generation`) plus their optional `*-target-id` companions exist. `confirmation` list keyed by `(request-id component step)` with mandatory `by-user`. `request-options/confirm-steps` per-request override. `run-log` keyed by `(when seq)` with `seq` resetting on `clear-run-log-generation` increment. `run-log-dropped` surfaced. All match the design's claims. ✓

---

## Convergence and recommendation

Round-6 integration predicted "0-1 items." This review finds **1 HIGH (doc-only oversight) + 1 MEDIUM (pre-existing prose imprecision) + 3 LOWs (optional polish)**. Within the predicted envelope.

**Recommendation: green-light Phase 4 implementation after a v5.3 doc-only pass.**

Priority order for v5.3:

1. **HIGH 1 §7.2** — replace the v5.1-shaped act() sketch with the v5.2 form (or forward to §7.3 to avoid duplication).
2. **MEDIUM 1 §3.6 / §8.4** — reword "Layer rootgen constructs" to "list-entry materialization constructs" for both restored and fresh paths.
3. **LOW 1 §3.6.1** — note the deliberate substitution of `_stash_dynstate(None)` for `_signal_no_restore()`.
4. **LOW 2 §7.3** — note where the elided `...` factory args come from.
5. **LOW 3 §7.2** — tighten the `_owner_publish` cb-firing prose.

Effort estimate: under 30 min total. None of these change architecture, types, or YANG. After v5.3 lands, the next concrete step is `acton build` of the Phase 4 scaffold and the first TDD cycle (`pack.act` + `test_pack.act`, per the round-6 integration's recommendation).

If v5.3 happens, I'd expect round-8 to be a true zero-finding lock-in (the HIGH 1 item is the last category — "doc-precision oversight in v5.2's integration pass" — that has any plausible surface left).

If the project wants to start Phase 4 immediately and treat the HIGH 1 as a pre-implementation cleanup (i.e., the implementer reads §7.3 / §8.2 as canonical and ignores §7.2's stale sketch), that's also defensible — none of the findings represent architectural risk. The lock-in case for "v5.2 ships as-is" is reasonable; the case for "one more 30-minute pass" is slightly stronger because §7.2 will mislead any future fresh reader (e.g., an external integrator) who hasn't internalized the round-6 → v5.2 rationale.
