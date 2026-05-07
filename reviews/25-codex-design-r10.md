# Round-10 Codex Review - v5.3.3 Full Self-Containment Verification

Reviewed: `docs/02-sw-install-design.md` v5.3.3 (`7cdb388`)

Verdict: **Phase 4 architecture remains green, but v5.3.3 is not yet fully self-contained.** The four r9 placeholder sites are fixed: §4.3 now includes the confirmation shape and `<confirm-all>` sentinel behavior, §4.7 includes the enabled transition table and inter-step gate, §4.8 restores the v3 baseline table with the v5 `scp-port`/`runner-status` overrides, and §10/§11 include the test and implementation sequences.

However, the fresh-reader pass found two remaining content gaps outside the four r9 line cites. Both are documentation/self-containment issues, not architecture reversals.

## Findings

### MED - §4.6 is still too thin for the control-subtree contract it is now referenced to define

`02-sw-install-design.md:464`, `02-sw-install-design.md:486`, `02-sw-install-design.md:519`, `02-sw-install-design.md:1281`

§4.3's newly inlined confirmation YANG says "see §4.6 for the full list of generation/target leaves," and §4.8/§11 both depend on §4.6 as the authoritative `control/` subtree reference. But current §4.6 only contains the single-slot `last-trigger-result` polling caveat; it does not inline the earlier v4 scoping rule:

- every generation counter except `request-generation` has an optional `*-target-id`;
- set target exists -> accepted and applied;
- set target missing -> rejected via `last-trigger-result`;
- unset target -> applies to the latest request;
- `last-trigger-result` carries trigger kind, generation, target id, result, reason, and timestamp.

Impact: an implementer using v5.3.3 alone still has to infer or git-dive the target scoping and `last-trigger-result` contract. This also weakens the §4.8 claim that the full YANG baseline is present, because the table lists `last-create-result` but not the v4-added `last-trigger-result` oper subtree even though §4.6 and §15.5 #9 require it.

Recommended fix: inline the v4 §4.6 body, updated with v5 wording, and add an explicit §4.8 row for `/sw-rfs:rfs[name]/software-pack/last-trigger-result` as oper feedback for `start`/`cancel`/`confirm-all`/`clear-run-log`.

### MED - §9.5 is an empty cross-reference target for credential reuse

`02-sw-install-design.md:940`, `02-sw-install-design.md:947`, `02-sw-install-design.md:1226`, `02-sw-install-design.md:1277`

§8.2 and §11 now depend on §9.5 for `DeviceMgr.get_dmc()` and the use-time credential discipline, but §9.5 is only a heading. The actual v4 text established the important details: `DeviceMgr.get_dmc()` already exists at `device.act:401`, `FileTransfer` uses that existing API, the factory shape is `proc(swdev.DeviceMgr, DeviceMetaConfig) -> FileTransfer`, and DMC should be read at use time because it is mutable via `set_dmc(...)`.

Impact: v5.3.3's implementation sequence tells Phase 4 to rely on §9.5, but a fresh reader cannot recover the credential contract from §9.5 itself. The surrounding comments preserve fragments of the rule, so this is not semantic drift, but it is still a self-containment miss.

Recommended fix: inline the v4 §9.5 body, preserving the v5.2 skeleton's resolved call shape: construct the factory with `dev.get_dmc()` when needed, and for long-lived transfer objects call `dev.get_dmc()` inside `put/delete` to avoid stale credentials.

### LOW - One residual "carried" pointer remains outside version history

`02-sw-install-design.md:350`

The sweep for placeholder language still finds: "(Mostly carried from v4; the substantive v5 changes are listed inline.)" This one is not as damaging as the r9 placeholders because the §4 subsections now contain substantial prose, but it conflicts with the "fully self-contained" framing and is easy to remove.

Recommended fix: replace it with a content summary of the control surface, or delete it.

## Verification Notes

- Confirmed the four r9 line-cited placeholders are expanded in v5.3.3.
- §4.7 remains aligned with §8.3 invariant 4: the enabled transition table is present and the inter-step gate is preserved.
- §10 preserves the expected test hooks: `clear-run-log` resets, stale-token `< current.generation_token`, `materialized_by_request_generation` idempotency, and the v5.2 topology-misconfigured test.
- §11 preserves the Phase 4 sequence hooks: `transform_wrapper` stash, existing `DeviceMgr.get_dmc()` reference, `NoopFileTransfer`, and `RemoteFileInspector` for pre-staged images.
- Placeholder sweep found no remaining literal `(unchanged)` / `(otherwise unchanged)` implementation stubs, but did find the §4 intro "Mostly carried" wording above.

## Ship/Hold Call

**Hold v5.3.3 as the "fully self-contained" document** until §4.6 and §9.5 are inlined. **Phase 4 architecture remains green**; the hold is on the readability/self-containment claim, not on the locked v5.3.1 design decisions.
