# AGENTS.md

Context for anyone (human or agent) picking up this repo. Written 2026-09-01.

**Re-verify anything dated before you rely on it.** The regulatory figures below were
checked against primary sources on 2026-09-01; thresholds move on statutory cycles and
an LLM's training data goes stale silently. See "How to re-verify" at the end.

---

## What this is

**Procurement Pathfinder** — a single-page decision tool that walks ESC Region 13 staff
through a 5-step wizard (amount → funding source → category → awarded vendor → result)
and returns the required procurement method, legal authority, and next actions.

Owner / subject-matter authority: **Vicki Cagle, Contracts & Procurement Coordinator.**
Policy content is hers. Do not "fix" thresholds, method names, or statutory citations
based on your own reading of the regulations — surface the question to her instead.
Several things that look like bugs are deliberate policy (see Open Questions).

## Architecture

`index.html` is the whole application. There is no build step, no package.json, no tests.

- React 18 UMD + `@babel/standalone` compiling the inline `<script type="text/babel">` in
  the browser. Tailwind v2 utility classes plus inline `style={{}}` objects.
- One component, `ProcurementTool`, holding five `useState` values.
- `determineResult()` is the entire decision engine — a flat cascade of
  `if (category) → if (vendor) → if (amount)` returning a result object, or `null`.
- `LinkedText` turns literal substrings ("BID NBR", "Procurement") in result strings into
  anchors. This is why result copy must keep those exact spellings — renaming them in a
  string silently kills the link.
- Deploys as a static file (GitHub Pages from `Region13/procurement-compliance-tool`).
  The logo loads from a `?raw=true` GitHub URL, not the local `.png` — the local copy is
  effectively unused by the page.

### Hard constraints — do not regress these

Commit `528fef1` (PR #1, "adds sri hashes so we can trust cdn content") added Subresource
Integrity to every CDN dependency. **Keep it.** With no build step and no CSP, SRI is the
only thing standing between a compromised CDN and arbitrary JS in staff browsers.

- All four `<head>` dependencies must stay **exact-version pinned** with an `integrity`
  hash and `crossorigin="anonymous"`. Never floating majors (`react@18`), never unpinned.
- **Tailwind must stay on the v2.2.19 prebuilt stylesheet.** Do not "upgrade" to
  `cdn.tailwindcss.com`. That is the v3 play CDN — a runtime compiler shipped as a script,
  which cannot carry an SRI hash at all and is not intended for production. Tailwind v3
  ships no prebuilt CDN stylesheet, so there is no hashable v3 option. Staying on v2 is a
  deliberate security tradeoff, not neglect.

This has already regressed once (see History) — it is the most likely thing to regress again.

## History

| When | What |
|---|---|
| `528fef1` (PR #1) | Added SRI hashes + pinned versions; replaced direct HTML injection with components |
| 2026-09-01 | Vicki delivered `update.html` — a policy update built from the **pre-PR#1** file, so it silently reverted all SRI hardening |
| 2026-09-01 | Merged: her logic onto the hardened head. `update.html` is now redundant and can be deleted |

**Lesson:** Vicki works from whatever copy she has locally. Expect future updates to arrive
as a whole replacement file built from a stale base. Always diff the `<head>` before
accepting one, and merge rather than copy.

## The 2026-09-01 policy change

Amount tiers went from four to three — `$0–$49,999` and `$50,000–$99,999` merged into a
single `$0–$99,999` band:

- Removed the "Simplified Acquisition/Small Purchase → Obtain 3 quotes" branch from both
  Goods/Services and Construction.
- Consequence: **nothing sets `showQuoteLink: true` anywhere in the file.** The Quote
  Attempt Verification Form link is unreachable, and `QUOTE_VERIFICATION_FORM_URL` plus its
  render block are dead code. Left in place deliberately, pending the question below.

## Regulatory reference (verified 2026-09-01)

### Federal

| Threshold | Amount | Note |
|---|---|---|
| Micro-purchase (MPT) | **$15,000** | was $10,000 |
| Simplified acquisition (SAT) | **$350,000** | was $250,000 |
| Max self-certified MPT | **$50,000** | annual; above this needs cognizant-agency approval |

Both FAR figures rose **effective 2025-10-01** under the fifth statutory inflation review —
[90 FR 41872 (2025-08-27)](https://www.federalregister.gov/documents/2025/08/27/2025-16412/federal-acquisition-regulation-inflation-adjustment-of-acquisition-related-thresholds),
confirmed by [GSA SmartPay Bulletin 002](https://smartpay.gsa.gov/guidance-and-audits/smart-bulletins/002/).

2 CFR 200.320 **hardcodes no dollar amounts** — it points at the FAR figures through
§ 200.1, so the Uniform Guidance bands move automatically with the FAR. Quoting a dollar
figure "from 200.320" is always a mistake; cite § 200.1 / FAR 2.101 and give the date.

- 200.320(a)(1): micro-purchases need no competitive quotes, but price reasonableness must
  still be documented. Self-certification up to $50,000 annually, documentation retained
  per § 200.334; above $50,000 requires written cognizant-agency approval.
  ([Cornell LII](https://www.law.cornell.edu/cfr/text/2/200.320) — eCFR blocks automated
  fetches and bounces to a bot-check page.)
- 200.320(a)(2): small purchase procedures apply **between the MPT and the SAT** — price or
  rate quotes from an adequate number of qualified sources. This is the band the deleted
  "3 quotes" branch covered.
- 200.324: cost or price analysis required above the SAT. This is the tool's $350,000
  "Cost Price Analysis" warning — correct and current.

**Pending, not in effect:** OMB published a proposed rule 2026-05-29 that would
substantially rewrite 2 CFR Part 200. Proposed ≠ effective. Confirm status before assuming
anything above still holds.

### Texas

- **TEC 44.031(a)** — competitive procurement required at **$100,000** and above, raised
  from $50,000 by [HB 1132](https://capitol.texas.gov/tlodocs/88R/analysis/html/HB01132E.htm)
  (88th Leg.). This is the source of the tool's "$100k — formal competition required."
- **TGC 2254** — professional services; qualification-based selection, price not a factor.
- **TGC 2269** — construction delivery methods (CSP / CMAR / Design-Build).

## Open questions for Vicki

Unresolved as of 2026-09-01. Do not answer these by reading the regulations — ask.

1. **Does the new $100,000 informal tier apply to federal funds?** The merged branch returns
   `authority: '2 CFR 200'` with "Under threshold - Informal process acceptable" for
   `$0–$99,999` **regardless of the funding source selected at step 2**. Federally, small
   purchase procedures under 200.320(a)(2) run from the MPT all the way to $350,000, and
   $99,999 exceeds even the maximum self-certifiable MPT of $50,000.
   - If the answer is "state/local only," the fix is to make that branch funding-aware —
     which would also make the quote-form link reachable again.
   - If it applies to federal funds too, ESC13 needs cognizant-agency approval on file for
     an MPT above $50,000. She may already have it.
   - The old `$0–$49,999` tier looked exactly like a $50,000 self-certification, and
     ESC13's fiscal year starting September 1 lines up with the annual certification cycle.
2. **Is "Micro-Purchase" the right label for the whole $0–$99,999 band?** If the band is
   really about TEC 44.031's state threshold, "Simplified Acquisition/Small Purchase" may
   fit better above $15,000.
3. **Is the 3-quote requirement gone, or did it just move?** Determines whether the dead
   `showQuoteLink` code gets restored or deleted.

## Known quirks (pre-existing — verified present before the 2026-09-01 merge)

Do not attribute these to the policy update; they predate it. Confirmed by running the
harness below against `git show 528fef1:index.html`.

- **`showFederalWarning` ignores funding source.** Goods and Construction at `$350,000+`
  show the "Federal purchases over $350,000" warning and cite 2 CFR 200 even when funding
  is State or Local.
- **Professional Services never shows it.** The no-vendor QBS path returns no
  `showFederalWarning` even on federal funds at `$350,000+`. The vendor path checks
  `isFederal` correctly, so the two halves of that category are inconsistent.
- `showWarning` state is set and reset but never read. Dead.
- The `categories` and `vendors` arrays are used only for the step-5 "Your Selections"
  labels; step 3 and step 4 hardcode their buttons instead of mapping over them. Adding an
  option means editing two places.

## How to re-verify

No test suite. These two checks catch most breakage and take seconds.

**Does the JSX still compile, under the exact pinned Babel?**

```bash
curl -sL -o /tmp/babel.min.js https://unpkg.com/@babel/standalone@7.29.2/babel.min.js
node -e "
const fs=require('fs'), Babel=require('/tmp/babel.min.js');
const m=fs.readFileSync('index.html','utf8').match(/<script type=\"text\/babel\">([\s\S]*?)<\/script>/);
Babel.transform(m[1],{presets:['react']}); console.log('JSX OK');
"
```

**Exercise every decision path** (81 combinations; catches `null` results, which render a
blank step 5, and confirms which paths set flags):

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const src=html.slice(html.indexOf('const determineResult'), html.indexOf('const handleNext'));
const f=new Function('amount','funding','category','vendor', src + '\n return determineResult();');
let nulls=0;
for(const a of ['0-99999','100000-349999','350000+'])
 for(const fu of ['federal','state','local'])
  for(const c of ['goods','professional','construction'])
   for(const v of ['no','esc13','coop']){
     const r=f(a,fu,c,v);
     if(!r){nulls++; console.log('NULL:',a,fu,c,v);}
   }
console.log('null results:', nulls);
"
```

Update the amount array when tiers change. As of 2026-09-01: 81 combinations, 0 nulls,
0 paths setting `showQuoteLink`.

**Verify the SRI hashes still match what the CDN serves:**

```bash
for u in \
  "https://unpkg.com/react@18.3.1/umd/react.production.min.js" \
  "https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js" \
  "https://unpkg.com/@babel/standalone@7.29.2/babel.min.js" \
  "https://unpkg.com/tailwindcss@2.2.19/dist/tailwind.min.css"; do
  echo "sha384-$(curl -sL "$u" | openssl dgst -sha384 -binary | openssl base64 -A)  <- $u"
done
```

All four matched `index.html` on 2026-09-01. A mismatch means either the file was
republished or something is wrong — investigate before updating the hash.
