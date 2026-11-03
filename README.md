# csv-doctor

A single-file, offline CSV / Excel-export cleaner. No install, no upload, no account.
Open `index.html` in a browser and paste your data in.

**Live version:** https://csv-doctor.app.workbuddy.host/?sharecode=I7AADPFGF2

---

## Why this exists

Most "clean your CSV" tools either strip characters blindly or ask you to trust them with
your data. Both are wrong.

The real failure mode in spreadsheet exports is not noise, it's **silent data loss**:

- `$10,000.50` written without quotes gets split by the comma into `$10` and `000.50`.
  A naive cleaner sees a row with one column too many, truncates it, and **the amount
  disappears**. Nobody notices until the books don't balance.
- A column mixing `2026/03/04` and `2026-03-05` looks fine and sorts wrongly forever.
- Full-width punctuation (`；` `，` `０-９`) turns an entire CSV into a single column.

So this tool has three rules, in priority order:

1. **Never lose data silently.** If a row cannot be resolved confidently, it is kept and
   flagged, not dropped.
2. **Only fix what is deterministic.** Reversible normalisation happens automatically.
   Anything that requires guessing is surfaced for a human decision instead of guessed.
3. **Show your work.** Every change is counted and attributable to a rule.

## What it fixes

| Rule | Example | Result |
|---|---|---|
| Near-duplicate rows | `" Acme  Ltd "` vs `"Acme Ltd"` | flagged, not auto-merged |
| Unquoted thousands separators | `$10,000.50` → `$10` + `000.50` | rejoined when the grouping is strict |
| Mixed date formats | `2026/3/8`, `2026.03.04`, `2026年3月4日` | normalised to `YYYY-MM-DD` |
| Ambiguous US/EU dates | `03/06/2026` | **left untouched** — genuinely ambiguous |
| Full-width digits | `１２３４` | `1234` |
| Currency variants | `￥1,299.00`, `RMB 880`, `1,200元` | `¥1299`, `¥880`, `1200` |
| Trailing zeros | `1299.00` | `1299` |
| Ragged rows | wrong column count | kept and flagged with row numbers |

## Verification

Not "it seems to work". Actually measured:

```
23 / 23 assertions passing
5,000 dirty rows parsed and cleaned in ~60 ms
```

The test suite runs in a real browser against the real parser, asserting exact cell values —
not string matching on the output.

## Three bugs found while building it

These are worth mentioning because they are the kind of thing that costs a client money:

1. **`1,200元` never cleaned.** The regex used `\b` as a word boundary, which does not
   match between a digit and a CJK character. The rule silently did nothing on Chinese data.
2. **`$10,000.50` truncated.** An unquoted thousands separator produces an extra column.
   The original code sliced the row to width — destroying the amount. Fixed by rejoining
   only strict `\d{3}` groupings **before** truncation, and by keeping wide rows intact
   rather than cutting them.
3. **Rejoin ran in the wrong order.** Merging cells after cleaning could glue two unrelated
   adjacent columns. Moved before cleaning, restricted to numeric shapes.

## Limits (stated on purpose)

- It does not guess ambiguous dates. That is a decision, not an omission — a wrong date
  costs more than a flagged one.
- It does not merge near-duplicate rows automatically. It reports them so a human decides.
- `.xlsx` is not supported; export to CSV first. The tool tells you this instead of failing
  mysteriously.
- Near-duplicates use normalisation (case, whitespace, punctuation), not fuzzy scoring.

## Files

```
index.html   the whole tool — HTML, CSS, JS, parser, rules, tests. Nothing else.
```

No dependencies. No build step. Open it and it works, including offline.
