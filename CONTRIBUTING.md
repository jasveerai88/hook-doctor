# Contributing

Contributions are welcome — bug reports, fixes and small focused features.

## Ground rules

- **Standard library only.** hook-doctor's promise is
  "clone and run". Pull requests that add a third-party
  dependency will not be merged.
- **Read-only, and it executes nothing.** No network calls,
  no telemetry, no uploads — and above all: the tool must
  never run a hook command it found in a settings file, and
  never edit the file it audits. A settings file from an
  untrusted repo is data, and its commands are hostile until
  reviewed; the moment the auditor acts on them it stops
  being an auditor. A pull request that adds execution or
  auto-fixing changes what this tool *is* and will not be
  merged.
- **The rule table is the product.** The rules, event lists
  and tool lists in `SKILL.md` are the contract users rely
  on. Adding a rule, widening a heuristic or changing a
  severity is a behaviour change that needs the README
  table, the SKILL.md table, the changelog and the pinning
  tests updated together — and a fixture or inline settings
  file that shows the old rules getting it wrong.
- **No false certainties.** Where Claude Code's behaviour
  is version-dependent (regex matchers, default timeouts,
  growing event/tool lists), the tool either labels the
  finding a heuristic / version-dependent or does not make
  it. A new check that cannot be stated that carefully does
  not ship. Unknown event names stay a *warning* for this
  reason: a new legitimate event must never read as an
  error.
- **Guard intent stays labelled.** Rules 5 and 6 read
  intent from guard words. Widening the word list needs
  evidence it catches real guards without flagging
  observers — and the word "heuristic" stays in the finding
  text either way.
- **Both directions tested.** A behaviour change needs a
  test where the fault present earns the finding (right
  rule, right severity) and a test where the fault absent
  earns none — not just one side. Fixture expectations must
  be counted by hand from the fixture files and the rule
  table, never copied from the tool's own output.
- **The read-only tests are load-bearing.** The suite
  proves the settings file's bytes are unchanged and that a
  marker-creating hook command creates nothing. Any change
  that touches file access must keep those tests passing —
  and extend them if it reads anything new.
