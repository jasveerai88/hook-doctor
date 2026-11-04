# Changelog

## v0.1.0 — 2026-11-04

First release.

- **One command, `audit`** — reads a single Claude Code
  settings file and statically checks every hook in it
  against an explicit rule table. Read-only: it edits
  nothing and never executes a hook command; it reads the
  settings file plus the hook scripts that file itself
  names, and existence-checks the paths the commands name.
- **Nine rules, each with its documented basis** —
  `INVALID_JSON` (a settings file that does not parse makes
  every hook *and permission rule* in it inert; the parse
  position is reported), `HOOKS_SHAPE` (structure the
  loader cannot use; an unknown event name as a warning —
  a typo there kills a whole group silently), `MATCHER_CASE`
  (a case-variant matcher like `bash` can never match the
  `Bash` tool; matchers are case-sensitive), `MATCHER_NEVER_MATCHES`
  (matcher on an event that takes none, tool name on an
  event that matches a different field, event name used as
  a tool matcher), `EXIT_CODE_TRAP` (a guard-intent hook
  with no `exit 2` and no JSON decision behind it — exit 1
  does not block; on `PermissionRequest` only the JSON
  decision counts), `EVENT_NOT_BLOCKING` (a guard on an
  event that fires after the fact or decides nothing),
  `COMMAND_PATH` (a path the command names does not exist,
  resolved against the settings folder and the home folder;
  a directly-run script that is not executable),
  `DUPLICATE_HOOK`, and `TIMEOUT_MISSING` (INFO).
- **Heuristics and version-dependent behaviour are
  labelled, never stated as certainties** — the guard-intent
  rules say "heuristic" in their own text (intent is read
  from the words guard / block / deny / prevent / forbid /
  protect in the command, the script's name or its
  content); the timeout finding notes the default is
  version-dependent; regex-flavoured matchers are
  deliberately not judged because current versions treat
  them as regexes and older ones as literal strings.
- **`--home`** — `~` and `$HOME` (and relative paths, as a
  fallback) resolve against the given home folder instead
  of the real one, so fixtures and other people's trees
  audit the way their owner would experience them.
- **`--json`** — one JSON object with the findings (rule,
  severity, event, matcher, truncated command, message,
  fix), a summary that recounts exactly against the
  findings list, and a `clean` flag; errors are
  `{"error": ...}` objects, argparse usage errors
  included. A file that exists but is broken JSON is a
  *finding* (exit 1); only a missing or unreadable file is
  an input error (exit 2).
- **Exit codes for gating** — 0 when no hook is dead as
  configured (WARN/INFO may exist), 1 on any ERROR
  finding, 2 on input error.
- **`SKILL.md`** at the repo root for AI coding agents: the
  complete rule table with bases, the event/tool lists,
  the guard-intent and blocking-signal definitions, the
  JSON schema, exit codes, and the honesty rules (findings
  are about the file; a clean audit is not a working guard;
  never edit a file to silence a finding).
- Bundled constructed fixtures (`examples/`): a broken
  settings file planting the classic faults against a fake
  home (two of its scripts deliberately missing — that
  absence is the fault), a clean settings file, and an
  invalid-JSON file with a trailing comma.
- 44 stdlib-unittest tests: both directions for every
  rule, the fixtures checked against a hand-counted
  findings table (12 findings: 5 errors, 6 warnings,
  1 info), the parse-error position, the
  `PermissionRequest` exit-code precision, the read-only
  guarantee (bytes unchanged; a marker-creating hook
  command creates nothing), and the JSON recount.
- Pure Python standard library; local only; read-only;
  MIT licence.
