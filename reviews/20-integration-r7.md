# 20 — Integration of Round-7 Design Reviews

Consolidates:
- `docs/reviews/18-codex-design-r7.md` — Codex (pty-2)
- `docs/reviews/19-claude-design-r7.md` — Claude (pty-3)

**Headline:** Both reviewers' verdicts are "green-light Phase 4 after these fixes." Round-7 produced **1+1 HIGHs (different items)** — within the round-6 integration's predicted "0-1 items" envelope. Convergence trajectory: **6 → 5 → 3+3 → 2+2 → 1+1.**

The two HIGHs are different in kind:
- Codex r7 caught a **genuine correctness bug** in §4.1's idempotency check ordering. Must fix.
- Claude r7 caught a **stale snippet** in §7.2 that contradicts §7.3 and §8.2 (v5.2 oversight — the round-6 A1 fix didn't propagate to §7.2).

Both are local doc/algorithm fixes. v5.3 is a sub-30-minute pass.

---

## v5.3 fix list (priority-ordered)

### HIGH 1 — §4.1 idempotency check ordering (codex r7 HIGH 1)

The current pseudocode runs:

```
if (current is not None and cfg_request_gen > 0 and
    current.materialized_by_request_generation == cfg_request_gen):
    return    # short-circuit BEFORE pack-change detection
```

**Bug:** if the operator sets `request-generation=42`, request A is materialized with `materialized_by_request_generation=42`. Then if the pack-data later changes (with request-generation still 42), Trigger B (pack-change) never gets to fire — the short-circuit returns first.

**Fix:** rescope the idempotency check to ALSO require pack-data to match. Move it AFTER computing `target_pack` and check against `last_pack_snapshot` too:

```
# Idempotency short-circuit: only suppress if BOTH the explicit-generation
# anchor matches AND the pack data hasn't changed since materialization.
if (cfg_request_gen > 0 and
    self.dynstate.current is not None and
    self.dynstate.current.materialized_by_request_generation == cfg_request_gen and
    target_pack == self.dynstate.last_pack_snapshot):
    return    # crash-recovery short-circuit; no work to do
```

This preserves the crash-recovery property (re-firing the same request-generation against the same pack is no-op) while letting Trigger B fire when pack-data changes within an unchanged generation.

### HIGH 2 — §7.2 stale `transform_fn` snippet (claude r7 HIGH 1)

§7.2's act() sketch still has the v5.1 shape with `transform_fn` as a DeviceRunner constructor arg. v5.2 correctly removed it from §3.6, §7.3, and §8.2 — but missed §7.2.

**Fix:** replace §7.2's act() sketch with the v5.2 shape (mirror §7.3) OR delete the construction details from §7.2 and forward to §7.3 as the single source.

### MEDIUM 1 — §3.6 / §8.4 lifecycle prose imprecision (claude r7 MED 1)

The prose says "Layer(...) rootgen constructs SwInstallTransform" — technically wrong for per-list-entry transforms. Construction actually happens:
- For restored entries: during `Layer.load_from_db()` replaying ATTR_KEYS records (`ttt.act:1285-1297`).
- For fresh entries: during `ListState.acquire` on first edit_config.

In both cases, the runner exists with empty in-memory dynstate before `transform_wrapper` first fires for that entry. The lifecycle conclusions are correct; only the location prose is imprecise.

**Fix:** reword §3.6 step 1 and §8.4 step 1 to say "list-entry materialization constructs SwInstallTransform" with explicit restored vs fresh paths.

### LOW 1 — §3.6.1 _signal_no_restore substitution note (claude r7 LOW 1)

Round-6 integration A4 said "add explicit `_signal_no_restore()` for the empty-restore case." v5.2 instead overloads `_stash_dynstate(None)` for both cases. Functionally equivalent; the integration deviation should be noted explicitly so a future reader doesn't flag it as a missed action item.

**Fix:** add a one-line note in §3.6.1.

### LOW 2 — §7.3 elided factory args (claude r7 LOW 2)

§7.3's act callback constructs `DeviceRunner(...)` with `...` eliding the factory args (`local_fi`, `remote_fi_factory`, `file_transfer`, `ops_factory`, `cli_session_factory`, `log_handler`). Doc completeness — they're closed over from `make_sw_install_transform`'s outer scope.

**Fix:** add a comment in §7.3 noting the elided closure-captured args.

### LOW 3 — §7.2 `_owner_publish` prose imprecision (claude r7 LOW 3)

Prose says cb fires "only when merge_subscription_tree returns non-None — never with a None merged tree." Strictly, the cb also fires on errors (`err is not None`). The watchdog rationale is correct; the prose oversimplifies.

**Fix:** tighten to "fires when `err is not None` OR `merged != last_merged`; latter cannot transition out of `None == None`."

### LOW 4 — §2 stale "5-second" reference (codex r7 LOW 1)

§2 still says watchdog fires "after a 5-second startup window." §7.2/yang.act correctly say 15s.

**Fix:** §2 → "15-second default grace" or refer to `MISSING_GLOBAL_CONFIG_GRACE`.

### LOW 5 — §3.6 stale `_signal_no_restore` comments (codex r7 LOW 2)

§3.6 code comments mention `_signal_no_restore` (e.g., "may be called with stashed=None for the empty-restore path via _signal_no_restore"). §3.6.1 actually says no separate signal exists.

**Fix:** remove `_signal_no_restore` references from §3.6 comments.

---

## What v5.2 got right (per both r7 reviewers)

Both reviewers explicitly verified the following landed correctly and survived live-code scrutiny:

- §3.6 / §7.3 single-owner factory `stash_cb` install — pattern is structurally correct.
- Empty-restore signal via `_stash_dynstate(None)` + FIFO mailbox commitment.
- §3.7 Tier-A batching invariant + IOS-XR `op_id_*` moved to Tier B.
- §7.2 missing-global-config 15s watchdog algorithm — fires unconditionally; addresses "missing forever" correctly.
- §10 `test_topology_misconfigured.act` requirement.
- §3.3 IOS-XR projection prose corrected.
- §7.1 `devname_from_rfs_path` citation.
- §8.7 Q2 stale warning removed.
- YANG `runner-status` description mirrors design §7.2 algorithm.
- Lifecycle ordering (StartupBootstrap.recompute(force=True) → transform_wrapper sees restored dynstate).
- Plain `ttt.Transform` substrate rationale (verified vs `_RFSTransaction.finalize` and `RFSFunction.init_dynstate`).
- `dev_registry.get(name)` synchronous use.
- `SubscriptionSpec(period=5.0)` semantics.
- `merge_subscription_tree` returning None when no spec has data.
- YANG control surface + run-log keying.

These survive v5.3 unchanged.

---

## Round-8 expectation

If v5.3 lands cleanly:
- Codex: predicted "After that one algorithm fix and a small stale-snippet cleanup, I would green-light Phase 4."
- Claude: predicted "If v5.3 happens, I'd expect round-8 to be a true zero-finding lock-in."

Both reviewers would also accept "ship v5.2 with the implementer treating §7.2 as stale and §4.1 as buggy" — i.e., start Phase 4 immediately and fix doc precision in flight. **My recommendation: do v5.3 doc-only pass first** (≤30 min) then round 8. The §4.1 algorithm bug is a real correctness issue; an implementer reading v5.2 verbatim would write the buggy version.

After round 8 confirms zero findings, Phase 4 begins with `acton build` of the scaffold + first TDD cycle (`pack.act` + `test_pack.act`).
