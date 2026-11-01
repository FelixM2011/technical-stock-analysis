# Technical Stock Analysis

Agent skill (Hermes / ClawHub format) for single-stock technical analysis: candle-pattern + Point & Figure signal logic derived from a local reference book, on top of **keyless** public data sources (Stooq CSV, stockanalysis.com, Finviz snapshots, old Reddit). No API keys required.

## What's here

| File | Role |
|---|---|
| `SKILL.md` | Skill front matter + standing rules (trend gating, P&F box/line semantics, signal-per-leg rule, hardening keyless APIs) and the validation battery to run after every change |
| `references/crash-repricing-analysis.md` | Full single-event crash/repricing workflow: ticker resolution gate -> keyless data acquisition -> event-day metrics -> Fib/MA level map -> fundamental cross-checks -> scenarios + timing stance + report layout |
| `references/reddit-sentiment-check.md` | Reddit sentiment protocol: per-ticker query matrix, reliable read-only routes (old Reddit HTML / browser / local metasearch), empty-vs-blocked disambiguation, microcap recap-thread fallback |

## Not in this repo (deliberate)

- `data_fetch.py`, `tak_analyze.py` - the live implementation scripts. They reference machine-local paths and stay in your workspace; `SKILL.md` fully specifies their interface (range parameters, output sections, hardening rules), so an equivalent implementation works against any symbol from Stooq/Yahoo keyless endpoints.
- The source book (`TechnischeAktienanalyse.pdf`) - local-only reference for the exact pattern definitions.

## How to adapt locally

```text
your-skills/technical-stock-analysis/    <- this repo (SKILL.md + references/)
your-workspace/data_fetch.py             <- keyless OHLCV fetcher  (interface: symbols, ranges a/j/y/w/m)
your-workspace/tak_analyze.py            <- TAK + P&F analyzer      (-k "SYMBOL" --range j|y|...)
```

Validation battery from `SKILL.md` to run after any change: unit series (edge cases + textbook examples), live run diff against a real symbol, and signal-name matching by exact leading word rather than raw substring (a naive check would match "BUY" inside "SELL").

## License

MIT - see `LICENSE`. Every generated stock report produced for this methodology carries the disclaimer *"This is not investment advice."* This skill and its outputs are research tooling only.
