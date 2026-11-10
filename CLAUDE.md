# CLAUDE.md

A minimal kdb+tick stack fed by live crypto market data through a Java
feedhandler. `ROADMAP.md` drives everything; read the phase before touching
its code.

## Keep it publishable

The repo starts private but is written so it can be made public at any time.

- Nothing here references a private repo, an employer, a job description, a
  recruiter, or a compensation figure. If a rationale only makes sense by
  naming one of those, rewrite it as an engineering rationale or drop it.
- Market data comes from free public feeds only; no licensed vendor data.
- Never commit a kdb+ licence file (`*.lic`), tp logs or HDB data.

## Language

Reply in Traditional Chinese, keeping technical terms in English. Commit
messages and code comments are English.

## Working rhythm

Per phase: spec in `docs/specs/` → code → numbers in `docs/evidence/` →
decision record in `docs/decisions/`. The decision record is written by the
owner, not drafted by the assistant; the assistant may review and challenge it.
One logical change per commit.
