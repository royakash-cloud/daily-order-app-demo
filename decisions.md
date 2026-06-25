# Decisions Log — Daily Order App (v1)

This file records architecture and design decisions made during development,
with the reasoning behind each. Kept for future reference / interview discussion.

---

## 1. Static site + GitHub-file persistence (no backend, no database)

**Decision:** Host as a static site on GitHub Pages. No server, no database.
Data is stored as JSON files committed to a GitHub repo, read/written via the
GitHub REST API.

**Why:**
- Single writer (one PC), multiple readers (phones) — doesn't need a real
  backend or sync engine.
- Zero hosting cost, no credit card required anywhere in the stack.
- A previous attempt at this app used Firebase and a more complex multi-device
  sync model and became too hard to maintain/debug solo. This is a deliberate
  simplification.

**Alternatives considered:** Firebase/Firestore (rejected — too complex to
self-maintain, requires more moving parts than the problem needs).

---

## 2. PDF parsing happens only on one fixed PC, in-browser

**Decision:** All PDF upload + parsing happens client-side (pdf.js) on one
specific Windows 7 PC. Phones never parse PDFs and never write data — they
only fetch the already-written JSON and render it.

**Why:**
- Avoids needing a server to do parsing.
- Matches real usage: only one person (the pharmacist) uploads stock lists,
  always from the same machine.
- Phones are read-only consumers — simpler permission model, no write
  conflicts possible.

---

## 3. Reuse existing PDF parsers instead of rewriting

**Decision:** Port the 4 distributor parsers already written and working in
`stock_pdf_importer.html` from an earlier project, rather than rewriting them
from scratch.

**Why:**
- These parsers already handle real-world quirks per distributor format
  (e.g. Sri Ganapati / New Ganapati's WPS-generated PDFs need
  character-gap-based space reconstruction because the underlying PDF text
  has inconsistent spacing). Rewriting risks reintroducing already-solved
  edge cases.

---

## 4. GitHub token is never persisted

**Decision:** The GitHub personal access token (fine-grained, scoped to only
this one repo, contents read/write) is typed in fresh each session on the PC.
It is never saved to localStorage, disk, or the repo itself, and never appears
in the phone-facing read-only code path.

**Why:**
- Security: a token with write access to the repo should not sit unencrypted
  on a shared/older machine (Windows 7) or leak into a code path served to
  other devices.
- Trade-off accepted deliberately: re-typing the token each session is a
  minor inconvenience in exchange for not storing a credential at rest.

---

## 5. Scope cut to exactly two features

**Decision:** Version 1 does only:
1. New Items List (diff of latest upload vs previous upload, per distributor)
2. Availability Search (product name → quantity per distributor)

Explicitly excluded: order list/cart, local distributors, Firebase/multi-user
sync, stock quantity editing, persisted tokens, order history beyond
"previous vs current".

**Why:**
- The earlier version of this project grew too complex (order lists, local
  distributors, Firebase) to debug solo. This restart deliberately narrows
  scope to the two things actually needed day-to-day, so the owner (non-
  programmer) can understand and fix the whole system themselves.

---

## 6. "Replace, don't accumulate" data model

**Decision:** Each new stock list upload for a distributor completely replaces
the previous one in the stored JSON. No history is kept beyond "previous vs
current" — that's the only basis for the new-items diff.

**Why:**
- Simplicity: avoids needing a database or versioned storage. A flat JSON
  file with one snapshot per distributor is enough for the actual use case
  (the owner only cares what's newly available since the last list).

---

## 7. On-page Activity Log instead of relying on browser console

**Decision:** Every operation (PDF upload, parsing, GitHub write, search,
GitHub read) logs a plain-English attempted/succeeded/failed message to a
visible, expandable "Activity Log" panel on the page itself.

**Why:**
- The owner doesn't routinely use browser dev tools and needs to self-
  diagnose problems (e.g. "wrong PDF format uploaded", "token expired")
  without outside help.

---

## 8. Two separate HTML pages, not one page with conditional UI

**Decision:** Build `upload.html` (PC-only: drop PDFs, parse, eventually
write to GitHub) and `index.html` (phone-facing: read-only New Items +
Search) as two separate files, rather than one page that shows different UI
depending on device.

**Why:**
- Keeps upload/parsing/token code physically out of the file phones load —
  no risk of write logic or token-handling code ever being reachable from
  the read-only path, and no need for device-detection logic to get that
  guarantee.
- Matches the real usage pattern: only one person, on one fixed PC, ever
  needs the upload page; everyone else only ever needs the read-only page.

---

## 9. Build order: parsers → GitHub write/read → UI

**Decision:** Verify all 4 PDF parsers work correctly first, before building
the GitHub write step; verify GitHub write/read works before building the
search/new-items UI on top of it.

**Why:**
- Incremental verification at each layer makes it easier to isolate bugs
  (parsing vs. storage vs. UI) and matches the owner's stated need to be able
  to debug each piece independently.

---

## 10. Independent parser verification against real PDFs, not just browser eyeballing

**Decision:** Before trusting the ported parsers, ran the exact same
`extractLines`/`parse` logic in a standalone Node script (pdfjs-dist) against
the owner's actual stock list PDFs for all 4 distributors, and diffed the
output against the raw PDF text — rather than relying only on manual
spot-checks in the browser UI.

**Why:**
- Caught a real bug this way: the Sri Ganapati PDF (19-6-2026) has literal
  watermark/footer text reading `ZZZZZZ 15181` and `ZZZZZZ 17347` near the
  end of the document. The generic "name + trailing number" rule for the
  sg_ng format had no way to distinguish this from a real product line, so
  it was parsed as 2 fake products (qty 68 and 10). Left unfixed, these
  would have falsely shown up as "new items" on every future upload.
- **Fix applied:** skip any line where a letter repeats 5+ times consecutively
  (`/([A-Za-z])\1{4,}/`) — narrow enough that no real medicine name would ever
  match it, so it only catches this kind of junk.
- Kundu, Shivam, and New Ganapati parsers checked out correct on spot-checks
  (specific product names/quantities matched the source PDF exactly) — no
  changes needed for those three.

---

## 11. Save-anomaly confirmation gate before any GitHub overwrite

**Decision:** Before writing a new distributor's stock list to GitHub,
compare the new product count against the count from the last save for that
same distributor. If it changed by more than 40% in either direction, block
the save behind a plain-English confirmation dialog explaining the size of
the change and that the last good data will be permanently replaced — the
save only proceeds if the owner explicitly confirms.

**Why:**
- The data model is "replace, not append" ([[decision-6]] / item 6 above) —
  there is no history to recover from if a bad parse (e.g. a distributor
  changing their PDF format without warning) gets saved over good data.
- A non-programmer owner can't fix a broken parsing rule on the spot if no
  one is available to help at that moment. The actual risk isn't "I can't
  code a fix" — it's "bad data silently destroyed good data before anyone
  noticed." This gate makes that specific failure require an explicit,
  informed choice instead of happening silently.
- 40% was chosen as a threshold loose enough to not nag on normal day-to-day
  stock fluctuation, but tight enough to catch the kind of swing a format
  change or wrong-file upload would cause (e.g. the Sri Ganapati case
  would have shown a guaranteed warning if the previous save had a normal
  count and the watermark bug had gone unfixed).

---

## 12. GitHub Contents API is fine for now — noted limit for later

**Decision:** Use the plain GitHub Contents API (simple GET/PUT of a JSON
file, base64-encoded) for reading and writing distributor data, rather than
the lower-level Git Blob/Tree API.

**Why / known limit:** the Contents API only supports files up to 1MB. All
4 distributors' first real saves came in well under that (largest, Sri
Ganapati, was 523KB), so this is not a problem today. If Sri Ganapati's
product count grows enough to push that file past ~1MB, reads via this API
would start failing and the app would need to switch to the Git Blob API
for that one file. Not building that now — just noting it so a future
"GitHub read failed" on that specific distributor isn't a mystery.

---

## 13. Limitation found: Node-based parser verification isn't a perfect mirror of the browser

**Decision/finding:** When the real Kundu save came in at 969 products
(vs 845 found by the earlier Node-based verification script for the
identical PDF), investigated rather than assumed either number was simply
"right." Ruled out missing standard font data in Node (loading the real
font files made no difference). Root cause: the Kundu PDF has duplicate-
looking product names appearing more than once across its 24 pages with
different quantities, and the parser intentionally keeps the highest
quantity seen for an exact name match. Node and the real browser found
slightly different sets of these duplicate occurrences due to subtle
row-positioning sensitivity — not a clear logic bug like the earlier
"ZZZZZZ" watermark issue.

**Why this matters going forward:**
- The live app is still reliable day-to-day: the same browser, parsing the
  same PDF, will consistently produce the same result each time on the PC.
  This is not a bug affecting actual usage.
- My standalone Node verification script is a useful tool for catching
  clear-cut bugs (e.g. ASCII junk text), but is **not a perfectly reliable
  mirror of real browser PDF parsing** for subtle row-grouping edge cases.
  A count difference between my Node script and the live app's saved result
  is not, by itself, proof of an app bug.
- **The trustworthy way to verify a specific upload going forward is the
  search/spot-check box already built into `upload.html`** (same browser
  engine that did the real parsing), not re-running it through a separate
  Node script.
