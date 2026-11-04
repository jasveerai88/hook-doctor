---
name: hook-doctor
description: Statically audit one Claude Code settings file's hooks against an explicit rule table — matchers that can never fire (wrong case, wrong event, wrong field), guards that can never block (exit-1 trap, non-blocking events), hook commands pointing at missing or non-executable files, invalid JSON that silently kills every hook in the file, duplicate handlers and missing timeouts. Use when setting up or reviewing agent hooks, before relying on a guard, or when a hook "just doesn't fire". Read-only — it edits nothing and never executes a hook command. Exit 1 means at least one hook is dead as configured.
---

# hook-doctor — agent skill

## Purpose

Hooks fail silently: a guard that never ran and a guard
that was never configured look identical from inside a
session. The causes are mostly static — readable off the
settings JSON and the scripts it names, without running
anything. This tool applies an explicit rule table to one
settings file and reports every hook that can never fire or
can never block, with the rule, the location and a one-line
fix. It is read-only and executes nothing: it reads the
settings file named on the command line, plus the hook
scripts that file itself names (to judge their intent), and
checks that the paths the commands name exist.

**The honesty core of this tool:** a finding is a statement
about *the file*, judged against the documented hook
semantics encoded below — not a statement that the hook was
run. A clean audit means no hook is dead or misfiring *by
these rules*; it says nothing about whether a guard's logic
is correct. Never relay a clean audit as "the hooks work",
and never edit a settings file to silence a finding without
saying exactly that in your report.

## When to use / not use

Use when:
- You are about to rely on hooks — permission guards,
  secret blockers, stop-checks — and want the dead ones
  found first.
- A hook "isn't firing" and you want the static causes
  eliminated before reaching for `--debug`.
- Gating a setup in CI or a review script: `audit` exit 1
  = at least one ERROR finding.

Do NOT use when:
- You need to know what actually ran in a live session.
  That is the runtime's `/hooks` / `--debug`; this is the
  static audit that runs before.
- The config belongs to a different agent product. The
  rule table encodes Claude Code's documented hook
  semantics; other products' hooks differ.
- You want the file fixed. The auditor never edits. Apply
  the printed fixes yourself, deliberately.

## Commands

```bash
# Audit one settings file (text view)
python3 src/hook_doctor.py audit SETTINGS.json

# Resolve ~ and $HOME paths against a specific home folder
python3 src/hook_doctor.py audit SETTINGS.json --home DIR

# One JSON object, same findings and exit codes
python3 src/hook_doctor.py audit SETTINGS.json --json
```

Layers (`~/.claude/settings.json`, project
`.claude/settings.json`, `.claude/settings.local.json`,
plugin `hooks/hooks.json`) are audited one file per run.
There are no other commands and no configuration file.
`--home` defaults to the real home folder; it exists so
audits of someone else's tree (or of fixtures) resolve `~`
and `$HOME` the way that tree's owner would experience
them.

## The rule table (complete)

Severity: **ERROR** — the hook can never work as
configured; the audit exits 1. **WARN** — the hook probably
does not do what its author intends; exit stays 0. **INFO**
— worth knowing; never fails the audit.

| # | Rule | Severity | Basis (documented semantics) |
|---|---|---|---|
| 1 | `INVALID_JSON` | ERROR | The file fails to parse (position reported: line, column, parser message), or its top level is not an object. A settings file Claude Code cannot parse contributes nothing — every hook *and permission rule in it* is inert. |
| 2 | `HOOKS_SHAPE` | ERROR / WARN | ERROR: `hooks` not an object keyed by event; an event's value not a list; a group not an object or without a `hooks` list; a matcher that is not a string; a `command` handler with no command. WARN: an event name outside the documented event set below — if it is a typo, the group never fires. (A genuinely new event in a newer version also lands here as a warning to verify, never an error.) |
| 3 | `MATCHER_CASE` | ERROR | Matchers are **case-sensitive** and tool names are exact. On a tool event, an alternative that differs from a documented tool name only by case (`bash`, `EDIT`) can never match the tool it names. Only case-variants are flagged — an unknown plain name may be an MCP tool or a newer tool and is left alone. |
| 4 | `MATCHER_NEVER_MATCHES` | WARN | Three shapes: (a) a matcher on an event that takes none — it is silently ignored and the hook runs on **every** such event; (b) a tool name (exact or case-variant) used as the matcher on an event that matches a different field (see the event table) — that value can never occur in that field; (c) on a tool event, a matcher alternative that names a known **event** instead of a tool. |
| 5 | `EXIT_CODE_TRAP` | WARN (heuristic) | On a blocking event, a guard-intent hook with **no blocking signal** in its command or in the script it runs: no `exit 2`, no JSON decision. Documented: only exit 2 blocks; exit 1 and every other code are non-blocking errors and the action proceeds. On `PermissionRequest`, exit codes alone never deny — only a JSON decision counts, so there the signal is the decision alone. |
| 6 | `EVENT_NOT_BLOCKING` | WARN | A guard-intent hook on an event that cannot block (see table): `PostToolUse`/`PostToolUseFailure` fire after the call already ran; `Notification`, `SessionStart`, `SessionEnd` decide nothing. The guard can only watch. |
| 7 | `COMMAND_PATH` | ERROR / WARN | ERROR: a path the command itself names does not exist. `~`/`$HOME` resolve against the home folder (see `--home`); relative paths against the settings file's folder, then the home folder. WARN: a script the command runs **directly** (first token, or first after a shell operator) exists but is not executable. Scripts run through an interpreter (`python3 x.py`) get no executability finding. Paths behind `$CLAUDE_PROJECT_DIR` and globs are skipped, never guessed at. |
| 8 | `DUPLICATE_HOOK` | WARN | The same event + matcher + command registered more than once in the file: it runs once per registration. (Within one file only — layers are audited separately.) |
| 9 | `TIMEOUT_MISSING` | INFO | A handler with no `timeout` key. If it hangs, the default timeout applies — and the default is version-dependent, which is itself the point: set it explicitly. |

### The event table the rules use

| Class | Events | Meaning for the rules |
|---|---|---|
| Tool-matcher events | `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest` | The matcher filters on the tool name → `MATCHER_CASE` / 4(c) apply |
| No-matcher events | `UserPromptSubmit`, `Stop`, `PostToolBatch`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `WorktreeCreate`, `WorktreeRemove`, `MessageDisplay`, `CwdChanged` | A matcher is ignored → rule 4(a) |
| Other-field matcher events | `SessionStart` (session source: startup, resume, clear, compact), `PreCompact` (trigger: manual, auto), `Notification` (notification type), `SubagentStart`, `SubagentStop` (subagent name) | A tool name there can never match → rule 4(b) |
| Blocking events | `PreToolUse`, `UserPromptSubmit`, `Stop`, `SubagentStop`, `PermissionRequest` | Can refuse the action → rule 5 applies |
| Non-blocking events | `PostToolUse`, `PostToolUseFailure`, `Notification`, `SessionStart`, `SessionEnd` | Cannot refuse anything → rule 6 applies |

Event names beyond these lists that the tool recognises as
documented (`Setup`, `SessionEnd`, `StopFailure`,
`PermissionDenied`, `FileChanged`, `ConfigChange`,
`InstructionsLoaded`, `Elicitation`, `ElicitationResult`)
get structure, path, duplicate and timeout checks but no
matcher/blocking judgement — the tool does not invent
semantics it does not know. Anything else is rule 2's
unknown-event warning.

### Documented tool names (for rule 3)

`AskUserQuestion`, `Bash`, `BashOutput`, `Edit`,
`EnterPlanMode`, `ExitPlanMode`, `Glob`, `Grep`,
`KillShell`, `LS`, `MultiEdit`, `NotebookEdit`, `Read`,
`Skill`, `Task`, `TodoWrite`, `WebFetch`, `WebSearch`,
`Write` — labelled in the code as the *commonly documented*
names, because the list grows between versions; that is
exactly why only case-variants (wrong under every reading)
are errors.

### Guard intent and blocking signals (for rules 5–6)

Guard intent is a **heuristic**, labelled as one in every
finding it produces: the command string, the referenced
script's file name, or the script's content contains one of
the words **guard, block, deny, prevent, forbid, protect**
(as a whole word or name segment — `guard-rm.sh`,
`block_secrets.py` count; `guards` in prose and `blockade`
do not). The script is read only when the command names it
and it exists; at most its first 64 KB.

A blocking signal is any of: `exit 2` / `exit(2)` /
`sys.exit(2)` in the command or script; `permissionDecision`
in the command or script; a `decision … block` JSON shape;
`hookSpecificOutput`. On `PermissionRequest`, only the
JSON-decision signals count.

Matchers containing regex characters (`. ^ $ + ? ( ) [ ]
{ } \` or `*`) are **not judged**: current Claude Code
versions treat such matchers as regexes, older ones as
literal strings, so no finding about them could be stated
without picking a version. Alternatives starting `mcp__`
are likewise left alone.

## Output JSON schema (`--json`)

One object: `tool` (`"hook-doctor"`), `version`, `command`
(`"audit"`), `settings` (file name only), `home` (the home
folder paths resolved against), `hooks` (handler count),
`events` (event names present, sorted), `parse_error`
(bool), `findings`, `summary`, `clean`.

Each finding: `rule`, `severity` (`ERROR` / `WARN` /
`INFO`), `event` (or null), `matcher` (or null), `command`
(the handler's command, truncated to 120 characters, or
null), `message` (what is wrong, with the basis where it is
not obvious), `fix` (the one-line remedy).

`summary`: `{findings, errors, warnings, info}` —
guaranteed to recount exactly against the `findings` list.
`clean` is true iff `errors` is 0. Findings are sorted by
severity (ERROR, WARN, INFO), then event, rule, matcher,
command — the order is stable across runs.

On error, stdout is a single object: `{"error": "<message>"}`
with exit code 2 — including argparse usage errors in
`--json` mode. Never plain text, never a traceback.

## Exit codes

- `0` — no ERROR findings. WARN/INFO findings may exist;
  read them, they are the tool working.
- `1` — at least one ERROR finding: a hook is dead as
  configured. Surface the errors, with their fixes, to the
  human.
- `2` — ERROR (input): the settings file is missing or
  unreadable. Note the distinction this tool draws: a file
  that exists but is **broken JSON is a finding (exit 1)**,
  not an input error — the broken file is precisely the
  defect being reported. No findings exist for a failed
  (exit 2) run; do not quote counts from one.

## Limits & honesty rules the agent must respect

- **Findings are about the file, not a run.** Report an
  ERROR as "this hook can never fire as configured, by rule
  X" — never as "this hook did not fire in your session"
  (you did not observe the session).
- **A clean audit is not a working guard.** The rules prove
  a hook *can* fire and *can* block. Its logic — what it
  checks, what it lets through — is untested by this tool.
  Say so when you relay a clean result.
- **Never edit the settings file to silence a finding and
  report the audit as passed.** Fix the underlying defect
  (the path, the case, the event), or report the finding as
  accepted-with-reason. Editing to silence is destroying
  the evidence the tool exists to surface.
- **The heuristic is labelled; keep the label.** When you
  relay an `EXIT_CODE_TRAP` finding, keep the word
  "heuristic" attached. A guard word in a script's comments
  can misread intent both ways; the human overrules.
- **Version drift is real.** Matcher-regex behaviour,
  default timeouts and the event/tool lists move between
  Claude Code versions. Where a finding's basis is
  version-dependent, the finding text says so — quote that
  caveat rather than dropping it.
- **Read-only is the product.** The auditor never executes
  a hook command and never edits the settings file; a test
  proves both. Do not wrap it in a way that runs what it
  reads — a settings file from an untrusted repo is data,
  and its hook commands are hostile until reviewed.
