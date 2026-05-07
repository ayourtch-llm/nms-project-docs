# 17 — Integration of Round-6 Design Reviews

Consolidates:
- `docs/reviews/15-codex-design-r6.md` — Codex (pty-2)
- `docs/reviews/16-claude-design-r6.md` — Claude (pty-3)

**Headline: both reviewers' verdicts are "green-light Phase 4 after a v5.2 doc-only pass."** Independent convergence on the same 2 HIGHs and 3 MEDIUMs is the strongest signal we've had: no architectural disagreement, no platform-side changes needed, only doc precision remaining.

Convergence pattern across rounds: **HIGH count 6 → 5 → 3+3 → 2+2** (identical between reviewers at r6). Architecture endorsed unanimously. Round 7 expected to be the lock-in.

---

## Convergent issues (both reviewers found independently)

| # | Issue | Decision |
|---|-------|----------|
| **A1** | **Action-ref stash has two install sites.** §3.6's "Code shape" puts `transform_fn.stash_cb = self._stash_dynstate` inside the DeviceRunner actor body — that's exactly the cross-actor mutation v5.1's CL5_1 was meant to fix. §7.2/§7.3's factory install (`fn.stash_cb = runner._stash_dynstate` inside `act`) is correct. Two install sites also race if both run. | **Drop §3.6's actor-body install. Remove `transform_fn` from `DeviceRunner` constructor signature.** §7.3 factory is single source of truth. Reword §3.6 prose to say "the act callback installs the runner's _stash_dynstate action ref into the function instance, before transform_wrapper can fire." |
| **A2** | **`missing-global-config` timer described in 3 inconsistent places.** §7.2 prose: "first non-None on_global_config callback." §7.2 state diagram: "5s timeout from runner construction." YANG: "5s of first config callback." None is implementable as written. Worse: claude r6 traced `_owner_publish` and found the cb does NOT fire with `None` — it doesn't fire at all when `merge_subscription_tree` returns None. So "first non-None" means the timer never starts when the topology is wrong (the very alarm the timer exists for). | **Replace with one concrete algorithm.** Anchor at runner construction (or first `on_local_config`); fire after grace ≥ subscription period + load_from_db budget (suggest 15s); flag `runner-status = missing-global-config` if `global_config_seen == False` at expiry, regardless of whether any callbacks fired. Mirror in YANG. |
| **A3** | **§4.1 has no reconciliation pseudocode.** Header says `### 4.1 create-request (unchanged)` with empty body. §3.7 references "the §4.1 reconciliation rule becomes...". An implementer has nothing concrete to read. The `materialized_by_request_generation` anchor needs explicit precedence: pack-change vs cancelled-forces-new vs request-generation > last_observed; what value to write for pack-change-driven materialization. | **Replace `(unchanged)` with concrete pseudocode.** Cover the three triggers, their precedence, and what value `materialized_by_request_generation` takes per path. |
| **A4** | **§8.4 implicit FIFO assumption between `_stash_dynstate` and `on_local_config`.** Both are action sends to the runner's mailbox. v5.1's §3.6 fallback ("if not dynstate_initialized, treat as fresh install") is dead code if FIFO is guaranteed (good); but if not guaranteed (or if mailbox arrives by different routes), the fallback silently drops restored dynstate. | **Commit explicitly to FIFO** (with citation in §8.4) and remove the §3.6 fallback. Distinguish the genuinely-fresh case via a `_signal_no_restore()` action sent on the empty-restore path. |
| **A5** | **Topology-misconfiguration regression risk.** Layer wiring constraint is documented (§2 / §7.2), but failure mode is `runner-status = missing-global-config` after a watchdog fires. No compile-time check; no startup error. One app composition mistake away from regression. | **Add `test_topology_misconfigured.act` to §10.** Required Phase 4 test that wires the sw-install transform without `/software-install/` pass-through and asserts `runner-status=missing-global-config`. Serves as a copy-paste template for app integrators. |

---

## Convergent LOW items (also identified by both)

| # | Issue | Fix |
|---|-------|-----|
| **L1** | **Stale "v5" / "round-5" tail text** in `00-orientation.md` ("current: v5"), §16 ("Round-5 review"), `yang.act` revision text. | Update all to v5.2 / round-7. |
| **L2** | **§3.3 IOS-XR projection prose contradicts YANG.** Prose says "v5 YANG models only the common+SROS subset" but YANG includes `op-id-*` under IOS-XR `when`. | Reword: "common diagnostics + SROS `rebooted` + IOS-XR `op-id-*`. IOS-XR `packages`/`reload-required` and Junos per-RE diagnostics deferred to Phase 6." |
| **L3** | **§8.7 still has stale Q2 warning** referencing "❓Round-5 question Q2" — but §12 marks it resolved. | Remove the warning or recast as confirmation. |

## Claude-r6-unique low items

| # | Issue | Fix |
|---|-------|-----|
| **CL6_L4** | **§7.1 cites wrong existing helper.** Compares against `_DeviceTransaction.devname_from_device_path` (`ttt.act:2206`); the closer-shaped reference is `devname_from_rfs_path` (`ttt.act:2042-2053`) which already walks ancestors. | Update citation. |
| **CL6_L5** | **`fn_holder: list[T] = []` as single-cell** is unidiomatic; comment explaining why a list is used (no first-class mutable cell; tuples immutable; closures need mutable container). | Add inline comment. |

---

## v5.2 action items (priority-ordered)

Five HIGH/MEDIUM items + 5 LOW items. All doc/skeleton fixes. Estimated effort: under an hour.

1. **§3.6 / §8.2 / §7.3** — drop `transform_fn` from DeviceRunner constructor; remove §3.6 actor-body stash_cb install; §7.3 factory is single source. Reword §3.6 prose. (A1)
2. **§7.2 / yang.act** — replace timer description with one concrete algorithm anchored at runner construction with grace ≥ subscription period. Mirror in YANG. (A2)
3. **§4.1** — replace `(unchanged)` with concrete reconciliation pseudocode covering pack-change / cancelled-forces-new / request-generation triggers, their precedence, and `materialized_by_request_generation` semantics per path. (A3)
4. **§8.4 / §3.6** — commit explicitly to FIFO mailbox ordering; drop §3.6 fallback; add explicit `_signal_no_restore()` for the empty-restore case. (A4)
5. **§10** — add `test_topology_misconfigured.act` requirement. (A5)
6. **§3.3** — fix IOS-XR projection prose. (L2)
7. **§8.7** — remove stale Q2 warning. (L3)
8. **§7.1** — update helper citation to `devname_from_rfs_path`. (CL6_L4)
9. **§7.3** — add comment explaining `fn_holder: list[T] = []` single-cell. (CL6_L5)
10. **Stale version text** — `00-orientation.md` "current: v5.2", §16 "Round-7 review", `yang.act` revision. (L1)

---

## Round-7 expectation

If v5.2 lands these 10 items cleanly:
- The convergence trajectory (6 → 5 → 3+3 → 2+2) suggests round 7 finds 0-1 items.
- Both reviewers' explicit "green-light after these fixes" verdict means r7 is the lock-in.

If round 7 surfaces another batch of 2+ HIGHs (unlikely given current trajectory), we go to v5.3. But round 6's findings are now 100% doc-precision rather than substrate or architecture — there's not much surface left for new HIGHs to emerge from.

**Phase 4 is now genuinely close to starting.** Once round 7 confirms the v5.2 fixes, the next step is `acton build` of the scaffold + first TDD cycle (probably `pack.act` + `test_pack.act`).
