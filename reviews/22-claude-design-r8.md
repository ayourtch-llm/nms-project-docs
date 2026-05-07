# 22 — Claude Round-8 Design Review (lock-in)

**Reviewer:** Claude (pty-1)
**Subject:** `docs/02-sw-install-design.md` v5.3 + `00-orientation.md` + `01-software-install-logic.md` + `docs/adr/cli-driver.md` + `stratoweave/sw-install/src/sw_install/yang.act`
**Method:** read the doc set top-to-bottom as a fresh reader (no carry-over from rounds 1-7); verified design claims against live `stratoweave/stratoweave/src/stratoweave/{ttt.act, app.act, device.act, adapters/adapter.act}` and `sorespo/src/sorespo/{rfs.act, layers/t_2.act}`. Formed the assessment from the doc set + live code first, then read `docs/reviews/{18,19,20,21}-*.md` to spot-check divergence.

---

## Verdict

**Phase 4 is ready to start.** The v5.3 fix list landed correctly: the §4.1 idempotency-check rescope (codex r7 H1) and the §7.2 stale `transform_fn` snippet replacement (claude r7 H1) are both in place, the §3.6 / §8.4 per-list-entry construction prose is corrected, and the smaller cleanups all show up where they should. Architecture, lifecycle, FIFO, and reconciliation stories check out against the live platform code.

I have no new HIGH findings. I do have one MEDIUM that is doc-only — the §4.1 algorithm got the v5.3 pack-equality qualifier, but §3.7's one-line prose summary of the same anchor did not, so the two sections now contradict each other on the short-circuit rule. An implementer who reads §3.7 as the Tier-A invariant will recreate the round-7 bug. Plus a couple of LOW items (factory-arg elision, version-label drift). All three are < 30 minutes of edits and do not require another full review round.

This matches codex r8's recommendation. We converged independently; that is a positive signal for lock-in.

Convergence trajectory: HIGH count `6 → 5 → 3+3 → 2+2 → 1+1 → 0+0` (per-reviewer). Within the round-7 integration's predicted "zero findings or 1 cosmetic item" envelope.

---

## Findings

### MEDIUM 1 — §3.7 still describes the pre-v5.3 generation-only short-circuit; contradicts §4.1

**Location:** `02-sw-install-design.md` §3.7 line 325 (the `materialized_by_request_generation` bullet under Tier A).

The bullet currently reads:

> The §4.1 reconciliation rule becomes: "if the trigger's `request-generation` value equals the current request's `materialized_by_request_generation`, the current request already corresponds to this trigger — no-op."

But §4.1's actual v5.3 rule (lines 381-389) is the rescoped two-condition check:

```acton
if (self.dynstate.current is not None and
    self.dynstate.current.materialized_by_request_generation == cfg_request_gen and
    target_pack == self.dynstate.last_pack_snapshot):
    return    # crash-recovery short-circuit
```

That is exactly the codex r7 HIGH 1 fix: requiring pack equality so a same-generation pack-data change can still fall through to Trigger B. §3.7 still has the over-broad pre-fix wording.

§4.1 itself is correct — this is **not** an algorithm regression. But §3.7 is the section that frames the Tier-A crash-safety invariant, and an implementer who treats §3.7's bullet as the spec will write the buggy form (suppressing pack-data changes when request-generation hasn't moved). The whole point of v5.3 was to remove that footgun from the doc.

**Fix.** Rewrite the line 325 bullet to mirror §4.1, e.g.:

> The §4.1 reconciliation rule becomes: "if the trigger's `request-generation` value equals the current request's `materialized_by_request_generation` AND `target_pack == last_pack_snapshot`, the current request already corresponds to this trigger and pack snapshot — no-op; otherwise allow Trigger A or Trigger B (pack-change) to materialize."

(Codex r8 raised the same item independently as their MEDIUM 1; I called it independently before reading their review.)

---

### LOW 1 — §7.3 factory body still elides `file_transfer_factory(...)` args

**Location:** `02-sw-install-design.md` §7.3 line 727:

```acton
file_transfer_factory and file_transfer_factory(...) or NoopFileTransfer(),
```

The v5.3 cleanup added a useful comment listing the closure-captured remaining args (`local_fi`, `remote_fi_factory`, `file_transfer`, `ops_factory`, `cli_session_factory`, `log_handler`), which is good. But the one non-trivial constructor — `file_transfer_factory` — is still called as `file_transfer_factory(...)`. Per the public API at `02-sw-install-design.md` lines 67-68, the factory's signature is:

```
file_transfer_factory: ?proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer = None
```

So an implementer needs to know where the `DeviceMetaConfig` comes from. §9.5 names `DeviceMgr.get_dmc()` as the credential-reuse path, so the intended call shape is presumably:

```acton
dev = dev_registry.get(devname)
dmc = dev.get_dmc()        # or however the API surfaces it
ft = file_transfer_factory(dev, dmc) if file_transfer_factory is not None else NoopFileTransfer()
```

The current snippet leaves that ambiguous. Phase 4 will likely figure it out at the keyboard, but this is in the section the design explicitly calls out as "the key shape Phase 4 implementation must follow," so it should be concrete.

**Fix.** Either spell the call out (preferred), or add a comment of the form "where `dmc = dev.get_dmc()` per §9.5".

(This was on the integration r7 LOW 2 punch list and was the one item only partially addressed in v5.3.)

---

### LOW 2 — Version labels lag v5.3 in two places

**`docs/00-orientation.md` line 8:**

```
> 3. `docs/02-sw-install-design.md` — proposed Acton module design (current: v5.2)
```

Should read `current: v5.3`. The body of 00-orientation didn't otherwise need v5.3-specific edits, so a one-character change.

**`stratoweave/sw-install/src/sw_install/yang.act` line 27 (revision description):**

The revision `2026-05-07` description begins:

```
v5.2 design — reflects round-6 review integration on top of v5.1.
```

The YANG schema content is unchanged from v5.2 (none of the v5.3 changes are schema-affecting), so a new YANG revision is not required. But the revision description is now misleading: this revision is what the v5.3 design doc points at. Either:

- update the revision description's first line to `v5.3 design — no schema change from v5.2; design doc bumped for prose corrections (§4.1 idempotency rescope, §7.2 / §3.6 / §8.4 prose, §3.6.1 substitution note).`, or
- explicitly state in 02-sw-install-design.md §16 that the YANG payload is unchanged from v5.2 and the revision description intentionally still reads v5.2.

Both are fine; I lean toward the YANG-side update because the YANG file's revision description is what RESTCONF clients and `gen_adata` consumers will read, and "design v5.3 with v5.2 YANG payload" is exactly the truthful statement.

(Codex r8 raised the same drift as their LOW 2.)

---

## v5.3 fix verification

Each item from `docs/reviews/20-integration-r7.md` checked against the v5.3 doc:

| Integration item | Status in v5.3 |
|------------------|----------------|
| HIGH 1 (§4.1 idempotency rescope inside Trigger A, requires both `materialized_by_request_generation` and `last_pack_snapshot` match) | **Landed** — lines 376-399 are the rescoped form; Trigger B at lines 404-411 is reachable when only pack changes. |
| HIGH 2 (§7.2 stale `transform_fn` snippet replaced) | **Landed** — §7.2 lines 569-588 now show only the subscription/watchdog-relevant slice and reference §7.3 as canonical; no `transform_fn` parameter anywhere. |
| MEDIUM 1 (§3.6 / §8.4 lifecycle prose for per-list-entry construction) | **Landed** — §3.6 lines 228-243 walk both restored and fresh paths through `Layer.load_from_db` ATTR_KEYS replay vs `ListState.acquire`; §8.4 step 2 is in the same shape. |
| LOW 1 (§3.6.1 `_signal_no_restore` substitution note) | **Landed** — §3.6.1 lines 305 documents the deliberate `_stash_dynstate(None)` overload of the integration's recommendation. |
| LOW 2 (§7.3 factory args spelled out) | **Partial** — comment added (lines 721-724); `file_transfer_factory(...)` still elides the DMC arg. See LOW 1 above. |
| LOW 3 (§7.2 `_owner_publish` prose tightened) | **Landed** — lines 604-611 now correctly say cb fires on `err is not None` OR `merged != last_merged`. |
| LOW 4 (§2 5-second → 15-second reference) | **Landed** — §2 line 124 says "default 15s after runner construction". |
| LOW 5 (stale `_signal_no_restore` comments removed from §3.6) | **Landed** — only mention left is the §3.6.1 substitution note (deliberate). |

The one outstanding item is §3.7's prose summary — see MEDIUM 1 above. That is a v5.3 oversight: the writer fixed §4.1 itself but didn't scan §3.7 for the same wording.

---

## Platform sanity check

I re-verified the substrate references in the design against live `ttt.act` / `app.act` / `device.act`. All check out:

- `app.act:138-152` is `StartupBootstrap._run`; `await async cfs.load_from_db()` precedes `cfs.newsession().recompute(_recompute_done, force=True)`, so the forced recompute sees restored dynstate. ✓
- `ttt.act:610-643` is `Layer.load_from_db` / `load_from_read_txn`. The cursor scan walks `swdb.PATH_SEPARATOR` / `swdb.ATTRIBUTE_SEPARATOR` boundaries — ATTR_KEYS replays before ATTR_DYNSTATE for any given list-entry path. ✓
- `ttt.act:1232-1331` is the `_List` / `ListState` per-list-entry construction. `_List.restore` (1285-1299) special-cases `ATTR_KEYS` to call `liststate.recreate(key, leaves.children)`; `ListState.recreate` (1355-1356) invokes `template(PathKey(...), lower)` — that's the `swi_factory` for sw-install. `ListState.acquire` (1316-1331) is the fresh-edit path. ✓
- `ttt.act:436-445` is `merge_subscription_tree` (returns `None` until any spec has data); `ttt.act:663-672` is `_owner_publish` (fires cb when `err is not None or merged != owner.last_merged`). The "`None == None` doesn't transition" claim that motivates the §7.2 watchdog is correct. ✓
- `ttt.act:1907` is `Transform(function, act=..., lower=..., ...)`. `_create_transform_node(path, lower)` is the inner proc; the outer `lower=` kwarg is shadowed and unused — the §14.1 platform observation is real. ✓
- `ttt.act:1942-1993` covers `_TransformTransaction.compute`, `update_memory`, `db_ops`, `restore`, `init_dynstate`, `update_dynstate`, and `finalize`. `compute` calls `transform_wrapper(merged, linked, self.memory, self.dynstate)`; `finalize` calls `self.function.on_conf(self.get(), self.memory)` only `if self.running:` — no empty-output suppression on plain `Transform`. ✓
- `ttt.act:2042-2053` is `devname_from_rfs_path`, the structural precedent for `devname_from_swi_path`. The hardened sw-install version (§7.1) adds a namespace check, which the platform helper lacks — design rationale is consistent with the live code. ✓
- `ttt.act:2071` is `_RFSTransaction.__init__`'s `self.dev = dev_registry.get(self.devname)` — synchronous, used directly. The Q2 resolution holds. ✓
- `ttt.act:2151-2158` is `_RFSTransaction.finalize` with the empty-output suppression (suppresses `on_conf` if output is an empty non-presence Container). This is exactly the reason sw-install uses plain `Transform` instead of `RFSTransform`. ✓
- `ttt.act:2184-2185` is `RFSFunction.init_dynstate` — passes `dev`, not `lower`, into `TransformActorParams`, confirming the second reason for the deliberate departure (`params.lower` would be unavailable for the global-config subscription). ✓
- `device.act:760` confirms `get_modules() -> (dict[str, ModCap], ?str)` — tuple shape per §8.7's CR4_2 fix. ✓
- `device.act:786` and `ttt.act:735` are the two `declare_subscriptions` surfaces; sw-install uses the layer one (per §7.3). ✓

Cross-checked sorespo for the closest live precedent of the wiring shape:

- `sorespo/src/sorespo/layers/t_2.act` shows `RFSTransform(BBInterface, dev_registry, BBInterfaceTransformCtor, lower, log_handler)`. The ctor is the `act`-equivalent; it constructs `BBInterfaceTransform` (an actor) and returns a `lambda c, m: act.on_conf(...)`. sw-install uses `Transform` (not `RFSTransform`) for the §7.1 reasons but the act-callback shape is structurally the same.
- The closure-shared `fn_holder` pattern in §7.3 is sw-install-specific (sorespo's transform classes don't need a back-reference from the function instance to the runner actor). The platform allows it; the FIFO + same-Transaction-actor argument for safety holds.

One small precision quibble on §7.3 (not a finding, just noting it for any future doc pass): the line 732-737 comment says "the act callback runs in the same actor that owns `fn` (the Transaction actor), so this is a same-actor write." Strictly, `fn = function_factory(log_handler)` runs synchronously inside `_TransformTransaction.__init__`, which executes in the **caller** of `template(...)` — i.e. the `ListState` actor for restored entries, or the actor running `ListState.acquire` for fresh entries. The Transaction actor receives the constructed `_TransformTransaction` (with `fn` already populated) as its `impl` argument. Subsequent `fn` accesses (the `act(params)` body, future `transform_wrapper` calls) all run inside the Transaction actor, and they are causally after the `init_dynstate` message that follows the construction. So the safety claim is correct — same-actor for all subsequent reads/writes — but "same actor that owns `fn`" is technically a post-message-pass statement, not a construction-time one. Cosmetic; do not block on this.

---

## What v5.3 got right

Everything from the integration r7 fix list except the two LOW items above. The substantive correctness pieces — §4.1 algorithm, §7.2 watchdog, §3.6 lifecycle, FIFO mailbox argument, Tier A batching invariant, plain-`Transform` substrate rationale, devname helper hardening, scp-port placement, runner-status precedence, topology-misconfigured test requirement — all survive a fresh read. The doc set is internally consistent on the implementation-blocking material; the remaining issues are stale text that didn't get rescanned during the targeted v5.3 pass.

---

## Recommendation

**Green-light Phase 4.** The v5.3 changes did the work they were supposed to do. The one MEDIUM finding (§3.7 prose) and the two LOWs (§7.3 factory call, version-label drift) are doc-only and add up to maybe 30 minutes of edits. They are best done as a quick polish pass at the **start** of Phase 4 (or even concurrently with the first scaffold + `pack.act` cycle) rather than as a v5.4 round.

If we want a clean lock-in: bundle the §3.7 fix, the `file_transfer_factory(dev, dmc)` spelling, and the v5.3 version-label updates into a single short commit, then start `acton build` on the scaffold. No more architecture iteration is warranted.

---

## Note on independence

I intentionally formed this assessment from the v5.3 doc set + live code before reading codex r8 (`docs/reviews/21-codex-design-r8.md`). When I did read it, the three items above are the same three items codex r8 surfaced (their MEDIUM 1 = my MEDIUM 1; their LOW 1 = my LOW 1; their LOW 2 = my LOW 2). Two reviewers landing on the same exact list independently is the strongest signal we can get that v5.3 is genuinely close to lock-in and these specific items are the residue. After they're cleared, the design is done.
