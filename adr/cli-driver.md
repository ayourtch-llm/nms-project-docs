# ADR — CLI driver via TextFSMPlus templates (Phase 5)

> **Status: Phase 5 design intent. Lifted from `docs/02-sw-install-design.md` v2 §9.7 per round-2 reviewer guidance — both reviewers said the §9.7 content was too detailed/committal for a Phase 4 design doc, and recommended an ADR.**

## What Phase 4 commits to (so Phase 5 can build on it)

- **`DeviceOps` facade** in `02-sw-install-design.md` §9.7 — per-OS facade that hides NETCONF/CLI strategy choice from steps. Step signatures take `ops: DeviceOps`.
- **`SrosOps` carries a `cli_session: ?CliSession` field** — Phase 4 constructs with `None` (NETCONF only). **No per-method CLI stubs that raise NotImplementedError** — that would be dead surface; Phase 5 adds CLI methods alongside existing NETCONF ones inside `SrosOps`.
- **`cli_session_factory: ?proc(...)` parameter** on `make_sw_install_transform` — Phase 4 accepts it, ignores when None.
- **`RunLogHandler` honors `swi_redacted=True`** in structured-data dicts — Phase 5 transcript redaction will use this hook so secrets in CLI templates don't pollute the persistent run-log.

If Phase 5 picks up these handoffs cleanly, the boundary chosen in Phase 4 is correct.

## Context

This document captures the design intent for sw-install's CLI driver, to be implemented in Phase 5. It depends on a parallel platform workstream (the acton-utils textfsm extension) that itself isn't yet built. Phase 4 of sw-install ships the **`DeviceOps` boundary** (see `02-sw-install-design.md` §9.7) so step signatures already accommodate the future CLI strategy; everything below is the implementation-side intent that informs Phase 5.

The Python `software_install_script.py` mixes three transports per OS:
- **NETCONF** — most state queries, install/activate/commit RPCs.
- **CLI** — interactive commands like `admin reboot now`, `request system reboot`, `[confirm]`-style prompts, and some show-command output scraping.
- **SCP/SFTP** — image upload via paramiko.

Phase 4 is NETCONF-only and ships `NoopFileTransfer` for byte transfer. Phase 5 adds CLI interaction and a real `FileTransfer`. This ADR is about the CLI side.

## Decision

**Use TextFSMPlus-style templates as the CLI interaction language**, not ad-hoc match-and-respond loops in step bodies.

TextFSMPlus is the user's existing Rust implementation at `/Users/ayourtch/rust/ayclic/aytextfsmplus`. It is a TextFSM superset adding three line-action extensions on top of standard TextFSM:

- **`Send "..."`** — send text to the stream (with `${...}` variable interpolation, evaluated as `aycalc` expressions).
- **`Preset`** — caller-supplied values fed into the engine before it runs.
- **`Done`** — terminal state signaling successful interactive completion.

The same engine drives both **parsing** (existing TextFSM templates from ntc-templates work unmodified — battle-tested at 1790/1818 in the Rust impl) and **interactive sessions** (Send/Preset/Done templates handle prompts).

### Worked example: SROS `admin reboot now`

```
Value Preset Confirm (yes|no)

Start
  ^.*Are you sure.* -> Send ${Confirm} WaitGoodbye

WaitGoodbye
  ^.*[Cc]onnection.*closed -> Done
```

Same engine, same template shape. No separate Expect library, no ad-hoc match loops in step bodies.

## Architecture (Phase 5)

### `CliSession` — the TextFSMPlus driver

```acton
class TextFsmPlusTemplate(value):
    """Compiled TextFSMPlus template — Send/Preset/Done extensions over standard TextFSM."""
    states: dict[str, StateRules]
    values: dict[str, ValueDef]    # incl. Preset declarations

class TextFsmPlusResult(value):
    captured: dict[str, list[str]]      # standard TextFSM output (parse mode)
    final_state: str                    # "Done" on success, error state name on failure

class CliSession(object):
    """Wraps an SSH/Telnet stream; runs TextFSMPlus templates over it.

    Parse mode: feed bytes, extract DataRecords. Interactive mode: feed bytes,
    on Send action emit text downstream, on Done complete successfully.
    """
    proc def run_template(self,
                          tmpl: TextFsmPlusTemplate,
                          presets: dict[str, str],
                          cb: action(?TextFsmPlusResult, ?Exception) -> None
                         ) -> None: ...
    proc def close(self) -> None: ...
```

Backed by the existing `netcli` / `netclics` packages for the underlying SSH/Telnet stream.

### `DeviceOps` strategy selection

Per-OS facades (e.g. `SrosOps`) hold both NETCONF and CLI strategies internally; per-method (or per-instance) they pick which to use based on `dev.get_capabilities()` snapshot at construction. Capability snapshot is **per-run**, not per-step — fail clean on incompatible drift between runs.

```acton
actor SrosOps(
    dev: swdev.DeviceMgr,
    cli_session: ?CliSession,         # None means NETCONF-only mode (Phase 4)
    log_handler: ?logging.Handler,
):
    var prefer_cli: bool = False     # set by capability check at construction

    proc def get_version(self, cb):
        if prefer_cli and cli_session is not None:
            cli_session.run_template(SROS_SHOW_VERSION_TEMPLATE, {}, cb_parse_version(cb))
        else:
            dev.rpc_xml(..., cb_parse_netconf_version(cb))

    # ... and so on
```

Phase 4: `cli_session = None`; the CLI branches are unreachable. Phase 5 wires real CLI sessions and the templates.

### Templates as data, shipped with per-OS modules

Each `ops_<os>.act` ships templates as multi-line string constants — same pattern as the ayclic `templates.rs`. Templates compatible with the broader TextFSM ecosystem can be reused (ntc-templates output parsing slots in unmodified).

## Dependency: acton-utils TextFSM extension

The acton-utils ecosystem already has standard TextFSM. Phase 5 CLI work depends on **extending** it with:

1. **Three line actions:** `Send`, `Preset`, `Done`.
2. **An aycalc-equivalent expression evaluator** for `${...}` interpolation in `Send` actions. Either:
   - port `aycalc` directly (it's a small embeddable evaluator with `GetVar` / `CallFunc` extension traits), or
   - implement just the variable-substitution subset (`${VarName}` lookup with caller-supplied `GetVar`).

Option 2 is cheaper and enough for sw-install's use cases (challenge-response and TOTP are not in scope for the install/upgrade flow).

The Rust reference implementation is mature (1790/1818 ntc-templates compatibility) and small enough to port directly; estimated ~1-2 days of focused porting work for option 2, ~1 week for option 1.

**❓DECISION (separate workstream):** does the acton-utils textfsm extension happen as part of sw-install Phase 5, or as an independent platform-side workstream that sw-install consumes? Lean: independent workstream with sw-install as the first consumer. Captured here so it doesn't get lost.

## Issues to address during Phase 5 implementation

Round-2 reviewers raised several legitimate issues that this ADR does not (yet) resolve. They are open for Phase 5 design:

1. **Prompt synchronization.** When does the engine know "the device has finished printing and is waiting for input"? Need either a known prompt regex or a quiet-time threshold. Templates currently rely on prompt regex; for cases where prompts vary (multi-vendor, BOF mode, configuration mode, login banners) the per-OS template library has to enumerate them.
2. **Paging (`--More--`).** SROS, IOS-XR, Junos all support `terminal length 0` but only after entering an exec-like mode. Templates should run a `terminal length 0` handshake before issuing the real command. This is sw-install-internal — operators may have their own session settings that conflict; document.
3. **Privilege modes.** Cisco IOS-XR's `enable` mode, SROS's admin mode, etc. Templates have to match the prompt and elevate where needed. Existing ayclic templates do this (see `cisco-ios-telnet-login.tfsm` pattern).
4. **Terminal width.** `terminal width 511` sets a wide width to avoid line-wrapping that breaks regex matching. Should be in every interactive session preamble.
5. **Command echo.** Most CLI sessions echo commands back — templates need to consume the echo line before parsing the response. Standard TextFSM patterns handle this (`Filldown` etc.).
6. **Secrets in presets and logs.** Passwords, keys, and other sensitive data passed via `Preset` must not appear in run-log entries or transcripts. The transcript-log transport in ayclic offers a `TranscriptSink` trait; the Acton port needs an equivalent with a redaction filter that knows which preset values are sensitive.
7. **Error states.** TextFSMPlus has `Error` line action (terminate with explicit failure). Templates should branch to it on known device errors ("error: invalid command", "% Invalid input detected"). The runner converts a `final_state` indicating error into a step `FAILURE` with the captured error text in the run-log.

## Why this is in an ADR, not the design doc

Both round-2 reviewers (codex r2 §9.7, claude r2 §9.7) pointed out that ~110 lines of TextFSMPlus design inside a Phase 4 design doc:

- commits to a specific implementation before the Phase 5 work begins
- has unresolved platform ownership (the acton-utils extension)
- leaves several real implementation issues open (the seven above)
- distracts from Phase 4's NETCONF-only architecture

Lifting it here preserves the design intent — including the user's specific direction to anchor the CLI driver on TextFSMPlus rather than ad-hoc Expect-style state machines — without bloating the Phase 4 design doc or pre-committing more than the `DeviceOps` boundary.

## Phase 4 obligations to Phase 5

The Phase 4 design and code must:

- ✅ Ship the `DeviceOps` boundary (§9.7 of `02-sw-install-design.md`) so step signatures already accommodate CLI strategy.
- ✅ Ship per-OS `Ops` modules (e.g. `ops_sros.act`) with NETCONF strategy real and a `cli_session: ?CliSession` field. **No per-method CLI stubs raising `NotImplementedError`** — Phase 4 doesn't select the CLI strategy, so per-method stubs would be dead surface that increases test burden without delivering behavior.
- ✅ Ship the `cli_session_factory: ?proc(...)` parameter on `make_sw_install_transform`.
- Not invent a parallel CLI mechanism that Phase 5 would have to migrate away from.

Phase 5 then adds CLI strategy methods alongside the existing NETCONF ones inside the per-OS `Ops` actor; the `DeviceOps` facade signature doesn't change.

If Phase 5 picks up these handoffs cleanly, the boundary chosen in Phase 4 is correct.
