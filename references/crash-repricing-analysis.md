# Single-Event Crash / Repricing Analysis

Use case: a stock drops sharply (roughly >=10 %) in one session or a tight window tied to a known event (earnings plus capex guidance, election, rating change, regulation). Goal is a structured analysis with concrete levels - not a forecast. Every report ends with "This is not investment advice."

## 1. Resolve the ticker (before any data work)
DE-ISIN / ADR IDs are frequently just a local listing of a US symbol. Establish the primary (usually US) identifier first and take all price data from that exchange, where liquidity and the primary source live; the secondary identifier serves only as a target-market reference.

Verification gate: confirm the resolved combination against at least two independent sources - compare company name/ISIN on each page before adopting it. Summary tables under identical tickers may list a completely different company (symbol collision across exchanges); a name mismatch between source and resolution aborts data work for that page; do not adopt its values. Locally cached price JSON only re-enters a workflow when its last date field proves the current fetch round - otherwise pull fresh.

## 2. Acquire the data (keyless, reputable)
- Daily OHLCV >=3 years: Stooq CSV `https://stooq.com/q/d/l/?s=<TICKER>.us&i=d` -> saved locally as JSON (`<ticker>_series3y.json`, one row per `[date, open, high, low, close, volume]`). The file additionally contains the still-running trading day's partial final row - usable intraday print: compute all metrics with and without that bar and label results "provisional".
- Detect source degradation early and switch sources: a `content-type: text/html` response to an API URL, a JS challenge, or HTTP 403/429 means a blocked source - immediately fall back (Stooq -> stockanalysis.com -> Finviz export) and never re-probe the same URL inside a loop. One fresh probe per metric block per source; a bot-wall loop eats iterations without yield.
- Yahoo's quoteSummary/chart APIs rate-limit after just a few calls from one host - do not hammer them; Stooq + stockanalysis.com suffice keyless for this workflow.
- Fundamentals: the statistics page of `stockanalysis.com/stocks/<ticker>/`, extracted via web extraction; fallback is the Finviz snapshot (`quote.ashx?t=<TICKER>` with a desktop user-agent, no cookies needed - partial coverage such as market cap, margins, liquidity ratios). Label/value scraping on those snapshot pages is only half-reliable: labels repeat in hidden duplicate rows and values sit between nested tags; take the first stable match per field and mark missing fields "unverified" instead of deriving them.
- Earnings events: never quote naked numbers from press headlines - fetch the official IR release (PDF) as primary source and extract to plain text locally; segment tables and forward guidance live there. JS-heavy business articles return mostly navigation when parsed by regex/HTML: support headlines keyless via Google News RSS (`https://news.google.com/rss/search?q=<Company>+<Keyword>&hl=en-US&gl=US` gives title + timestamp per item) instead of forcing the article parse.
- Take price data for each metric block from exactly one listing: dual-listed symbols (Xetra DE + US ADR) are measured in different currencies depending on source - mixed currencies in the same table invalidate gap, support, and Fibonacci calculations.

## 3. Compute event-day metrics
- Event-day table: previous close, open-gap %, day's high/low, closing move (%), volume in multiples of the 60-day average (excess volume = panic / liquidity signal).
- RSI(14) per Wilder (before / after), ATR(14), annualized volatility over the last 252 sessions, MAs at 14/20/50/200.
- Cite post-crash MA values side by side with pre-crash values - a -48% bar distorts short-term averages; without the counter-value the number is misleading on its own.
- Compute every indicator (RSI/ATR/MA/Fib/volume factor) through a saved scratch script and execute it, not as ad-hoc inline one-liners or multi-line code in chat: past certain length translation errors and typos accumulate there silently - and can produce wrong numbers with no error message; scripts are rerunnable, debuggable, checkable.

## 4. Build the level map
- Fibonacci retracement over the full pre-event rally (rally low -> ATH), levels at 23.6 / 38.2 / 50 / 61.8 %: compute them upward from the leg's low side (`level = low + f * (high - low)`) and relate them to the current price - not "fib present" checkbox logic. Computing it as `high - f * span` puts all nearby retracement levels too high near ATH (sign-convention error). Sanity check: on a down-leg every retracement level must lie strictly between high and low.
- Multi-leg declines / stepwise crashes: additionally mark leg-Fib over the most recent relevant swing (last local top -> current bottom) in the same level table, labelling each level with its swing ("Fib 38 % of the October leg") - leg-specific levels are more useful for next-step recomputation than the whole-rally fib that frames them.
- Name old panic clusters from previous crisis events within the 3-year history as an extreme support zone (the last bastion before air pocket).
- Resistance from below: crash high/close band -> prior MA50/MA200 -> earlier consolidation edges.
- Gap midpoint (half the distance from prior close to event-day low) as first counter-target when the break fails.

## 5. Fundamental cross-checks before committing to a spot price
PE ttm/forward, revenue/earnings ttm, margins, free cash flow -> FCF yield at current price, cash vs debt, dividend, short % / days-to-cover (distinguishes sentiment repricing from forced selling), analyst consensus + price target relative to spot, and the most recent rating action.
Extreme per-stock values (e.g., net cash far above market capitalization) always carry an explicit "VERIFY" marker and are presented with a warning, never flatly asserted - regulatory reserves/liabilities frequently do not appear in simplified overview pages.

### Earnings variant: finding the trigger
Check number events against the official release before judging "bad numbers" - headlines usually name only the consequence (price down -x %), not the driver:
1. Compare reported revenue/operating income and EPS to the known consensus range: core beat with a price drop = disappointment in guidance, segment detail, or capex - search inside the release for forward guidance (next-quarter revenue outlook + annual capex target) and the slowest-growing segment; that is usually the real trigger.
2. Numbers published after close: the move appears as the *following* day's pre-market gap - the event table gets prior close -> pre-market move, with publication-day intraday action only as context. RSI/MA computations without the post-event close state so explicitly (otherwise they read ~50/neutral instead of the true post-event posture).
3. Capex-driven moves: expose P/FCF-TTM as headline metric - a raised capex target drops FCF almost proportionally; a 10 % price drop at unchanged current-year guidance plus an announced jump next year reprices on exactly that risk.

## 6. Scenarios + timing stance
- Three named scenarios (base/bull/bear), each tied to concrete levels or triggers: break of event-day low = continuation risk toward the old cluster; hold above the crash close + RSI stabilization = first recovery leg up to the gap midpoint.
- Record a timing finding: no instant entry at the event-day low while no reversal candle has confirmed - for high-volatility events wait 1-3 days or enter in tranches.

## Report Layout (structured output)
English, Markdown with header and numbered sections: 1) Event-day numbers, 2) Technical picture, 3) Fundamentals, 4) Verdict & scenarios. All metrics in tables with concrete figures; sources with timestamps at the end. A candlestick chart (~last 180 sessions) as PNG with labelled horizontal lines for key levels is sent alongside (`MEDIA:<path>`). Chart-rendering chain: system Python first, then the project/per-profile venv checked for matplotlib; if both lack it, do not install in the reporting flow - instead use a pure PIL hand renderer (PIL is available in system Python): white canvas, per-session wick line high->low + body rectangle min(open,close)->max(open,close), bull/bear colour coding, y-axis with ~5 labels from window min/max, labelled dashed level lines, date ticks every ~10-15 bars, title row on top, then `img.save(path.png)`. Window padding: enough range for high/low plus the key levels; force an in-image label (not a line) for key levels outside the window, otherwise the relevant zone is lost.

Every report ends with: **"This is not investment advice."**
