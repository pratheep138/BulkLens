# BulkLens

BulkLens is a market-intelligence workspace for investigating bulk deals, block deals, client history, and stock conviction. Trendlyne is the current live source, alongside planned support for official NSE and BSE filings. The product is an analytical research tool, not an investment-advice system.

## Run it

Open `index.html` directly in a browser. No build tool or dependency installation is required.

With Node.js installed, run `npm install` once and then `npm start`. The dashboard is available at `http://localhost:4173`.

The running server exposes these routes:

- `GET /api/trendlyne`: fetches the current Trendlyne bulk/block-deal window and returns normalized rows.
- `GET /api/trendlyne/client?name=...`: loads a client's historical bulk/block-deal rows.
- `GET /api/trendlyne/client-analysis?name=...`: calculates one client's exclusions, suspected matched activity, FIFO positions, returns, and trust score.
- `GET /api/trendlyne/client-summaries`: calculates summaries for all clients in the latest feed.

The dashboard loads the latest feed at startup and when **Refresh intelligence** is selected. Client summaries start only after deal rows exist; this avoids the previous race where cards appeared but trust scores stayed blank. A fresh Trendlyne snapshot clears the server-side client-analysis cache so an earlier day's score is not reused for the new window.

## Deal interpretation

Trendlyne bulk/block disclosures show execution details, not the complete beneficial-owner or demat-account relationship. The data cannot prove that two client names are a joint account, related entities, or one intraday trader. `Intraday = No` means the source did not explicitly mark the row intraday; it does not prove that the activity was a long-term investment.

The application keeps all raw rows visible in deal history. It changes only the position and conviction analysis:

1. Rows explicitly marked intraday are excluded from carried-position calculations.
2. Same-client, same-stock, same-day buy and sell quantities are matched using `min(total buys, total sells)`. Only the matched quantity is excluded; unmatched excess quantity remains available for FIFO analysis.
3. Cross-client activity is flagged as suspected matched institutional activity only when date, symbol, exchange, price, quantity, and opposite action all match, and the client names are distinct after normalization.
4. Matching is an analytical warning, not confirmation of intraday trading, a joint account, or a transfer.

Example interpretations:

- Tata Motors: BOFA Securities Europe SA buying and Bank of America National Association selling the exact same quantity at the exact same price and time window is shown as **Suspected matched intraday / transfer**. It should not be treated as a clean bullish client signal.
- Entero Healthcare: HDFC Mutual Fund's buy quantity does not exactly match the Prasid Uno Family Trust sell quantities. Those rows remain genuine bulk/block buy and sell flow and are not suppressed as a matched transfer.

## Client scores and signals

Trust scores are provisional research scores based on available historical positions, realized returns, current prices, profitable-position rate, average return, positive-return magnitude, exclusions, and holding tenure. They are signed:

- Positive score: the available calculated history is directionally positive.
- Negative score: the available calculated history is directionally negative.
- `—`: the client has no position with a calculable return yet, for example historical holdings without a current price. This is unrated, not a score of zero.

The score must not silently fall back to `70` or another default. A real negative result is preserved as negative; zero is not used as the generic "processed" state.

Client-card visual states are intentionally high-signal:

- `LONG-TERM`: green.
- `SHORT-TERM`: red.
- `INTRADAY-HEAVY`: stronger red with a thicker border.
- `RECENT SOLD STOCKS`: red container because selling is a risk/reduction signal.
- `SUSPECTED MATCHED INTRADAY / TRANSFER`: red warning container and border.
- `1 MONTH BUY SIGNALS`: normally green. If the client's signed history is negative, only the stock name is highlighted red. This is an averaging-down warning: the client may be adding to a historically losing position, which is not a clean entry signal by itself.

The visual warning does not override the raw transaction history; it only makes the relevant risk easier to identify.

## Loading status

The pipeline status is fixed at the top of the dashboard. During processing it uses a high-contrast amber state with a spinner and the message `Processing client histories and ranking`. When summaries finish, it changes to the green ready state. If the summary request fails, it changes to an unavailable state rather than presenting stale or default scores.

## Load a deal export

Use **Load export** in the Deal sources panel and choose a CSV or JSON file. The importer recognizes common column names for date, company, symbol, client, buy/sell, value, quantity, exchange, deal type, and optional trust score. Imported rows are browser-local. Imported rows do not currently run through the server-side Trendlyne client-history analysis automatically; use the live Trendlyne path or extend the ingestion boundary before relying on imported rows for scored client history.

```csv
Date,Company,Symbol,Client Name,Buy/Sell,Quantity,Value
10 Sep 2026,Solar Industries,SOLARINDS,Mirae Asset MF,BUY,125000,INR 42.6M
```

Imported records replace the current browser deal rows and remain local to the browser session. The current CSV parser is intentionally lightweight and expects simple comma-separated fields; use JSON when the export contains commas inside quoted values or richer metadata.

## Current implementation notes

- `server.js` owns Trendlyne fetching, HTML row normalization, price lookup, client analysis, FIFO matching, score calculation, and API responses.
- `app.js` owns feed loading, client-card rendering, score propagation into the deal table, search, file import, and the top pipeline status.
- `styles.css` owns the green/amber/red risk language and the fixed top processing status.
- `clientAnalysisCache` is keyed by client and invalidated after a fresh live deal snapshot.
- The client-card summary request must not run before the live `deals` array is populated. The current startup handoff waits for that condition.
- Raw deal history remains visible even when rows are excluded from carried-position analysis.

## Product direction

- **Bulk deal feed:** ingest Trendlyne exports or licensed feeds, plus NSE/BSE deal records, and normalize company, client, side, quantity, value, date, exchange, and source.
- **Client intelligence:** score counterparties using realized returns, disclosure consistency, holding period, repeat behavior, explicit intraday rows, suspected matched activity, and averaging-down warnings.
- **Stock conviction:** combine deal flow, client quality, price context, liquidity, and fundamentals into a signed research signal rather than treating every purchase as bullish.
- **Lens Agent:** explain each signal with the evidence used, confidence, and sources checked. The current browser demo uses a representative dataset; production ingestion should run server-side and use an approved Trendlyne export/API, licensed market-data feed, or official exchange filings. Do not scrape protected pages or bypass access controls.

## Known limitations and next steps

- Create an API that exposes `GET /deals`, `GET /clients/:id/history`, `GET /stocks/:symbol/rating`, and `POST /ingestion/:source/sync`.
- Keep ratings explainable by returning the score, score direction, and a list of weighted factors.
- Retain `source`, `sourceRecordId`, `retrievedAt`, and `rawPayloadHash` for provenance and deduplication.
- Add verified corporate-action adjustments before comparing historical prices and returns.
- Add a proper exchange or licensed feed for beneficial-owner and account-relationship evidence; do not infer joint accounts from matching names alone.
- Add automated tests for exact cross-client matches, partial quantity matches, unmatched excess quantities, negative scores, unrated clients, and the client-summary loading race.

## Session handoff

When continuing work, start by reading this README and checking `server.js`, `app.js`, and `styles.css`. The most important behavioral rules are:

1. Never convert an unscored client into a default score.
2. Never remove raw deal rows just because analysis suspects intraday or matched activity.
3. Exclude only matched quantities from carried positions; preserve unmatched quantities.
4. Treat exact cross-client matches as suspected transfer/intraday activity, not confirmed fact.
5. Keep negative-history one-month purchases visibly distinct at the stock-name level.
6. Keep processing status prominent while summaries are loading so blank scores are not mistaken for final scores.
