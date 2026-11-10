# kdb-tick-stack

A minimal kdb+tick stack — feedhandler, tickerplant, real-time and historical
databases, gateway — fed by live public crypto market data through a Java
feedhandler.

It is built one phase at a time, each a runnable increment ending in measured
numbers and a decision record. The plan is in [ROADMAP.md](ROADMAP.md).

## Layout

| Path | Contents |
|---|---|
| `docs/specs/` | One spec per phase, written before the code |
| `docs/decisions/` | What was chosen, what was rejected, why |
| `docs/evidence/` | Timings, recovery drills, measured output |

## Licence note

kdb+ itself is licensed by KX and is not distributed here. Licence files
(`*.lic`) are git-ignored and must never be committed.
