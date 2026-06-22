# klaviyo-grafana-insights-engine
A custom analytics engine that pulls Klaviyo data into Supabase, processes it through SQL models and scheduled functions, and visualizes it in Grafana. Delivers automated, scalable insights for multi-brand email performance, revenue, and daily ops
# Insights Engine

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
MIT 

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
The process is automated end-to-end, meaninf no sheets, exports, or manual steps.

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

# .env
SUPABASE_URL="https://your-project-id.supabase.co"
SUPABASE_SERVICE_ROLE_KEY="your-super-secret-service-role-key"

KLAVIYO_API_KEY="your-klaviyo-api-key"
KLAVIYO_API_VERSION="2024-10-15" # Check Klaviyo docs for latest version

# Cron schedule (Supabase uses standard cron syntax)
# Runs every day at 2:00 AM UTC
CRON_SCHEDULE="0 2 * * *"

---

### 1. Environment Variables (`.env`)
Create a file named `.env` in your project root. Fill in your actual credentials.

```bash
# .env
SUPABASE_URL="https://your-project-id.supabase.co"
SUPABASE_SERVICE_ROLE_KEY="your-super-secret-service-role-key"

KLAVIYO_API_KEY="your-klaviyo-api-key"
KLAVIYO_API_VERSION="2024-10-15" # Check Klaviyo docs for latest version

# Cron schedule (Supabase uses standard cron syntax)
# Runs every day at 2:00 AM UTC
CRON_SCHEDULE="0 2 * * *"
```

### 2. Database Schema & SQL Models (`supabase/migrations/001_init_klaviyo_tables.sql`)
Run this in your Supabase SQL Editor. It creates the raw storage table and the optimized view for Grafana.

```sql
-- 1. Create the raw metrics table
CREATE TABLE IF NOT EXISTS daily_client_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  client_id TEXT NOT NULL,
  client_name TEXT,
  date DATE NOT NULL,
  metrics JSONB NOT NULL, -- Stores the full normalized JSON payload
  revenue NUMERIC DEFAULT 0,
  orders_count INT DEFAULT 0,
  unsubscribes INT DEFAULT 0,
  opens INT DEFAULT 0,
  clicks INT DEFAULT 0,
  UNIQUE(client_id, date)
);

-- 2. Create an index for fast Grafana filtering
CREATE INDEX IF NOT EXISTS idx_metrics_date_client 
ON daily_client_metrics(date, client_id);

-- 3. Create a SQL View for Grafana (Aggregated & Cleaned)
-- This view handles any future logic like rolling averages
CREATE OR REPLACE VIEW grafana_klaviyo_dashboard AS
SELECT 
  date,
  client_id,
  client_name,
  revenue,
  orders_count,
  (revenue / NULLIF(orders_count, 0)) AS avg_order_value,
  unsubscribes,
  opens,
  clicks,
  (clicks::float / NULLIF(opens, 0)) AS click_through_rate,
  (unsubscribes::float / NULLIF(opens, 0)) AS unsubscribe_rate
FROM daily_client_metrics
ORDER BY date DESC, client_id;

-- 4. Enable Row Level Security (Optional but recommended)
ALTER TABLE daily_client_metrics ENABLE ROW LEVEL SECURITY;
ALTER VIEW grafana_klaviyo_dashboard ENABLE ROW LEVEL SECURITY;

-- Policy: Allow Service Role (Grafana/Functions) full access
CREATE POLICY "Allow service role full access" ON daily_client_metrics
  FOR ALL USING (auth.uid() IS NOT NULL) -- Adjust based on your auth setup
  WITH CHECK (auth.uid() IS NOT NULL);
```

### 3. Supabase Edge Function (`supabase/functions/sync-klaviyo/index.ts`)
Create this file in your Supabase project (via CLI or Dashboard). This handles the ETL logic.

```typescript
// supabase/functions/sync-klaviyo/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

serve(async (req) => {
  // 1. Setup Supabase Client
  const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
  const supabaseKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
  const supabase = createClient(supabaseUrl, supabaseKey);

  // 2. Get Klaviyo API Key
  const apiKey = Deno.env.get("KLAVIYO_API_KEY");
  if (!apiKey) return new Response("Missing API Key", { status: 500 });

  try {
    // 3. Fetch Data from Klaviyo (Example: Profile Metrics)
    // Note: You may need to paginate or adjust the endpoint based on your needs
    const response = await fetch(
      `https://a.klaviyo.com/api/metrics?filter[client_id]=your-client-id`, 
      {
        headers: {
          "Authorization": `Klaviyo-API-Key ${apiKey}`,
          "revision": Deno.env.get("KLAVIYO_API_VERSION") || "2024-10-15"
        }
      }
    );

    if (!response.ok) throw new Error(`Klaviyo API Error: ${response.statusText}`);
    
    const data = await response.json();
    
    // 4. Normalize & Upsert Data
    // Loop through data and format for SQL
    const records = data.data.map((item: any) => ({
      client_id: item.attributes.client_id,
      client_name: item.attributes.name,
      date: new Date().toISOString().split('T')[0], // Today
      metrics: item.attributes,
      revenue: item.attributes.revenue || 0,
      orders_count: item.attributes.orders_count || 0,
      unsubscribes: item.attributes.unsubscribes || 0,
      opens: item.attributes.opens || 0,
      clicks: item.attributes.clicks || 0,
    }));

    // Upsert into Supabase
    const { error } = await supabase
      .from('daily_client_metrics')
      .upsert(records, { onConflict: 'client_id,date' });

    if (error) throw error;

    return new Response(JSON.stringify({ success: true, records: records.length }), {
      headers: { "Content-Type": "application/json" },
    });

  } catch (error) {
    console.error(error);
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
});
```

### 4. Cron Job Configuration (`supabase/config.toml`)
If you are using the Supabase CLI, ensure your `config.toml` has the cron trigger set up for this function.

```toml
[functions.sync-klaviyo]
verify_jwt = false 

# Add this to your local dev or production deployment scheduler
# Note: In production, you often use a separate cron service (like GitHub Actions)
# or the Supabase Cron extension if available in your plan.
```

*Note: For the "Run Cron" step in production, the easiest way without extra tools is to set up a **GitHub Action** that hits your Edge Function URL once a day, or use the **Supabase Cron** feature if enabled in your dashboard under "Database Extensions" -> `pg_cron`.*

### 5. Grafana Data Source Setup
1.  **Add Data Source:** Select **PostgreSQL**.
2.  **Connection Details:**
    *   Host: `db.your-project-id.supabase.co`
    *   Port: `5432`
    *   Database: `postgres`
    *   User: `postgres`
    *   Password: Your Supabase DB password (not the API key).
3.  **Query:**
    *   Select `grafana_klaviyo_dashboard` from the table dropdown.
    *   Use the filters in the query builder for `client_id` and `date`.

### How to Deploy
1.  **Initialize:** `npx supabase init`
2.  **Link:** `npx supabase link --project-ref your-project-id`
3.  **Migrate DB:** `npx supabase db push`
4.  **Deploy Function:** `npx supabase functions deploy sync-klaviyo --env-file .env`
5.  **Test:** Manually trigger the function via the Supabase Dashboard to ensure data flows.

Once the data is flowing, you can build your Grafana dashboards using the `grafana_klaviyo_dashboard` view.
