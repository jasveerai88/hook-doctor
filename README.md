# hook-doctor

**Your safety guard was never running. This finds out
before you rely on it.**

Agent hooks fail silently. A guard that never fired and a
guard that was never configured look identical from inside a
session — successful hooks are quiet, failed ones are
non-blocking, and the action you meant to stop goes ahead
anyway. The causes are mostly static: a matcher in the wrong
case (`bash` never matches the `Bash` tool), a guard script
that exits 1 believing it blocks (only exit 2 blocks), a
guard wired to an event that fires *after* the call it was
meant to stop, a settings file whose JSON broke months ago
and quietly took every hook in it down.

`hook-doctor` audits **your own** Claude Code settings file
against an explicit rule table and reports every hook that
can never fire or can never block — with the rule, the
location, and the one-line fix. It is read-only: it edits
nothing and **never executes a hook command**. (Sister tools:
`secret-guard` scans files for leaked secrets; `clone-guard`
triages *foreign* repos. This one checks your own config
for correctness.)

- Pure Python 3 standard library — no dependencies, no install step
- Local only — reads the one settings file you point it at,
  plus the hook scripts that file itself names
- No runtime needed: the check runs before the session, not
  after the incident (`/hooks` shows what loaded; `--debug`
  tells you afterwards)
- Script-friendly exit codes: `0` = no dead hooks, `1` = at
  least one hook is dead as configured, `2` = error

## The rules

Every finding names its rule. Severities: **ERROR** = the
hook can never work as configured; **WARN** = it probably
does not do what you intend; **INFO** = worth knowing.

| Rule | Severity | What it catches |
|---|---|---|
| `INVALID_JSON` | ERROR | The settings file is not valid JSON (position reported), or its top level is not an object — Claude Code loads *nothing* from it: every hook and permission rule in the file is inert |
| `HOOKS_SHAPE` | ERROR / WARN | `hooks` not an object, an event value not a list, a group with no `hooks` list, a command handler with no command (ERROR) — or an event name outside the documented set, which is how a typo silently kills a whole group (WARN) |
| `MATCHER_CASE` | ERROR | A matcher alternative that is a case-variant of a documented tool name (`bash` for `Bash`). Matchers are case-sensitive, so it can never match the tool it names |
| `MATCHER_NEVER_MATCHES` | WARN | A matcher on an event that takes none (`Stop`, `UserPromptSubmit`, …) — silently ignored, the hook runs unfiltered; a tool name as matcher on an event that matches a different field (`SessionStart` matches the session *source*); a matcher that names an event instead of a tool |
| `EXIT_CODE_TRAP` | WARN | A guard-intent hook (its command or script says guard / block / deny / prevent / forbid / protect) on a blocking event, with no blocking signal anywhere: no `exit 2`, no JSON decision. Exit 1 is a non-blocking error — the call proceeds. On `PermissionRequest` only a JSON decision counts |
| `EVENT_NOT_BLOCKING` | WARN | A guard-intent hook on an event that cannot block (`PostToolUse`, `Notification`, `SessionStart`, `SessionEnd`, …) — it fires after the fact, or decides nothing |
| `COMMAND_PATH` | ERROR / WARN | A path the command itself names does not exist (ERROR — resolved against the settings folder and the home folder), or a script run directly exists but is not executable (WARN) |
| `DUPLICATE_HOOK` | WARN | The identical handler registered twice for the same event + matcher — it runs once per registration |
| `TIMEOUT_MISSING` | INFO | A handler with no `timeout` — if it hangs, the (version-dependent) default applies |

Where a behaviour is version-dependent or a check is a
heuristic, the finding **says so** (the exit-code trap is
labelled a heuristic in its own text). The full table — with
the documented basis for every rule, the exact event lists,
the guard-word list and the blocking signals — lives in
[SKILL.md](SKILL.md).

## Usage

```bash
# Audit your user settings
python3 src/hook_doctor.py audit ~/.claude/settings.json

# Audit a project settings file, resolving ~ and $HOME
# against a specific home folder
python3 src/hook_doctor.py audit .claude/settings.json \
    --home /home/you

# One JSON object for scripts, agents and CI
python3 src/hook_doctor.py audit .claude/settings.json --json

# Gate on it: exit 1 means a hook is dead as configured
python3 src/hook_doctor.py audit .claude/settings.json \
    || echo "dead hooks — fix before relying on them"
```

Settings layers (`~/.claude/settings.json`, a project's
`.claude/settings.json`, `.claude/settings.local.json`, a
plugin's `hooks/hooks.json`) are audited one file per run —
point the command at each layer you rely on.

## The fixtures

`examples/` holds constructed settings files (fictional, and
labelled as such). `broken-settings.json` plants the classic
faults; its scripts live under `examples/fake-home/` — pass
`--home examples/fake-home` so `$HOME` resolves there. Two
scripts are **deliberately missing** (`guard-rm.sh` and
`scripts/notify.sh`): that absence is the fault being
demonstrated. `clean-settings.json` is the same shape done
right. `invalid-settings.json` has a trailing comma.

## Real runs (verbatim)

The broken fixture — five dead hooks and six misfires,
worked out by hand from the rule table before the tool was
run:

```text
$ python3 src/hook_doctor.py audit examples/broken-settings.json --home examples/fake-home
hook-doctor v0.1.0 — audit · settings broken-settings.json
8 hook handlers across 5 events · home for ~ and $HOME paths: examples/fake-home
Read-only: it edits nothing and never runs a hook command. It reads the settings file, plus the scripts that file itself names.

ERRORS — hooks that can never work as configured (5):
[COMMAND_PATH] Notification · ./scripts/notify.sh
  the command path ./scripts/notify.sh does not exist (resolved against the settings folder and the home folder) — this hook can never run
  fix: create the script at that path, or point the command at a script that exists
[COMMAND_PATH] PreToolUse · matcher "bash" · "$HOME/.claude/hooks/guard-rm.sh"
  the command path $HOME/.claude/hooks/guard-rm.sh does not exist (resolved against the settings folder and the home folder) — this hook can never run
  fix: create the script at that path, or point the command at a script that exists
[MATCHER_CASE] PreToolUse · matcher "Write|edit" · python3 "$HOME/.claude/hooks/block-secrets.py"
  matcher "edit" can never match the Edit tool — matchers are case-sensitive and tool names are exact
  fix: change the matcher to "Edit"
[MATCHER_CASE] PreToolUse · matcher "bash" · "$HOME/.claude/hooks/guard-rm.sh"
  matcher "bash" can never match the Bash tool — matchers are case-sensitive and tool names are exact
  fix: change the matcher to "Bash"
[COMMAND_PATH] SessionStart · /opt/hooks/start.sh
  the command path /opt/hooks/start.sh does not exist (resolved against the settings folder and the home folder) — this hook can never run
  fix: create the script at that path, or point the command at a script that exists

WARNINGS — hooks that probably do not do what you intend (6):
[DUPLICATE_HOOK] PostToolUse · matcher "Bash" · "$HOME/.claude/hooks/guard-format.sh"
  this exact handler is registered more than once for PostToolUse — it will run once per registration
  fix: remove the duplicate registration
[EVENT_NOT_BLOCKING] PostToolUse · matcher "Bash" · "$HOME/.claude/hooks/guard-format.sh"
  guard-intent hook on PostToolUse: this event cannot block the action it guards — it fires after the fact or decides nothing, so the guard watches the thing happen
  fix: move the guard to PreToolUse, the tool event that can refuse a call — or drop the blocking intent and keep this as an observer
[EVENT_NOT_BLOCKING] PostToolUse · matcher "Bash" · "$HOME/.claude/hooks/guard-format.sh"
  guard-intent hook on PostToolUse: this event cannot block the action it guards — it fires after the fact or decides nothing, so the guard watches the thing happen
  fix: move the guard to PreToolUse, the tool event that can refuse a call — or drop the blocking intent and keep this as an observer
[EXIT_CODE_TRAP] PreToolUse · matcher "Write|edit" · python3 "$HOME/.claude/hooks/block-secrets.py"
  guard-intent hook on PreToolUse shows no way to block: no exit 2 and no JSON decision in the command or the script it runs — exit 1 (and every other code) is a non-blocking error and the action proceeds. Heuristic: the intent was read from guard words (guard / block / deny / prevent / forbid / protect) in the command or script
  fix: exit 2 with the reason on stderr, or print a JSON decision (permissionDecision: deny on PreToolUse) when the guard should stop the action
[EXIT_CODE_TRAP] PreToolUse · matcher "bash" · "$HOME/.claude/hooks/guard-rm.sh"
  guard-intent hook on PreToolUse shows no way to block: no exit 2 and no JSON decision in the command or the script it runs — exit 1 (and every other code) is a non-blocking error and the action proceeds. Heuristic: the intent was read from guard words (guard / block / deny / prevent / forbid / protect) in the command or script
  fix: exit 2 with the reason on stderr, or print a JSON decision (permissionDecision: deny on PreToolUse) when the guard should stop the action
[MATCHER_NEVER_MATCHES] Stop · matcher "Bash" · echo stopped
  Stop hooks do not take a matcher — this one is silently ignored and the hook runs on every Stop
  fix: remove the matcher, or move the hook to an event that filters (tool events match on the tool name)

INFO — worth knowing (1):
[TIMEOUT_MISSING] PreToolUse · matcher "Write|edit" · python3 "$HOME/.claude/hooks/block-secrets.py"
  a handler under PreToolUse sets no timeout — if it hangs, the (version-dependent) default timeout applies before anything else can proceed
  fix: set an explicit "timeout" in seconds on the handler

Summary: 12 findings · 5 errors · 6 warnings · 1 info
Verdict: FAIL — 5 error(s): at least one hook is dead as configured. Fix the errors before relying on these hooks.
(exit code: 1)
```

The clean fixture — a guard that exits 2, scripts that
exist, exact matchers, timeouts everywhere:

```text
$ python3 src/hook_doctor.py audit examples/clean-settings.json --home examples/fake-home
hook-doctor v0.1.0 — audit · settings clean-settings.json
4 hook handlers across 3 events · home for ~ and $HOME paths: examples/fake-home
Read-only: it edits nothing and never runs a hook command. It reads the settings file, plus the scripts that file itself names.

No findings — every hook passes the static rules.
Summary: 0 findings · 0 errors · 0 warnings · 0 info
Verdict: CLEAN — no dead hooks or misfires found by the static rules.
(exit code: 0)
```

And the invalid fixture — one trailing comma takes down
every hook in the file, and the finding says exactly where:

```text
$ python3 src/hook_doctor.py audit examples/invalid-settings.json --home examples/fake-home
hook-doctor v0.1.0 — audit · settings invalid-settings.json
The settings file does not parse — no hook in it can load.
Read-only: it edits nothing and never runs a hook command. It reads the settings file, plus the scripts that file itself names.

ERRORS — hooks that can never work as configured (1):
[INVALID_JSON] settings file
  the file is not valid JSON — line 13, column 7: Expecting property name enclosed in double quotes. Claude Code loads nothing from a file it cannot parse: every hook and permission rule in it is inert
  fix: fix the JSON at the reported position (a trailing comma and unquoted keys are the usual causes)

Summary: 1 findings · 1 errors · 0 warnings · 0 info
Verdict: FAIL — 1 error(s): at least one hook is dead as configured. Fix the errors before relying on these hooks.
(exit code: 1)
```

## Use with AI agents

`audit --json` prints one JSON object: the findings (rule,
severity, event, matcher, truncated command, message, fix),
a summary that recounts exactly against the findings list,
and a `clean` flag that is true iff there are no ERRORs.
Errors are `{"error": ...}` with exit code 2, argparse usage
errors included. Agents: read [SKILL.md](SKILL.md) first —
the complete rule table with bases, the event lists, the
JSON schema, and the honesty rules (a clean audit is a
statement about the file against these rules, never a
promise that a guard's logic is right; and never edit a
settings file to silence a finding without saying so).

## Tests

```bash
python3 -m unittest discover -s tests -v
# Ran 44 tests — OK
```

The suite covers both directions for every rule: the fault
present earns the finding with the right rule and severity;
the fault absent earns none. The bundled fixtures are
checked against a hand-counted findings table (12 findings:
5 errors, 6 warnings, 1 info). It also pins the parse-error
position, the `PermissionRequest` precision (exit 2 is not a
decision there), the read-only guarantee — the settings
file's bytes unchanged, and a hook command that *would*
create a marker file if executed creates nothing — and the
JSON summary recounting exactly against the findings list.

## Who this is for

Anyone whose hooks carry weight: permission guards, secret
blockers, stop-checks, audit trails — in Claude Code or a
fleet built on it. If a hook of yours has ever "just not
fired" and nobody could say why, this is the check that
would have said why, from the file alone, before the
session. If you need to know what *actually ran* in a live
session, that is the runtime's `/hooks` and `--debug` — this
tool is the static audit that runs first.

## Internal use

This tool does a real job for its author: the author's
automation stack runs on hooks — event automations, watchers,
guards — and a guard that silently never fired is precisely
the failure the whole oversight layer exists to catch. The
auditor runs over the engine's own settings layers as a
standing check; its fixtures encode the failure list the
debugging guides keep rediscovering.

## Known limits

- **One file per run.** Layers are audited individually;
  there is no merged cross-layer view yet. (Duplicate
  detection is therefore within one file.)
- **Regex matchers are not judged.** A matcher made only of
  plain characters is an exact name or a `|`-list of them;
  any other character makes it a regex in current versions
  and a literal string in older ones. hook-doctor takes no
  side and flags neither — check regex matchers against
  your version's docs.
- **`$CLAUDE_PROJECT_DIR` paths are skipped.** They cannot
  be resolved statically without knowing the project root,
  so the tool does not guess at them.
- **Unknown matcher names are left alone.** On a tool
  event, a plain name that is not a documented tool may be
  an MCP tool (`mcp__…`) or a tool from a newer version.
  Only case-variants of documented names are flagged —
  those are wrong under every reading.
- **Guard intent is a heuristic.** It is read from guard
  words in the command, the script's name and the script's
  content. A guard named like a poem will be missed; an
  observer named like a guard will be flagged. The finding
  labels itself a heuristic so a human can overrule it in
  seconds.
- **Command strings are scanned token-wise** (shell quotes
  honoured). A path glued to an operator without spaces
  (`cd /x&&./run.sh`) can hide from the path check.
- **Only command handlers get command checks.** `prompt`,
  `http` and `agent` handlers are counted and
  timeout-checked but their payloads are not audited.
- **A clean audit is not a working guard.** The rules prove
  a hook *can* fire and *can* block. Whether its logic
  checks the right thing is yours to test.

## Privacy

What it touches: only the settings file you point it at,
and the script files that settings file itself names (read,
never run). What it sends: nothing — there is no network
code in this tool. What it changes: nothing — the auditor
is read-only, and a test proves the settings file's bytes
are untouched.

## Support

If this finds one dead guard before it costs you an
incident, consider sponsoring:
**https://github.com/sponsors/jasveerai88**

## Licence

MIT — see [LICENSE](LICENSE).
