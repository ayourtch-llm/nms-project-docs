# Round-9 Codex Review — v5.3.2 Self-Containment Verification

Reviewed: `docs/02-sw-install-design.md` v5.3.2 (`0f9bc13`)

Verdict: **Phase 4 remains green.** I found **no semantic drift** in the newly inlined sections relative to the v3/v4/v5 material that prior rounds validated. The architecture, trigger/idempotency model, runner lifecycle, Tier-A/Tier-B/Tier-C invariants, cancellation watchdog/drain algorithm, pre-staged-image handling, `scp-port` placement, and Phase 4 scope are unchanged from the v5.3.1 lock-in design.

One readability issue remains for the specific v5.3.2 self-containment objective: a few non-inlined "carried/unchanged" placeholders are still present outside the listed 14-section lift. This does not reopen Phase 4 architecture, but it means the document is not quite self-contained for a fresh reader yet.

## Findings

### MED — Remaining placeholder references still require git-diving for a fresh reader

`02-sw-install-design.md:476`, `02-sw-install-design.md:482`, `02-sw-install-design.md:1218`, `02-sw-install-design.md:1228`

v5.3.2 successfully inlines the major empty sections called out in the request, but these remaining placeholders still point back to prior versions:

- §4.7 says "(Rest of v4 §4.7 carried unchanged.)" after adding only the v5 inter-step gate sentence. The actual enabled-state transition table is not present in v5.3.2.
- §4.8 has a table row saying "(carry v4 entries unchanged unless noted)" instead of inlining the baseline YANG diff entries.
- §10 says "(Mostly carried from v3.)" and omits the concrete test list from v3, except for the v5.2 topology test addition.
- §11 says "(Otherwise unchanged.)" and omits the concrete Phase 4 implementation sequence from v3.

Impact: this is a self-containment/readability miss, not a design correctness issue. A reader can still understand the locked architecture by following the v5 sections, but a fresh implementation reader still has to git-dive for enabled semantics, the full YANG diff baseline, test coverage expectations, and Phase 4 sequencing.

Recommended fix: inline those four remaining carried blocks, preserving the v5 overrides already present. This should be another content-only pass.

## Verification Notes

- Compared v5.3.2 against v5.3.1: the diff is limited to the status/header wording, heading suffix cleanup, and prose insertion/replacement in the previously stubbed sections.
- Checked the inlined §4.5, §5/§5.1/§5.2, §6.1-§6.5/§6.7, §8.1/§8.5/§8.6, §9.1-§9.4/§9.7, §13, and §15 text against the earlier authored versions and the later v5 corrections. I did not find drift from the validated design.
- Checked the newly inlined sections against the previously fleshed-out v5.x sections. No contradictions found. Notable preserved v5 corrections include:
  - `clear-run-log` resets `run_log_dropped` and per-request `seq`.
  - step callback shape includes `lfi`, `rfi`, `ft`, and `step_log`.
  - `ComponentPlan` failure handling discards `NewState` on failure.
  - retry/backoff keeps decimal internal state and ceilings the persisted/projected seconds.
  - cancel uses stale-token `< current.generation_token` plus the 600s watchdog.
  - Phase 4 `NoopFileTransfer` still relies on `RemoteFileInspector` for pre-staged images.
  - `DeviceOps` keeps Phase 4 NETCONF-only SROS behavior without dead CLI stubs.

Bottom line: **no Phase 4 blocker, no semantic drift, no contradiction in the lifted prose.** The only remaining issue is that v5.3.2 is not fully self-contained until the four residual "carried/unchanged" placeholders above are also expanded.
