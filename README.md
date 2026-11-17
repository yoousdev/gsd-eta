# gsd-eta

Gemessene Laufzeiten für [GSD](https://github.com/OpenGSD/gsd-core)-Subagents statt der hardcodierten Schätzung „~1–5 min; expected, not a freeze".

GSD-Workflows drucken vor jedem Subagent-Spawn eine feste ETA, die real um Faktor 6–40 daneben liegt. `gsd-eta` liest die tatsächlichen Laufzeiten aus der lokalen OpenCode-Datenbank und liefert p50/p90 pro Agent — als Tabelle, Einzelwert oder fertige Spawn-Zeile mit Startzeit und erwarteter Rückkehr.

```console
$ gsd-eta
gsd-code-fixer:           p50   6 min, p90  11 min (n=23, 30d)
gsd-code-reviewer:        p50   7 min, p90  10 min (n=77, 30d)
gsd-executor:             p50  11 min, p90  27 min (n=287, 30d)
gsd-planner:              p50   7 min, p90  17 min (n=170, 30d)
...

$ gsd-eta -s executor
Spawning executor... (läuft in einem Subagenten — keine Ausgabe, bis er zurückkehrt; Start 07:57 Uhr, gemessen p50 11 min / p90 27 min → zurück erwartet ca. 08:24 Uhr; erwartbar, kein Einfrieren)
```

## Anforderungen

- bash, `sqlite3`, `gawk` (Standard auf Fedora/Debian)
- OpenCode-Datenbank unter `~/.local/share/opencode/opencode.db` mit Subagent-Runs (`session_v2` bzw. ältere `session`-Tabelle)

## Installation

```bash
git clone https://github.com/yoousdev/gsd-eta.git
mkdir -p ~/.local/bin
cp gsd-eta/gsd-eta ~/.local/bin/
chmod +x ~/.local/bin/gsd-eta
```

Danach `~/.local/bin` im `PATH` sicherstellen (bei den meisten Distro-Defaults vorhanden).

Optional, damit GSD-Workflows die gemessenen Werte statt „~1–5 min" drucken — in die `AGENTS.md` des OpenCode-Config-Roots:

```markdown
## GSD-Subagent-Spawn-ETA: `gsd-eta` statt „~1–5 min"

Wenn ein GSD-Workflow vor einem Subagent-Spawn eine ETA verlangt („no output
until it returns, ~1–5 min; expected, not a freeze"): zuerst `gsd-eta -s <agent>`
ausführen und die fertige Zeile drucken (enthält aktuelle Startzeit, gemessene
p50–p90-Werte und die erwartete Rückkehr-Uhrzeit) statt der hardcodierten
1–5 min. DB leer oder nicht erreichbar (Exit ≠ 0) → hardcoded `~1–5 min`
behalten. `gsd-eta` liest nur die OpenCode-DB — nie gsd-core patchen.
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
- **Fenster:** pro Agent die letzten 30 Tage, wenn dort ≥ 10 Runs liegen, sonst All-time (Spalte `30d`/`alle`). Alte Modell-Ären altern raus, die Werte werden mit jedem neuen Run genauer.
- **Filter:** Runs unter 30 s gelten als Crash-Stubs und fallen raus; Ausreisser werden bei 3 h gedeckelt; Perzentile per nearest-rank (`gawk` `asort`).
- **Keine Abhängigkeiten** außer bash/sqlite3/gawk, kein Daemon, kein Cache — jede Abfrage ist on-demand.

## Warum keine Skalierung nach Task-Anzahl?

Geprüft: Executor-Dauer vs. Task-Zahl korreliert mit r = 0,12 (n = 119) — die Task-Anzahl prädiktiert die Dauer nicht (Komplexität zählt, nicht Anzahl). Deshalb flat p50/p90.

## Lizenz

MIT
