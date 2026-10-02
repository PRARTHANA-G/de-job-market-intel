# German Data Job Market Intelligence Platform

Daily pipeline that ingests data and analytics job postings in Germany, models them in a
medallion lakehouse, tracks skill demand over time, and answers questions about the market
through a RAG layer with citations.

Status: phase 1 in progress. Full README with architecture diagram comes in phase 5.

## Data source

Job postings come from the [Arbeitnow](https://www.arbeitnow.com) public job board API.
Raw data is stored privately and is never committed to this repository.

## Architecture decisions

See [docs/adr](docs/adr).
