---
name: scoredesk-cibil-parser-dev
description: Developer knowledge base for ScoreDesk, a single-file client-side HTML/JS tool (index.html) that parses Indian credit bureau report PDFs (CIBIL/TransUnion, Experian, CRIF High Mark "Goodscore", and other layouts) entirely in the browser and renders a risk dashboard, account table, and downloadable Excel export. Use this skill whenever the user asks to fix, extend, debug, or add a new bureau/report layout to ScoreDesk or "the CIBIL tool" / "index.html credit report parser" — especially bugs where the score shows wrong, accounts aren't captured, a report's bureau isn't recognized, or the gauge/meter visual doesn't render. Also use it before writing any new parseFormatX function or touching parseReport()'s dispatcher chain, since it documents failure modes that have already bitten this codebase twice.
---

# ScoreDesk credit-report-parser development

ScoreDesk is a single `index.html` file (~3000 lines, no build step, no backend). It loads a PDF with pdf.js entirely client-side, regex-parses the extracted text into `parsedAccounts`/`parsedEnquiries`/`reportMeta`, then renders a risk dashboard and an Excel (`xlsx.full.min.js`) export. The privacy pitch on the page ("never uploaded to any server") is load-bearing for every design decision below — don't add anything that phones home.

## Before touching anything: get real extracted text, not the PDF

The single biggest source of bugs in this codebase is guessing at PDF text order instead of checking it. **Never write or fix a parser regex against your mental model of "how the PDF looks" — always extract the actual text layer first.**

```bash
python3 -c "
import pdfplumber
with pdfplumber.open('report.pdf') as pdf:
    text = '\n'.join((p.extract_text() or '') for p in pdf.pages)
with open('report_full.txt','w') as f: f.write(text)
"
```

pdfplumber's extraction order is a reasonable proxy for what pdf.js (used in the actual browser code) will produce, but they aren't identical — when in doubt, treat pdfplumber's output as "what the regexes have to survive," not gospel.

**Known extraction gotcha:** multi-column layouts where a table cell's label wraps onto a second line (e.g. a 2-word label like "Credit Grantor Type" printed as "Credit Grantor\nType") can come out of the text layer **row-interleaved with the neighboring column**, not in reading order — e.g. `"Acct Number XXXXXXXXXXXX2478\nLast Payment\nLast Paid Amount - -\nDate\n..."`. A value can end up genuinely separated from its own label by an unrelated fragment. There is no universal regex fix for this; the fix is per-layout normalization (see `normalizeWrappedLabelsI()` in the Goodscore parser for the pattern: a chain of `.replace(/Label part 1\s*\n\s*label part 2/gi, 'Label part 1 part 2')` calls that stitches known wrap points back into one line before field-matching runs). Budget for a few fields staying imperfect on messy layouts (e.g. exact account number, EMI) — get the fields that actually drive lending decisions right (balance, overdue, sanctioned amount, status, ownership) and don't chase 100% on cosmetic ones if the layout fights you.

## Dispatcher architecture (`parseReport` → `parseFormatA`...`parseFormatI`)

`parseReport(text)` tries each format's parser in sequence and stops at the first one that populates `parsedAccounts`. Each `parseFormatX(text)` is expected to **return early (no-op) if the text doesn't look like its own layout**, since a false positive silently prevents every later format — including the correct one — from ever running.

**This has caused a real production bug twice.** Format B splits account blocks on any occurrence of the literal string `"Member Name"`. A CRIF "Goodscore" report's *Credit Enquiries table* also happens to use "Member Name" as a column header. When that table had at least one real enquiry row, Format B matched it as an account-block start and vacuumed the entire rest of the document (every real account, the summary, everything) into one garbled "account," while the correct parser (Format I) never got a chance to run because the dispatcher only advances to the next format when the previous one found zero accounts.

**The fix pattern, and the one to follow for any new layout:** if a layout has a strong, unambiguous fingerprint (a literal string that's essentially unique to it — e.g. Goodscore/CRIF reports print `"CHM Ref. No."` on every single page), check for that fingerprint **at the very top of `parseReport`**, before the generic A–H heuristic chain runs at all, and route straight to that format's parser:

```js
if (/CHM Ref\.\s*No/i.test(text)){
  parseFormatI(text);
  if (!reportMeta.score){ reportMeta.scoreIsImage = /* generic fallback check */; }
  return;
}
```

This is more robust than trying to make every *other* format's block-detection stricter one collision at a time — you only have to get the new format's own fingerprint right once, and it can't be hijacked by anything upstream ever again. When adding a new bureau layout, look for a similarly strong fingerprint (a reference-number label, a distinctive section header, an unusual literal phrase) before falling back to "just add it as format J in the generic chain."

## Score extraction: don't trust proximity-based regexes

A once-shipped bug: Experian's score section reads (after PDF extraction) as *"...Experian Credit Score which ranges from 300 - 900...YOUR EXPERIAN\nCREDIT SCORE\n547\nRange: 300 – 900"* — a `/CREDIT SCORE[\s\S]{0,40}?(\d{3})\b/i` regex matches the **300** from the descriptive sentence, not the real 547 score, because it's closer. The fix that generalizes: don't search for "a number near this label," search for a number that's **structurally distinguishable** from the noise around it — in that case, a 3-digit number sitting completely alone on its own line was the one property unique to the real score (every other number nearby has surrounding text on the same line). Isolate the relevant section first (`text.match(/EXPERIAN CREDIT SCORE([\s\S]*?)REPORT SUMMARY/i)`), then apply the structural check inside that narrowed slice.

## `cibilScoreBand()` / `BAND_LABELS` / `BAND_COLORS` — the canonical 5-tier scale

Standard CIBIL score bands used across the dashboard (gauge segments, band pill, legend, and any other metric card that reuses this palette):

| Band | Range |
|---|---|
| Poor | 300–549 |
| Fair | 550–649 |
| Good | 650–749 |
| Very Good | 750–799 |
| Excellent | 800–900 |

Breakpoints live in `CIBIL_BAND_BREAKS = [549, 649, 749, 799]`. If you touch this, keep `BAND_LABELS`/`BAND_COLORS` as the single source of truth other UI reads from — don't hardcode a second color scheme for a new widget.

## SVG gauge: never give two `<defs>` elements on the same page the same `id`

`gaugeSvg(score)` is called at least twice per session — once for the Step‑1 live preview (which gets hidden via `display:none`, not removed from the DOM, when scanning finishes) and again for the Step‑2 risk dashboard. A shared gradient `<linearGradient id="rdGaugeGrad">` in both instances meant the second, visible instance's `url(#rdGaugeGrad)` paint reference could silently fail to resolve on some mobile browsers when the *first* copy of that id lived inside a `display:none` ancestor — the arc rendered invisible while the (solid-color, non-gradient) needle stayed visible, which is a very confusing bug report ("only the needle shows") if you don't know to look for duplicate ids. Current implementation sidesteps this entirely by drawing the gauge as solid-color arc segments (no `<defs>`/gradient/id at all) rather than fixing the id collision — prefer that pattern (no shared ids) over "make the id unique" if you're touching gauge rendering again.

## Testing workflow (no browser available in this environment)

There's no way to actually render the page here, so validate parser/logic changes by extracting the relevant `<script>` block(s) and running them under Node with a small driver:

```bash
# 1. Find the contiguous JS region (clean() through end of parseFormatE, or
#    wherever your edit sits) and slice it out — these functions don't touch
#    the DOM so they run fine standalone.
sed -n '<start>,<end>p' index.html > parser_slice.js

# 2. Wrap it with the module-level state it expects, then a driver that
#    calls parseReport(text) and prints reportMeta / parsedAccounts.
cat > full_test.js <<'EOF'
let parsedAccounts = [], parsedEnquiries = [], consumerName = "", reportMeta = {};
EOF
cat parser_slice.js >> full_test.js
cat >> full_test.js <<'EOF'
const fs = require('fs');
const text = fs.readFileSync('report_full.txt', 'utf8');
parseReport(text);
console.log(reportMeta.bureau, reportMeta.score, parsedAccounts.length);
EOF
node full_test.js
```

Also syntax-check the whole file after any edit — a single misplaced brace anywhere in a 3000-line inline `<script>` breaks the entire page silently:

```bash
python3 -c "
import re
html = open('index.html').read()
for i, s in enumerate(re.findall(r'<script(?:(?! src=)[^>])*>(.*?)</script>', html, re.S)):
    if len(s.strip()) > 50:
        open(f'script_{i}.js','w').write(s)
"
for f in script_*.js; do node --check "$f" || echo "SYNTAX ERROR in $f"; done
```

Re-run this full loop against **every previously-fixed sample report you still have text dumps for**, not just the new one — this is how the Format B collision above was caught as a regression risk before shipping, and it's cheap insurance against re-breaking format I/D while fixing format J.

## Known bureau layouts implemented so far

- **Format A–C, F–H**: various CIBIL/TransUnion export layouts (member-portal PDF, "CreditView" dashboard export, etc.)
- **Format D**: Experian. Per-account data lives in a clean 3-column `Account terms | Account description | Account details` table (parseable straight, one row per line) — but the *lender name* only exists as clean text in a separate header line (`"{TYPE}  {LENDER}  Acct N"`) printed above each account's table, not inside the table itself. The 2nd word of a 2-word account type (LOAN/CARDS) wraps onto the same line as the lender name in extracted text and has to be split back off (see `acctHeaderRegexD2` and the `typeWord2` regex in `parseFormatD`).
- **Format I**: CRIF High Mark "Goodscore"-branded reports. Fingerprint: `"CHM Ref. No."` header on every page (the actual "Goodscore"/CRIF logos are images, not text, so they can't be matched). Score is real text (`"Your Score is\n510"`), not an image, despite the visual gauge picture on the page. Each account is its own bordered card with a `"{LENDER} {Account Type}"` header, `"Account Number X | Account Type Y | Account Status Z"`, then a two-column field grid that suffers the label-wrap-interleaving problem described above. Has a `Credit Enquiries` table (label row: `Member Name | Inquiry Date | Purpose | Ownership Type | Amount | Remark`) — this is exactly the string that once collided with Format B; see the dispatcher section above.

When adding a new layout: extract a real sample's text first, look for a unique fingerprint string, add the early-dispatch short-circuit if one exists, write the field/account regexes against the *actual* extracted text (not assumed structure), and run the full Node test-harness loop against it plus every existing sample before considering it done.
