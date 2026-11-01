---
name: technical-stock-analysis
version: 1.0.0
author: Felix Morocutti (https://github.com/FelixM2011)
description: >
  Candlestick and Point & Figure crash analysis plus Reddit community sentiment for
  stocks, using keyless public data sources (Stooq, stockanalysis.com, Finviz, old Reddit).
category: finance
---

# Technical Stock Analysis (Candle-Pattern + P&F Workflow, Keyless Data)

## When to Use

Tasks in the "technical-analysis-for-a-single-stock" class: implement, correct, or explain candlestick- and Point & Figure (P&F)-pattern definitions whose source of record is a dedicated technical-analysis book used locally (`TechnischeAktienanalyse.pdf`), on top of keyless live price data.

## What It Is

A local project with these scripts (kept in your agent workspace, not shipped here):
- `data_fetch.py` — keyless daily-bar fetch via the Yahoo Finance Chart API (`range=a/j/y/w/m`; query params are always URL-encoded). Note: the chart/quote endpoints rate-limit after a few calls from one host — do not hammer them; Stooq + stockanalysis.com cover most needs without any key.
- `tak_analyze.py` — two stdout sections: **"Candle / Technical Analysis"** (signal checks at fixed lookbacks such as 2, 8 and longer ranges depending on `--range`) and **"Point & Figure Market Theory"**: box-size calculation plus exact buy/sell signal rules derived from the book. A live CLI run against a real symbol is the daily ground truth for both sections.

## Standing Rules (always on)

1. **The book is the source of record.** Before changing or explaining any pattern definition, verify the original passage in the local PDF by keyword search; do not substitute general candlestick lore. Book-specific quirks are intentional and internally consistent — do not "modernize" a term to match other authors (e.g., in this book's hanging-man rule the long upper shadow is what defines the pattern).
2. **No placeholder blocks in production code.** A condition that is always true followed by `pass` hides the real logic, which usually sits directly below it and is already correct: grep for the genuine rule before removing anything; afterwards the live output must stay bit-identical or differ only for a stated design reason.
3. **Trend gating (`trend_dir`).** Strict majority of daily steps AND the net direction must agree to yield `1`/`-1`, otherwise the state is `0`. A close-compare vs. previous day alone is not enough (zig-zag with small offsets misfires as an uptrend). In state `0` the signal check must report "no signal" — never guess a direction.
4. **P&F: only start a new line after ≥2 consecutive closes outside the box** at the *daily* closing price; an intra-bar touch counts neither as a retest nor as a new entry (an off-by-one `>=1` silently kills virtually every real breakout).
5. **Harden keyless API responses.** The chart payload's result field can be `null` for unknown symbols → return an empty list/dict. Treat HTTP 404 for wrong symbols separately from TypeError/IndexError in the retry/fallback path.
6. **P&F signals: exactly one signal per trend leg — never per box step.** A buy fires only when the current uptrend breaks above the highest point of the last *completed* up-leg (book, ch. 6); a sell is symmetric against the low of the old down-leg. The reference level is set only once that leg has been reversed; the initial trend (first column) deliberately yields no signal. Emitted per box step this produces dozens of duplicated signals in one trend and contradicts the book.
7. **Synthetic test series stay strictly positive** and near realistic price orders; if the box size is a percentage of the reference price, guard the price-versus-zero case — otherwise the incremental while-loops (`box_abs += / -=`) over a negative box height spin forever. Defensive `abs()` at derivation time.

## Validating After Every Change

1. Run the unit-test battery: book examples plus edge series (zig-zag, monotone up/down). Separate signals in tests by exact signal names — e.g., a substring match on "BUY" would also count inside "SELL" strings and corrupt buy counts; use `startswith` with the full word instead.
2. Live run against a real symbol: re-compare TAK/P&F sections in a diff — for data unaffected by your change, output must stay bit-identical or differ only for an explained design reason (every deviation gets a reason).
3. For hanging Python terminal runs: `timeout N python -u script.py > /tmp/out.txt 2>&1`, then read the file — SIGKILL drops the stdout buffer; built-in per-line flush reveals exactly where processing stopped, which is how infinite while-loops get localized cheaply.

## House Style for This Task Class
- Answer user itemizations in their order and with their numbering (1./2./3.), point by point; proof = short live-run excerpt, never guessing.
- Never invent pattern detections: if today's price path contains no signal, an empty signal field is the intended regression proof.
- No fabricated numbers, page references, or prices — everything comes from a live run or a cited book passage that was actually checked.

## Extended Analysis Cosm (references/)

- **Single-event crash / repricing analysis:** `references/crash-repricing-analysis.md` — ticker resolution, keyless data path, event-day metrics, level map, fundamental cross-check, structured report layout.
- **Reddit sentiment check for stocks:** `references/reddit-sentiment-check.md` — per-ticker query matrix, reliable read-only routes (old Reddit HTML, browser, local metasearch), empty-vs-blocked disambiguation, microcap fallback via recap threads.

## Local Project Files (not part of this repo)

The live implementation (`data_fetch.py`, `tak_analyze.py`) and the source book PDF stay in your own workspace — they contain local paths and are deliberately not published here. The rules above plus the two references fully define the methodology; re-implement or adapt them to your environment: any symbol, any keyless OHLCV source with comparable rate behaviour works as a stand-in for `data_fetch.py`.
