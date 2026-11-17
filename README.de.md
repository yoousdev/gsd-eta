# gsd-eta

Gemessene Laufzeiten für [GSD](https://github.com/OpenGSD/gsd-core)-Subagents statt der hardcodierten Schätzung „~1–5 min; expected, not a freeze".

GSD-Workflows drucken vor jedem Subagent-Spawn eine feste ETA, die real um Faktor 6–40 daneben liegt. `gsd-eta` liest die tatsächlichen Laufzeiten aus der lokalen OpenCode-Datenbank und liefert p50/p90 pro Agent — als Tabelle, Einzelwert oder fertige Spawn-Zeile mit Startzeit und erwarteter Rückkehr.

**English:** [README.md](README.md)

```console
$ gsd-eta
gsd-code-fixer:           p50   6 min, p90  11 min (n=23, 30d)
gsd-executor:             p50  11 min, p90  26 min (n=291, 30d)
...

$ gsd-eta -s executor
Spawning executor... (runs in a subagent — no output until it returns; started 08:56, measured p50 11 min / p90 26 min → expected back around 09:22; expected, not a freeze)
```

Der Agent übersetzt die Spawn-Zeile beim Drucken in die Sprache des Users (z. B. Deutsch: „Start 08:56 Uhr, gemessen p50 11 min / p90 26 min → zurück erwartet ca. 09:22 Uhr").

## Voraussetzungen

- bash, `sqlite3`, `gawk` (Standard auf Fedora/Debian)
- OpenCode-Datenbank unter `~/.local/share/opencode/opencode.db` mit Subagent-Runs (`session_v2` bzw. ältere `session`-Tabelle)

## Installation

```bash
git clone https://github.com/yoousdev/gsd-eta.git
mkdir -p ~/.local/bin
cp gsd-eta/gsd-eta ~/.local/bin/
chmod +x ~/.local/bin/gsd-eta
```

`~/.local/bin` muss im `PATH` liegen (bei den meisten Distro-Defaults vorhanden).

Optional, damit GSD-Workflows die gemessenen Werte statt „~1–5 min" drucken — in die `AGENTS.md` des OpenCode-Config-Roots (englischer Wortlaut im [englischen README](README.md#installation)):

```markdown
When a GSD workflow demands an ETA before a subagent spawn: first run
`gsd-eta -s <agent>` and print the ready-made line instead of the
hardcoded 1–5 min — translated into the user's language. If the DB is
empty or unreachable (exit ≠ 0) → keep the hardcoded `~1–5 min`.
`gsd-eta` only reads the OpenCode DB — never patch gsd-core.
```

Kein Patch von gsd-core — das Skript ist rein lesend und überlebt `/gsd-update`.

## Benutzung

| Befehl | Output |
|---|---|
| `gsd-eta` | Tabelle aller `gsd-*`-Agenten (n, p50, p90) |
| `gsd-eta <agent>` | eine Zeile, z. B. `gsd-eta executor` |
| `gsd-eta -s <agent>` | fertige Spawn-Zeile mit Startzeit + erwarteter Rückkehr (p90) |
| `gsd-eta nonsense` | Fehler auf stderr, Exit 1 → Fallback auf die Workflow-Default-ETA |

Kurznamen: `executor`, `researcher`, `planner`, `checker`, `fixer`, `mapper`. Ohne `gsd-`-Präfix wird automatisch ergänzt (`verifier` → `gsd-verifier`).

## Wie es rechnet

- **Quelle:** `session_v2` (lebend) `UNION ALL` `session` (Archiv) aus `~/.local/share/opencode/opencode.db`, nur-Lesend via `?immutable=1` — die DB wird nicht gelockt und nicht kopiert.
- **Fenster:** pro Agent die letzten 30 Tage, wenn dort ≥ 10 Runs liegen, sonst All-time (Spalte `30d`/`all`). Alte Modell-Ären altern raus, die Werte werden mit jedem neuen Run genauer.
- **Filter:** Runs unter 30 s gelten als Crash-Stubs und fallen raus; Ausreisser werden bei 3 h gedeckelt; Perzentile per nearest-rank (`gawk` `asort`).
- **Keine Abhängigkeiten** außer bash/sqlite3/gawk, kein Daemon, kein Cache — jede Abfrage ist on-demand.

## Warum keine Skalierung nach Task-Anzahl?

Geprüft: Executor-Dauer vs. Task-Zahl korreliert mit r = 0,12 (n = 119) — die Task-Anzahl prädiktiert die Dauer nicht (Komplexität zählt, nicht Anzahl). Deshalb flat p50/p90.

## Lizenz

MIT
