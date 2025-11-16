# klaviyo-insights-engine
A custom analytics engine that pulls Klaviyo data into Supabase, processes it through SQL models and scheduled functions, and visualizes it in Grafana. Delivers automated, scalable insights for multi-brand email performance, revenue, and daily ops
# Insights Engine

## Overview
Short description (350-char).  
Who it’s for.  
Why it exists.  
Core capabilities.

## Architecture
High-level diagram (ASCII).  
Layers: Klaviyo → Supabase → SQL models → Cron → Grafana.

## How It Works
1. Data flow  
2. Functions  
3. SQL models  
4. Dashboards  
5. Extensibility

## Installation & Setup
Prereqs  
Environment variables  
Deploy function  
Run cron  
Connect Grafana  
Test pipeline

## SQL Models
List tables + purpose  
Key metrics  
Notes for extensibility

## Dashboards
Workspace structure  
Filters  
Panels overview

## Roadmap
Planned enhancements  
Performance improvements  
Future data sources

## Notes
Known Klaviyo quirks  
Design decisions  
Why no-code tools were dropped

## License
MIT or whatever you choose.

                    +-------------------+
                    |     Klaviyo API   |
                    |  (metrics, events)|
                    +---------+---------+
                              |
                              | JSON payloads
                              v
                   +----------+-----------+
                   |   Supabase Edge      |
                   |   Function (ETL)     |
                   | - API fetch          |
                   | - normalization       |
                   | - SQL-safe transform |
                   +----------+-----------+
                              |
                              | structured data
                              v
               +--------------+----------------+
               |        Supabase DB            |
               | - daily_client_metrics        |
               | - (planned) campaign_metrics |
               | - SQL models, views           |
               +--------------+----------------+
                              |
                              | SQL queries
                              v
                   +----------+-----------+
                   |     Grafana          |
                   | - dashboards         |
                   | - filters            |
                   | - team workspaces    |
                   +----------------------+
How It Works

1. Data Flow

A Supabase Edge Function pulls Klaviyo metrics once per day. It fetches raw JSON from the API, normalizes it, and writes structured rows into Postgres.
The process is automated end-to-end — no Sheets, no exports, no manual steps.

2. ETL Logic

The ETL is fully coded for stability and clarity.
It handles:
• direct Klaviyo API requests
• inconsistent field names
• JSON transformation
• SQL-safe normalization
• upserts into clean tables

No-code tools were intentionally skipped because they break on case sensitivity, API quirks, and multi-client scale.

3. Storage Layer

Supabase Postgres stores:
• daily metrics per client
• normalized tables
• aggregated SQL models
• indexes for fast filtering

The schema is simple, predictable, and built to support multi-brand reporting without duplication or drift.

4. Automation

A Supabase cron job triggers the function daily.
The function processes all active client API keys, refreshes metrics, and logs results for debugging and monitoring.

Everything runs on schedule without intervention.

5. SQL Models

All transformations and intelligence sit in SQL:
• rolling windows
• trend analysis
• ratios
• health metrics
• flexible joins across brands and channels

New logic is added through SQL models or small function updates — not spreadsheets.

6. Visualization Layer (Grafana)

Grafana reads directly from Supabase.
Dashboards are dynamic, filterable by client and time range, and designed for both ops and exec use.
Teams can have isolated or shared dashboards depending on their needs.

The system grows easily: add a client key → data populates automatically.

7. Extensibility

The architecture is modular:
• new data sources can be added (Shopify, GA4, backend events)
• new dashboards can be created per team
• deeper analytics and aggregations can be layered on without touching the core

This is a proper analytics layer, not a patchwork of tools.

Architected and built by Natasa Šebek
