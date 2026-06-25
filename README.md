# daily-order-app

Pharmacy stock tracker — new items + availability search across 4
distributors (Kundu, Shivam, Sri Ganpati, New Ganpati), parsed from PDF
stock lists.

This is a static site (GitHub Pages, no backend). One PC uploads PDFs and
writes data to this repo; phones only read that data. See `CLAUDE.md` for
the full spec, and `decisions.md` for the reasoning behind each design
choice (kept for interview reference).

This file is the practical "how do I actually use this" guide.

---

## Getting a GitHub token (do this each time it expires)

Saving stock data to GitHub requires a **fine-grained personal access
token**, scoped to only this one repo. It is never saved anywhere — you
paste it fresh into `upload.html` each time you use it, and it disappears
the moment you close or refresh the page. This is intentional (see
`decisions.md` #4): a write-capable credential should never sit saved on
disk on a shared/older PC.

**Steps to generate one:**

1. Go to **github.com**, log in, click your profile picture (top-right) →
   **Settings**.
2. Scroll to the bottom of the left sidebar → **Developer settings**.
3. Click **Personal access tokens** → **Fine-grained tokens**.
4. Click **Generate new token**.
5. **Token name**: anything recognizable, e.g. `daily-order-app-pc`.
6. **Expiration**: e.g. 90 days. (Doesn't change how you use it day-to-day —
   you're re-pasting it every session regardless. It just means you'll need
   to make a new one after it expires.)
7. **Repository access** → **Only select repositories** → pick
   `royakash-cloud/daily-order-app`. Do not choose "All repositories."
8. **Permissions** → **Repository permissions** → set **Contents** to
   **Read and write**. GitHub will automatically also require **Metadata:
   Read-only** — that's normal, just leave it.
9. Click **Generate token** at the bottom.
10. GitHub shows the token **only once** — copy it immediately. It looks
    like `github_pat_11ABC...`.

**Important:** never paste this token anywhere except the token box in
`upload.html` — not into chat, email, or any other site. If it's ever lost
or expires, just repeat these steps to make a new one; nothing else breaks.

---

## Day-to-day flow — uploading a new stock list

1. Open `upload.html` on the PC.
2. Paste your GitHub token into the box at the top. (You'll see "Token set
   for this session" once it's recognized.)
3. Drop the distributor's PDF into its box (Kundu / Shivam / Sri Ganpati /
   New Ganpati). The page extracts the product list right there in the
   browser — nothing is sent anywhere yet at this point.
4. Check the result: the product count shown, and optionally type a known
   product name into the search box on that card to spot-check it's correct.
5. Click **Save to GitHub** on that distributor's card.
   - If this is the first-ever save for that distributor, it just saves.
   - If a previous save exists and the new count is wildly different
     (more than ~40% up or down from last time), you'll get a warning popup
     before anything is saved — see "What if something looks wrong" below.
6. The Activity Log at the bottom records what happened in plain English —
   expand it any time something needs explaining.
7. Repeat for each distributor PDF you have that day. They're independent —
   you don't need all 4 at once.

## What if something looks wrong (count warning, or an error)

- **"This looks abnormal — save anyway?" popup**: means the new count is
  very different from the last save for that distributor. This usually
  means either the wrong PDF was dropped in the wrong box, or that
  distributor changed their PDF format. If you're not sure, click Cancel —
  nothing is touched, your last good data on GitHub stays exactly as it
  was. You can always read that day's PDF manually as a fallback while this
  gets sorted out later.
- **"GitHub write failed... check your token"**: your token has likely
  expired or was mistyped — generate a new one using the steps above.
- **"Found 0 products"**: the PDF probably isn't in the format expected for
  that distributor (e.g. dropped into the wrong box), or the distributor's
  PDF layout changed. Bring the PDF to a Claude Code session to diagnose —
  it's usually a small, targeted fix to one parsing rule, not a rewrite.

## Using the read-only page (phones, or anyone checking stock)

Open `index.html` — no token needed, nothing to set up. It shows:
- **Search Availability** — type a product name, see matching products and
  quantities for each of the 4 distributors (or "Not available" if none).
- **New Items** — per distributor, what's newly present since the last PC
  upload. On a distributor's very first save, everything is new by
  definition — the page says so explicitly so it doesn't look like a glitch.

This page only reads from GitHub (via raw.githubusercontent.com) — it never
writes anything and has no token anywhere in its code.

## What's built so far

- ✅ PDF parsing for all 4 distributors (`upload.html`), verified against
  real sample PDFs.
- ✅ Saving parsed stock + new-items diff to GitHub, with the count-anomaly
  safety check above.
- ✅ Phone-facing read-only page (`index.html`) — New Items List and
  Availability Search, both built on the GitHub read step.
- ✅ Activity Log on both pages, logging every fetch/parse/save attempt in
  plain English.

All of Version 1's scope from `CLAUDE.md` is now built, and `index.html`,
`upload.html`, `stock_pdf_importer.html`, and `decisions.md` are all pushed
to GitHub (as of 2026-06-20) — GitHub Pages serves them at
`https://royakash-cloud.github.io/daily-order-app/`.

## Remaining / for future reference

Nothing is left to *build* for Version 1. These are the only open items —
keep this list updated rather than starting a new one:

- **Confirm the phone view works in real day-to-day use.** Open
  `https://royakash-cloud.github.io/daily-order-app/` on each phone, check
  New Items and Search both look right, and bookmark/"Add to Home Screen"
  it (see day-to-day flow above). Not a code task — just a check.
- **GitHub token expires periodically** (whatever expiration you picked
  when generating it — see "Getting a GitHub token" above). When
  `upload.html` says the save failed because of your token, just generate
  a new one with the same steps. This is expected, recurring maintenance,
  not a bug.
- **Watch Sri Ganapati's file size.** Per `decisions.md` #12, GitHub's
  Contents API (used for all reads/writes) caps out at 1MB per file. Sri
  Ganapati's save was ~523KB as of the last check — if its product count
  roughly doubles, reads/writes to `data/sri_ganapati.json` could start
  failing. If you ever see a "GitHub read failed" specifically for Sri
  Ganapati and nothing else, this is the first thing to check.
- **If a distributor changes their PDF's layout**, `upload.html` will
  likely report "Found 0 products" for that one distributor (see "What if
  something looks wrong" above). That's a small, targeted parser fix, not
  a rewrite — bring the new-format PDF to a Claude Code session.
- **Two write paths exist and don't sync automatically**: `upload.html`
  saves stock data straight to GitHub via its API (bypassing git on this
  PC entirely), while the page files themselves (`index.html`,
  `upload.html`, etc.) only reach GitHub through a normal `git push` from
  this folder. If you ever edit these files locally again, remember to
  commit and push — the data side will keep working regardless, but the
  pages won't update on GitHub Pages until pushed. (This is exactly what
  caused the "index file not updating" issue fixed on 2026-06-20.)

Anything not listed above and not in `CLAUDE.md`'s "NOT in Version 1"
section is out of scope by deliberate choice, not an oversight — see
`decisions.md` #5 for why.
