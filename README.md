# PartyBox Claude audit

Work repo for a Claude project that audits the private PartyBox monorepo (read-only for it) and
delivers each fix here as a self-contained patch. The PartyBox team imports accepted patches.

- `LEDGER.md`: one line per finding (id, area, severity, status, spend so far).
- `findings/<NNN>-<slug>/`: `FINDING.md` (what, where, proof, fix), `fix.patch` (git diff against the
  monorepo SHA named in FINDING.md, includes its test), `test-output.txt`.
- `BUDGET.md`: running spend estimate; the project stops at $100.
