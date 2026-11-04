# EU Electricity Analytics

An analytics pipeline that analyses electricity prices across European bidding zones and their relationship with wind and solar generation.

## Problem

Electricity prices in Europe change every 15 minutes, and they can even go negative. Prices depend on many factors, especially solar and wind generation. Understanding these patterns helps:

- **EV owners and households** find the best time to charge or run energy-heavy appliances.
- **Energy suppliers** design better dynamic tariffs.
- **Grid operators** anticipate when supply and demand are most out of balance.

## Business questions

1. When and why do electricity prices go negative, and how is that linked to solar and wind generation?
2. What's the cheapest time of day to run energy-heavy things like EV charging, by bidding zone and season?
3. How has the renewable share of generation changed over time, and does a higher share lower prices?

## Data source

The data comes from the [ENTSO-E Transparency Platform](https://transparency.entsoe.eu/):

- **Day-ahead prices** (document type `A44`), fetched with a GET request per bidding zone, identified by its EIC code.
- **Solar and wind generation data** (planned).

See [docs/data_exploration.md](docs/data_exploration.md) for details.

## Architecture

Coming soon: ingestion → warehouse → dbt → orchestration → dashboard.

## Status

- [x] Data exploration
- [ ] Design
- [ ] Ingestion