# Reddit Sentiment Check for Stocks (time window as requested)

Task class: "What's the sentiment around $TICKER on Reddit within the last 24 h / week / month?" - evaluate evidence only, no forecasts. Zero hits is itself a valid result and must be reported as such.

## Procedure
1. **Build the ticker matrix first** (after ticker resolution ISIN/ADR -> primary symbol, see section 1 of `crash-repricing-analysis.md`):
   - `$SYMBOL`, exact company name ("XP Incorporated"), short forms ("XP Inc"), known sub-brand / product names of the company, and "EXCHANGE SYMBOL". Short community-light tickers need several variants so something actually hits.
2. **Query routes in this order - max 1-2 calls per route, then move on:**
   - `https://old.reddit.com/r/<sub>/search/?q=<query>&restrict_sr=on&sort=new&t=day|week` via urllib or equivalent with a browser user-agent: scoped search in r/wallstreetbets, r/stocks; plus the sitewide `https://old.reddit.com/search/?q=...&sort=new&t=...`.
   - Alternative / parallel: the www Reddit search page via browser navigation + snapshot (same parameters). If old-Reddit blocks server-side, keep the other route open - never thrash on one endpoint.
   - Sitewide backup through your local metasearch instance if you run one (`<your-searxng>/search?q=...&time_range=<requested>&format=json`), keeping only `reddit.com` URLs; pause between variants and avoid a doubled `/search/` segment in the path.
   - `.json` / search-json endpoints throttle hard from single hosts: one attempt per endpoint, then switch route on error. Plain JSON can also block after several single-thread fetches; load that thread's HTML through the browser instead.
3. **Separate empty from blocked:** "no discussion" may only conclude from HTTP 200 + a parsable search page without result blocks ("No results"). On HTTP errors or an unparsable page, the route is down - probe another route; never turn a failure signal into a conclusion.
4. **Microcap fallback:** when no dedicated thread exists for thin names, check the daily recap threads: r/StockMarketMovers "MARKET RECAP - <weekday>, <date>" (Mon-Fri). Load today's recap HTML and grep with word boundaries so similar symbol prefixes don't slip in; mention there counts as a hit but is classified explicitly as weak signal.
5. **Collect and weigh evidence:** per hit, timestamp, subreddit, title, link, score/comment count; derive per-quote sentiment (positive/neutral/negative) - never extrapolate beyond the quotes.

## Output Format (verdict first)
- One-line verdict up front ("No relevant discussion found ..." / "Sentiment clearly negative ..."), then the evidence list as table/bullets (date | subreddit | title/link), then a framing block. Signal not found = explicit classification (e.g., neutral-uninterested, no community exists) instead of silent skipping - with a coverage note about which routes actually ran.
- No invented posts: when nothing is there, that absence *is* the answer, not a gap to fill.

## Pitfalls (generic)
- Word boundaries + variant breadth forgotten -> false "no hits" despite active discussion (short tickers like XP collide with other symbols; include suffixes and brand names).
- Time window not mapped 1:1 on the user's request (24 h = `t=day`) -> wrong signal.
- Treating an empty search path as equivalent to a rejected conclusion - run the full route matrix before "no discussion" stands.
