# Daily Investing Podcast — Claude Routine Instructions

You are running as an autonomous Claude Code Routine. You have shell access, can
modify files in this repository, and can push branches prefixed `claude/`.
Follow every step below exactly. Do not skip validation steps.

## 0. Persistent branch preparation

1. Fetch all branches.
2. If branch `claude/podcast-state` exists remotely, check it out and merge the
   latest `main` into it (`git merge origin/main`). Resolve trivial conflicts by
   preferring `main` for code/workflow files and preferring the existing
   `claude/podcast-state` version for `state/reported-stories.json`,
   `docs/feed.xml`, and `docs/podcast-history.json`.
3. If branch `claude/podcast-state` does not exist yet, create it from `main`.
4. Do all work on `claude/podcast-state`. Never commit directly to `main`.

## 1. Read prior state

1. Read `state/reported-stories.json`. If it does not exist or is not valid
   JSON, treat it as an empty list, but flag this in your final summary.
2. Discard entries older than 90 days relative to today's date (keep them out
   of the working set, but you may prune the file to drop anything older than
   120 days so it does not grow without bound).

## 2. Mandatory research architecture

Research using live web search. Use the following specialist research angles.
Do not skip any of them, even if a category turns out to have nothing new
today — an empty category is a valid outcome.

1. **Public equities & broad markets** — index-level moves only when they
   reflect a real structural development (not routine daily noise), notable
   earnings that change a long-term thesis, major buybacks/dividends policy
   changes, IPOs of significance, sector rotations with multi-year relevance.
2. **Real estate** — residential and commercial property market trends,
   mortgage rate developments and their effect on affordability, REITs,
   significant regulatory or tax changes affecting property investors,
   notable trends in specific geographies (Czech Republic and broader Europe
   get priority, but include major global trends too).
3. **Cryptocurrency & digital assets** — Bitcoin and major altcoin
   developments with lasting relevance (regulation, ETF flows, protocol
   changes, institutional adoption), explicitly excluding short-term price
   chatter with no underlying cause.
4. **Macro, rates & inflation** — central bank decisions (ECB, Fed, CNB),
   inflation data, currency moves, and anything that changes the backdrop for
   long-term asset allocation.
5. **Alternative & real assets** — commodities, gold and precious metals,
   infrastructure, private equity/venture trends, collectibles, farmland —
   anything materially relevant to a diversified long-horizon portfolio.
6. **Opportunistic short/medium-term situations** — special situations,
   temporary mispricings, unusual volatility with a clear catalyst,
   post-selloff entry points. Keep this category small: only include something
   here if it is a genuine, time-limited opportunity, not routine trading
   commentary. This is the only category where short-term framing is allowed.

## 3. Same-day category precedence

If one event fits more than one category (e.g., a central bank rate decision
affecting both macro and real estate), assign it to the single most specific
category:

1. Opportunistic short/medium-term situations (most specific — only if it
   is genuinely a time-limited opportunity)
2. Cryptocurrency & digital assets
3. Real estate
4. Public equities & broad markets
5. Alternative & real assets
6. Macro, rates & inflation (least specific — catch-all for broad
   macro-only stories)

Never report the same underlying event twice under two categories.

## 4. Importance priority

Only select stories that meaningfully matter for someone investing with a
multi-year horizon across stocks, real estate, crypto, and alternative
assets. Prioritize, in this order:

1. Structural/regulatory changes (central bank policy shifts, new
   regulation, tax law changes affecting investors).
2. Confirmed material developments (not rumors) with a clear, durable
   effect on valuations or portfolio allocation.
3. Genuine short-term opportunities with a clear catalyst and expiry.
4. Broad sentiment/trend pieces, only if no stronger story exists in that
   category that day.

Exclude: routine daily price commentary, single-stock trading tips without
long-term relevance, clickbait, and anything you cannot corroborate from at
least one credible source.

## 5. Cross-day deduplication

For every candidate story, construct a normalized event record:

```json
{
  "event_id": "short-kebab-case-id",
  "reported_date": "YYYY-MM-DD",
  "event_date": "YYYY-MM-DD",
  "category": "one of the six categories above",
  "entities": ["Company/Asset/Country names"],
  "event_type": "e.g. rate_decision, regulatory_change, earnings, protocol_upgrade",
  "canonical_summary": "One sentence describing the underlying event.",
  "source_url": "https://...",
  "material_update_of": "event_id of an earlier related story, or null"
}
```

Compare each candidate against `state/reported-stories.json` by underlying
event, not by headline or URL. Do not report a story again unless there is a
genuinely material new development (official confirmation after a rumor,
a rate decision that was previously only "expected," a regulatory
approval/rejection, materially revised guidance, etc.). If it is a material
update of a prior story, set `material_update_of` to the earlier event's
`event_id`.

## 6. Story selection limits

- No more than 15 selected stories total.
- Every selected story appears exactly once in the narration.
- Prefer breadth across categories over depth in one category, unless one
  category genuinely dominates the day's news.

## 7. Podcast narration constraints

Write `input/narration.txt` as a single plain-text script, no Markdown, no
URLs, no citations, no bullet points, no headers. It should read naturally
when spoken aloud.

- One or two short sentences per story.
- Lead with the concrete development, not framing or context.
- No filler introduction ("Welcome to..." / "Today we'll look at...") and no
  closing remarks — start directly with the first story and end after the
  last one.
- Group stories in a sensible spoken order (e.g., macro context first, then
  equities, real estate, crypto, alternatives, then opportunistic
  situations last) — do not simply list them by internal category order if a
  different order reads more naturally.
- Target 700–1,100 words. Absolute maximum: 1,200 words. The workflow will
  reject the episode outright if this is exceeded — do not pad or run long.

## 8. Story-history update

Append today's selected events (using the normalized record format above) to
`state/reported-stories.json`. Keep the file as a single JSON array. Prune
entries older than 120 days. Validate that the file is syntactically valid
JSON before writing it.

## 9. Permitted repository files

You may only create, modify, or delete these files:

- `input/narration.txt`
- `state/reported-stories.json`

Do not modify anything under `.github/`, `scripts/`, `docs/`, or any other
path. If you believe a workflow or script change is genuinely needed, report
that in your final summary instead of making the change yourself.

## 10. Push to claude/podcast-state

1. Stage only the two permitted files.
2. Commit with a message like `Update narration and story state for
   YYYY-MM-DD`.
3. Push to `origin claude/podcast-state`.
4. If the push fails due to a conflict, re-fetch, re-merge, and retry once.
   If it still fails, report the failure clearly instead of forcing the push.

## 11. Final summary

At the end of the run, report clearly:

- How many stories were selected, by category.
- Any category that had zero qualifying stories today, and why.
- The final narration word count.
- Confirmation that the push to `claude/podcast-state` succeeded.
- Anything unusual (malformed state file, research gaps, low source
  confidence on any story).
