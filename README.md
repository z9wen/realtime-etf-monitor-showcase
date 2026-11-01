# Real-Time ETF Market Monitor

A C++20 terminal application that streams US ETF market data from the official Interactive Brokers TWS API, keeps a durable local history, and turns it into real-time market context: VWAP, relative volume, relative strength and order-flow metrics for more than 30 index, sector and industry ETFs.

It is decision support only and never submits orders.

**Status:** working personal project, built in August 2026.
**Source:** private. This repository describes the data pipeline and the engineering decisions. I'm happy to walk through the code in an interview.

## At a glance

| | |
| --- | --- |
| Language | C++20 with CMake, compiled with `-Wall -Wextra -Wpedantic` |
| Size | about 17k lines excluding tests |
| Data source | Official IBKR TWS API, with separate client connections for historical backfill and live streaming |
| Storage | Local CSV cache partitioned by symbol and year, atomic writes, resumable completion manifest |
| Tests | 17 test executables with 59 test cases, run through CTest |
| Interface | Five-page terminal dashboard that adapts to the terminal size and redraws about once a second |

## Data pipeline

```mermaid
flowchart LR
    tws[TWS or IB Gateway] -- historical connection --> hist[Historical backfill<br/>session calendar from IBKR,<br/>monthly requests, 5 in flight]
    tws -- live connection --> live[Live quotes<br/>and trade ticks]
    hist --> cache[(Local cache<br/>symbol/year partitions,<br/>atomic writes, manifest)]
    cache --> merge[Merge and deduplicate<br/>by timestamp]
    live --> merge
    merge --> bars[Completed-bar sync<br/>1-minute to 5-minute,<br/>cross-series alignment]
    bars --> metrics[Metrics<br/>VWAP, relative volume,<br/>relative strength, order flow]
    metrics --> ui[Terminal dashboard]
    cache --> replay[Replay over<br/>cached history]
```

## Data correctness

- **Only completed bars drive the metrics.** The bar that is still forming is excluded, and comparisons across instruments use the newest timestamp that every series has completed. Nothing compares a finished bar with a half-built one.
- **Idempotent merges.** Live and cached bars are normalised by timestamp, sorted and deduplicated, so reconnects and repeated refreshes never double count.
- **The session calendar comes from the broker.** Trading sessions are taken from IBKR's own trading schedule instead of being guessed from weekdays. An empty response is never treated as a holiday: a scheduled session that returns no data is reported as missing and stays incomplete.
- **New York time throughout.** Regular and extended hours are anchored separately, with explicit time-zone conversion for display.
- **The feed type is visible.** The dashboard shows whether IBKR is sending real-time or delayed data, as reported by the API, so delayed data is never mistaken for live.

## Backfill and failure handling

- **Multi-year backfill that resumes where it stopped.** Missing sessions are grouped into calendar-month requests with five in flight. Each finished month is saved immediately and recorded in a completion manifest, so an interrupted run continues from the last finished month instead of starting over.
- **Errors are classified, not retried blindly.** Pacing responses are retried. A dropped connection retries the same batch, so a backfill never ends half-populated. IBKR's market-data line-capacity error is surfaced instead, because reconnecting can't create more lines. Explicit no-data responses are recorded separately for review, and no synthetic bars are ever written.
- **Atomic file writes.** Cache partitions, manifests and schedules are written to a temporary file and renamed into place, so a crash can't leave a half-written cache.
- **Verified migration.** An older flat cache layout was migrated to year partitions. Every timestamp is verified in its new partition before the original files are moved to a backup folder.
- **Graceful shutdown.** SIGINT and SIGTERM are handled through an async-signal-safe flag that the main loop checks.

## Testing and replay

- Tests cover bar alignment, timeframe conversion, request planning, error classification, cache storage and migration, indicators, order-flow aggregation and terminal input decoding.
- A replay mode runs the same analytics over cached history, including estimated transaction costs, to check behaviour before relying on it live.

## Research in progress

A machine-learning research branch is still experimental.

## Contact

[LinkedIn](https://www.linkedin.com/in/zewen-l-531569327/) · [GitHub](https://github.com/z9wen)
