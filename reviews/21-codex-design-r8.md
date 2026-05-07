# Round-8 Codex Design Review

Reviewer: Codex
Date: 2026-05-07

Scope read in requested order:

1. `docs/00-orientation.md`
2. `docs/01-software-install-logic.md`
3. `docs/02-sw-install-design.md` v5.3
4. `docs/adr/cli-driver.md`
5. `stratoweave/sw-install/src/sw_install/yang.act`

Then sanity-checked against live Stratoweave:

- `stratoweave/stratoweave/src/stratoweave/ttt.act`
- `stratoweave/stratoweave/src/stratoweave/app.act`
- `stratoweave/stratoweave/src/stratoweave/device.act`
- `stratoweave/stratoweave/src/stratoweave/adapters/adapter.act`
- `stratoweave/sorespo/src/sorespo/rfs.act`
- `stratoweave/sorespo/src/sorespo/layers/t_2.act`
- `stratoweave/sorespo/src/sorespo/sysspec.act`

I formed the assessment from the doc set and live code first, then read `docs/reviews/18-codex-design-r7.md`, `docs/reviews/19-claude-design-r7.md`, and `docs/reviews/20-integration-r7.md` to check divergence.

## Verdict

Phase 4 is ready from an architecture and live-platform standpoint. The two round-7 blocking items have landed in the main design:

- §4.1's request-generation idempotency check now requires both `current.materialized_by_request_generation == cfg_request_gen` and `target_pack == last_pack_snapshot`, and it is scoped inside Trigger A.
- §7.2 no longer contains the stale `transform_fn` constructor snippet and points to §7.3 as the canonical construction site.

I would not require another full review round before implementation. I do recommend a small doc-only cleanup before or at the start of Phase 4, because there are still a few stale secondary statements that contradict v5.3's corrected algorithm and version labeling. The only correctness-relevant one is §3.7's prose summary of the generation anchor; §4.1 itself is correct.

## Findings

### MEDIUM 1 - §3.7 still states the pre-v5.3 over-broad generation short-circuit

`docs/02-sw-install-design.md` §3.7 line 325 says:

```text
if the trigger's request-generation value equals the current request's
materialized_by_request_generation, the current request already corresponds
to this trigger - no-op.
```

That is the exact simplification v5.3 had to narrow in §4.1. The corrected rule is not generation-only; it is generation plus pack snapshot equality, and it only suppresses the explicit-generation path when the target pack also matches `last_pack_snapshot`.

The main §4.1 pseudocode is correct, so this is not a design blocker. But §3.7 is the Tier-A crash-safety section, and an implementer could reasonably read it as the invariant to enforce. That would recreate the r7 bug where a pack-data change with the same request-generation gets suppressed.

Recommendation: rewrite the §3.7 bullet to match §4.1, e.g. "if request-generation equals `current.materialized_by_request_generation` AND `target_pack == last_pack_snapshot`, the current request already corresponds to this trigger and pack snapshot - no-op; otherwise allow Trigger A or B to materialize."

### LOW 1 - §7.3 still has an elided `file_transfer_factory(...)` call despite the v5.3 fix note

Round-7 integration asked for the factory args to be spelled out. §7.3 now adds a useful comment listing the remaining constructor args, but the call still contains:

```acton
file_transfer_factory and file_transfer_factory(...) or NoopFileTransfer(),
```

This leaves the one non-obvious piece unresolved: where the `DeviceMetaConfig` argument comes from for the public API shape `proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer`. §9.5 points toward `DeviceMgr.get_dmc()`, so the intended call is probably something like "get DMC from DeviceMgr, then call `file_transfer_factory(dev, dmc)` when available." That should be written in the skeleton or called out directly in the comment.

This is doc completeness, not an architecture problem, but it is in the "key shape Phase 4 implementation must follow" snippet.

### LOW 2 - Version labels lag v5.3 in orientation and YANG revision text

`docs/00-orientation.md` line 8 still says `docs/02-sw-install-design.md` is "current: v5.2". `stratoweave/sw-install/src/sw_install/yang.act` revision description still starts "v5.2 design - reflects round-6 review integration on top of v5.1."

The design doc itself is clearly v5.3, and the YANG content that matters for Phase 4 still matches the design. This is just stale framing, but this round was explicitly a lock-in review of v5.3, so the doc set should not send a fresh reader back to v5.2.

Recommendation: update orientation to "current: v5.3" and either update the YANG revision description to mention v5.3 or intentionally say the YANG payload is unchanged from v5.2 while the design doc is v5.3.

## v5.3 Fix Verification

Landed correctly:

- §4.1's idempotency short-circuit is now inside Trigger A and requires both the materialized generation and pack snapshot equality.
- Trigger B remains reachable for pack-data changes when request-generation has not advanced.
- §7.2 replaced the stale `transform_fn` construction body with a focused snippet and a reference to §7.3.
- §3.6 and §8.4 now describe per-list-entry construction for both restored entries (`Layer.load_from_db()` replaying `ATTR_KEYS`) and fresh entries (`ListState.acquire`).
- §3.6.1 explicitly documents the deliberate `_stash_dynstate(None)` substitution for the integration's `_signal_no_restore()` recommendation.
- §7.2 now describes `_owner_publish` accurately: callback fires on error or `merged != last_merged`.
- The 15s missing-global-config grace is used in §2, §7.2, and the YANG runner-status description.
- Stale `_signal_no_restore` comments appear removed from the active design snippets.

Still stale:

- §3.7's one-line prose summary of the generation anchor did not get the v5.3 pack-equality qualifier.
- §7.3 still contains `file_transfer_factory(...)`.
- Orientation/YANG version labels still say v5.2.

## Platform Sanity Check

The core substrate assumptions check out against live code:

- `StartupBootstrap._run` loads LMDB before forced recompute, so restored `_TransformTransaction.dynstate` is visible to `transform_wrapper(..., dynstate)`.
- Per-list-entry construction is as v5.3 describes: `_List.restore` calls `ListState.recreate` for restored entries, and `ListState.acquire` constructs fresh entries on first config edit.
- Plain `ttt.Transform` remains the right substrate. `_TransformTransaction.finalize` does not suppress empty output, while `_RFSTransaction.finalize` does; `RFSFunction.init_dynstate` also does not pass `lower`.
- `Layer._owner_publish` fires the owner callback when `err is not None` or `merged != owner.last_merged`; therefore the "merged stays None forever" topology error needs the watchdog.
- `DeviceRegistry.get(name)` is synchronous and is used that way by `_RFSTransaction` and tests.
- `SubscriptionSpec(period=5.0)` is consistent with existing sorespo usage and prior live-code verification.
- The YANG control surface contains the expected generation counters, confirmation list, `runner-status`, request history, run-log `(when, seq)` key, and typed diagnostic projections.

## Recommendation

Green-light Phase 4. Do the three doc cleanups above as a pre-implementation polish pass, not as another architecture iteration. The remaining issues are small, local, and do not change the implementation plan.
