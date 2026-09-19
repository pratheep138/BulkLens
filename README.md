# BulkLens

BulkLens is a market-intelligence workspace for investigating bulk deals, client history, and stock conviction. Trendlyne can be one of the deal sources, alongside official NSE and BSE filings.

## Run it

Open `index.html` directly in a browser. No build tool or dependency installation is required.

With Node.js installed, run `npm install` once and then `npm start`. The dashboard is available at `http://localhost:4173`.

The running server exposes `GET /api/trendlyne`, which fetches the current Trendlyne bulk/block-deal page and returns normalized rows. It also exposes `GET /api/trendlyne/client?name=...`, which loads the client-specific Trendlyne history page. The dashboard calls the latest-deals endpoint on startup and when **Refresh intelligence** is selected; clicking any client opens its earlier-deal history, including same-day deals.

Client analysis is available at `GET /api/trendlyne/client-analysis?name=...`. It excludes rows explicitly marked intraday and same-day buy/sell round trips from carried-position calculations, matches remaining lots FIFO, fetches current prices from the linked Trendlyne stock pages, and reports open quantity, average buy price, current price, tenure, unrealized profit/loss, return percentage, and a provisional 0-100 trust score. The score is a research heuristic, not investment advice; it should be extended with verified disclosures and corporate-action adjustments before production use.

## Load a deal export

Use **Load export** in the Deal sources panel and choose a CSV or JSON file. The importer recognizes common column names for date, company, symbol, client, buy/sell, value, and optional trust score. For example:

```csv
Date,Company,Symbol,Client Name,Buy/Sell,Value
10 Sep 2026,Solar Industries,SOLARINDS,Mirae Asset MF,BUY,₹42.6M
```

Imported records replace the demo rows in the deal tables and remain local to the browser session.

## Product direction

- **Bulk deal feed:** ingest Trendlyne exports or licensed feeds, plus NSE/BSE deal records, and normalize company, client, side, quantity, value, date, exchange, and source.
- **Client intelligence:** score counterparties using realized returns, disclosure consistency, holding period, repeat behavior, and insider-risk flags.
- **Stock conviction:** combine deal flow, client quality, price context, liquidity, and fundamentals into a 0-100 score.
- **Lens Agent:** explain each signal with the evidence used, confidence, and sources checked. The current browser demo uses a representative dataset; production ingestion should run server-side and use an approved Trendlyne export/API, licensed market-data feed, or official exchange filings. Do not scrape protected pages or bypass access controls.

## Suggested next backend boundary

Create an API that exposes `GET /deals`, `GET /clients/:id/history`, `GET /stocks/:symbol/rating`, and `POST /ingestion/:source/sync`. Keep ratings explainable by returning both the score and a list of weighted factors. The ingestion response should retain `source`, `sourceRecordId`, `retrievedAt`, and `rawPayloadHash` for provenance and deduplication.
