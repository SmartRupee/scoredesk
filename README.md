# ScoreDesk

A single-file, client-side credit report reader. Upload a CIBIL/TransUnion, Experian, or CRIF High Mark ("Goodscore") PDF and it parses the score, accounts, and enquiries entirely inside your own browser — nothing is uploaded to a server — then shows a risk dashboard, a plain-English Credit Health Review, an instant search over the parsed accounts, and an Excel export.

## Running it

No build step. Open `index.html` directly in a browser, or serve the folder with any static file server (e.g. `python3 -m http.server`) and visit it.

## Supported report layouts

- TransUnion CIBIL — several export/print layouts (member portal PDF, "CreditView" self-serve dashboard, etc.)
- Experian
- CRIF High Mark ("Goodscore"-branded exports)

Full parser architecture notes, known PDF-extraction gotchas, and the testing workflow used to validate changes without a browser live in [`.claude/skills/scoredesk-cibil-parser-dev/SKILL.md`](.claude/skills/scoredesk-cibil-parser-dev/SKILL.md) — read that before touching `parseReport()` or adding a new bureau layout.

## Contributing / fixing a parser bug

1. Get the real extracted text of the failing PDF first (see the skill doc — don't guess at PDF structure).
2. Extract the relevant `<script>` block from `index.html` and test the parsing logic under Node (no browser needed for this part) — the skill doc has the exact commands.
3. Re-run against every previously-fixed sample report you still have, not just the new one, to catch regressions before shipping.
4. Update `.claude/skills/scoredesk-cibil-parser-dev/SKILL.md` in the same commit if you learned something that would've saved you time — that's what keeps it useful instead of going stale.

## License

See [`LICENSE.md`](LICENSE.md) — source is public for transparency (so anyone can verify the "your report never leaves your browser" claim for themselves), but this isn't an open-source license: reuse/redistribution needs permission. Pull requests to fix bugs or add a report layout are welcome.

