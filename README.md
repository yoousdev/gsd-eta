# gsd-eta

Measured runtimes for [GSD](https://github.com/OpenGSD/gsd-core) subagents instead of the hardcoded "~1–5 min; expected, not a freeze" estimate.

GSD workflows print a fixed ETA before every subagent spawn that is off by a factor of 6–40 in practice. `gsd-eta` reads the actual runtimes from the local OpenCode database and returns p50/p90 per agent — as a table, a single value, or a ready-made spawn line with start time and expected return.

**Deutsch:** [README.de.md](README.de.md)

```console
$ gsd-eta
gsd-code-fixer:           p50   6 min, p90  11 min (n=23, 30d)
gsd-code-reviewer:        p50   7 min, p90  10 min (n=80, 30d)
gsd-executor:             p50  11 min, p90  26 min (n=291, 30d)
gsd-planner:              p50   7 min, p90  17 min (n=170, 30d)
...

$ gsd-eta -s executor
Spawning executor... (runs in a subagent — no output until it returns; started 08:56, measured p50 11 min / p90 26 min → expected back around 09:22; expected, not a freeze)
```

## Requirements

- bash, `sqlite3`, `gawk` (default on Fedora/Debian)
- OpenCode database at `~/.local/share/opencode/opencode.db` containing subagent runs (`session_v2` and/or the older `session` table)

## Installation

```bash
git clone https://github.com/yoousdev/gsd-eta.git
mkdir -p ~/.local/bin
cp gsd-eta/gsd-eta ~/.local/bin/
chmod +x ~/.local/bin/gsd-eta
```

Make sure `~/.local/bin` is in your `PATH` (most distro defaults include it).

Optional, so GSD workflows print measured values instead of "~1–5 min" — add to the `AGENTS.md` in your OpenCode config root:

```markdown
## GSD subagent spawn ETA: `gsd-eta` instead of "~1–5 min"

When a GSD workflow demands an ETA before a subagent spawn ("no output
until it returns, ~1–5 min; expected, not a freeze"): first run
`gsd-eta -s <agent>` and print the ready-made line (contains the current
start time, measured p50–p90 values, and the expected return time) instead
of the hardcoded 1–5 min — translate it into the user's language when
printing. If the DB is empty or unreachable (exit ≠ 0) → keep the hardcoded
`~1–5 min`. `gsd-eta` only reads the OpenCode DB — never patch gsd-core.
```

No patching of gsd-core — the script is strictly read-only and survives `/gsd-update`.

## Usage

| Command | Output |
|---|---|
| `gsd-eta` | table of all `gsd-*` agents (n, p50, p90) |
| `gsd-eta <agent>` | single line, e.g. `gsd-eta executor` |
| `gsd-eta -s <agent>` | ready-made spawn line with start time + expected return (p90) |
| `gsd-eta nonsense` | error on stderr, exit 1 → fall back to the workflow's default ETA |

Short names: `executor`, `researcher`, `planner`, `checker`, `fixer`, `mapper`. A missing `gsd-` prefix is added automatically (`verifier` → `gsd-verifier`).

## How it works

- **Source:** `session_v2` (live) `UNION ALL` `session` (archive) from `~/.local/share/opencode/opencode.db`, read-only via `?immutable=1` — the DB is never locked or copied.
- **Window:** per agent, the last 30 days when they contain ≥ 10 runs, otherwise all-time (column `30d`/`all`). Old model eras age out; values get more accurate with every new run.
- **Filters:** runs under 30 s count as crash stubs and are dropped; outliers are capped at 3 h; percentiles via nearest rank (`gawk` `asort`).
- **No dependencies** beyond bash/sqlite3/gawk, no daemon, no cache — every query runs on demand.

## Why no scaling by task count?

Checked: executor duration vs. task count correlates at r = 0.12 (n = 119) — task count does not predict duration (complexity matters, not count). That's why it stays flat p50/p90.

## License

MIT
