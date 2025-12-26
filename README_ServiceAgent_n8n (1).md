# ServiceAgent Take-Home Assessment — Daily Weather Automation (n8n)

This repository contains a **production-style n8n automation** that:
1) Fetches daily weather data from a public weather API  
2) Generates a formatted weather summary  
3) Detects basic weather alerts (precipitation / heat / frost / none)  
4) Persists results to Supabase (**two tables: `weather_logs` + `workflow_logs`**)  
5) Sends a **separate daily email per city** with the summary + alert  

It also includes a **second “error notification” workflow** that triggers on workflow errors and emails the user with actionable debugging context.

> Assignment requirements & recommended schema are based on the provided ServiceAgent assessment brief. fileciteturn0file0

---

## What’s Included

### Workflows (2)
- **Main workflow (Daily Weather Automation)**  
  - Triggered daily by **Cron**
  - Supports **multiple cities** (loop)
  - Sends **one email per city**
  - Inserts a log row per city to `weather_logs`
  - Inserts an operational log row to `workflow_logs`
  - Uses retry logic for API robustness

- **Error workflow (Workflow Error Notifier)**  
  - Triggered by **Event / Error Trigger**
  - Sends an email with:
    - workflow name
    - failing node
    - error message + stack (where available)
    - run timestamp
    - execution / run URL (where available)
  - Also writes an operational log to `workflow_logs` (so failures are visible even if email fails)

### Bonus Points Completed (explicit)
✅ City configurable  
✅ Retry logic (Weather API request): **3 retries** with **1000 ms** delay  
✅ Loop over multiple cities (Split Cities → per-city processing)  
✅ Additional weather metrics included (beyond minimum required)  

---

## Tech Stack
- **n8n** (workflow automation)
- **OpenWeatherMap** (or similar public weather API)
- **Supabase** (Postgres + REST API)
- **SMTP Email** (Gmail SMTP recommended for reliability)
- Tested using **Google Chrome** (desktop)

---

## 1) High-Level Architecture

### A) Main Workflow — “Daily Weather Automation”
**Pipeline per run:**

1. **Cron Trigger** (daily at fixed time)
2. **Config (Set node)**  
   - contains default units, thresholds, and the city list
3. **Split Cities**  
   - converts the city list into individual items
4. **For each city**:
   - Weather API fetch (**HTTP Request**, with retries)
   - Normalize + extract fields (**Code node**)
   - Apply alert rules (**Switch/IF or Code node**)
   - Format email subject/body (**Code node**)
   - Insert into Supabase `weather_logs` (**HTTP Request** with *Continue On Fail*)
   - Send email (SMTP)
5. **Workflow-level logging**
   - Insert an operational record into `workflow_logs` describing the run status (success/partial/failure), counts, and any errors.

> Result: You get **one DB log row per city** in `weather_logs` + **one email per city**, and a single operational record in `workflow_logs` to audit the run.

### B) Error Workflow — “Workflow Error Notifier”
1. **Error Trigger (Event Trigger)**  
   - fires when any workflow fails (configured to watch main workflow)
2. **Insert into `workflow_logs`** (so errors are persisted even if downstream actions fail)
3. **Format error email** (Code/Set)
4. **Send error email** (SMTP)

> Result: If something breaks, you receive a **separate error email** and the failure is also stored in `workflow_logs`.

---

## 2) Supabase Data Model (Two Tables)

This implementation uses **two tables**:

### Table 1 — `weather_logs` (required)
Stores the daily weather output **per city**. fileciteturn0file0

#### Recommended schema (used)
```sql
create table if not exists public.weather_logs (
  id bigserial primary key,
  run_at timestamptz not null default now(),
  city text not null,
  temperature double precision,
  temperature_unit text,
  condition text,
  humidity int,
  wind_speed double precision,
  alert_type text,
  raw_response jsonb
);
```

#### What we store
- **Derived fields**: easy filtering/reporting (temp, humidity, wind, alert_type)
- **raw_response**: full API payload (debugging, audits, future enrichment)

---

### Table 2 — `workflow_logs` (operational logging)
Stores workflow-level operational telemetry so you can audit runs over time, troubleshoot errors, and prove reliability.

#### Suggested schema (used)
```sql
create table if not exists public.workflow_logs (
  id bigserial primary key,
  run_at timestamptz not null default now(),
  workflow_name text not null,
  run_type text not null, -- e.g., "daily_weather" or "error_notifier"
  status text not null,   -- e.g., "success", "partial", "failed"
  cities_requested int,
  cities_processed int,
  errors_count int,
  error_summary text,
  execution_id text,
  raw_context jsonb
);
```

#### Why we added this table
- **Visibility:** quickly see which days/cities ran successfully
- **Auditing:** trace when failures started
- **Debugging:** store execution metadata + error summaries
- **Ops maturity:** mirrors client workflows that require observability

---

## 3) Alert Rules (Required)

Alert type is computed using simple rules fileciteturn0file0:

### Precipitation alert
If condition contains any of:
- `rain`, `snow`, `drizzle`, `storm`, `thunder`

### Heat alert
- If `temperature > 32°C`

### Frost alert
- If `temperature < 0°C`

### Otherwise
- `none`

> Note: We run matching case-insensitively and use a normalized `condition_text` to avoid missing alerts.

---

## 4) Additional Weather Metrics (Bonus)

Beyond the required fields, we also extract and include (when available):
- `feels_like`
- `temp_min` / `temp_max`
- `pressure`
- `clouds` (%)
- `visibility`
- `sunrise` / `sunset` (timezone aware)

These are:
- Included in the **email body**
- Included in `raw_response` in Supabase for future reporting

---

## 5) Multi-City Handling (Bonus)

### Configured City List
In the **Config node**, define a list of cities:
```json
{
  "cities": ["Chicago", "New York", "San Francisco"],
  "units": "metric",
  "temp_unit": "C",
  "heat_c": 32,
  "frost_c": 0
}
```

### Split Cities Node
We use a “Split Cities” step to convert the `cities` array into individual items (one per city).  
This allows downstream nodes to run **per city** (clean and scalable).

### Separate Email Per City (This Implementation)
Because each city becomes its own item:
- the email subject/body is built for that city
- the email node runs once per item  
➡️ **You receive separate daily emails for every configured city.**

---

## 6) Reliability & Error Handling (Bonus + Production Practices)

### A) Weather API retries (Bonus)
Weather API HTTP Request is configured with:
- **max retries = 3**
- **retry delay = 1000 ms**

This reduces transient network/API flake.

### B) Supabase insert “Continue On Fail” (weather logs)
The Supabase insert into `weather_logs` is configured to **continue even if the insert fails**:
- The workflow still sends the weather email (so user gets the daily update)
- Failure details are available in the execution logs
- The run result is captured in `workflow_logs` for observability

### C) Error workflow notifications (separate workflow)
If any step errors (API auth, parsing, SMTP, etc.), the **Error Trigger workflow** sends an email and also writes to `workflow_logs`:
- workflow name
- failing node
- error message
- timestamp
- relevant execution metadata (where available)

---

## 7) Email Format

### Subject (required)
`Daily Weather for <CITY> – <YYYY-MM-DD>` fileciteturn0file0

### Body
Contains:
- City + date
- Temperature (°C)
- Condition
- Humidity
- Wind
- Alert line
- Additional metrics section (bonus)

HTML or plaintext is acceptable; this implementation supports either.

---

## 8) Setup Instructions

### Step 1 — Create API keys / credentials

#### Weather API (OpenWeatherMap recommended)
- Create an account and generate an API key
- We use the “Current Weather” endpoint:
  - `https://api.openweathermap.org/data/2.5/weather`

Query parameters per city:
- `q=<CITY>`
- `appid=<API_KEY>`
- `units=metric` (for °C)

#### Supabase
- Create a Supabase project
- Create `weather_logs` and `workflow_logs` tables using the SQL above
- Collect:
  - `SUPABASE_URL` (e.g., `https://xxxxx.supabase.co`)
  - `SUPABASE_SERVICE_ROLE_KEY` (recommended for this take-home)

#### SMTP Email (reliable choice)
- Gmail SMTP recommended:
  - host: `smtp.gmail.com`
  - port: `465` (SSL) or `587` (TLS)
  - username: your Gmail address
  - password: **App Password** (not your normal password)

---

### Step 2 — Import workflows into n8n
1. Open n8n in **Google Chrome**
2. Go to **Workflows → Import from File**
3. Import both exported workflow JSON files:
   - `daily_weather_automation.json`
   - `workflow_error_notifier.json`

---

### Step 3 — Configure node credentials

#### Weather API HTTP Request node
- Add the API key as a query param (`appid`) or via n8n credential variable.
- Ensure `units=metric` for Celsius.

#### Supabase Insert nodes (HTTP Request)
- Weather logs URL:
  - `{{SUPABASE_URL}}/rest/v1/weather_logs`
- Workflow logs URL:
  - `{{SUPABASE_URL}}/rest/v1/workflow_logs`
- Headers (both):
  - `apikey: {{SUPABASE_KEY}}`
  - `Authorization: Bearer {{SUPABASE_KEY}}`
  - `Content-Type: application/json`
  - `Prefer: return=representation`

#### Email node (SMTP)
- Configure SMTP credentials
- Set “To” recipient(s)
- Ensure From name/address is set clearly (e.g., “Daily Weather Bot”)

---

### Step 4 — Configure cities (bonus)
In the **Config** node:
- Edit `cities` array to your preferred list
- Default example includes **Chicago**

---

### Step 5 — Activate workflows
- Activate **Main workflow**
- Activate **Error workflow**

---

## 9) How to Run & Test (Detailed)

### A) Smoke test (single city)
1. In Config node, set `cities = ["Chicago"]`
2. Click **Execute Workflow**
3. Confirm:
   - Weather API fetch succeeded
   - alert_type computed
   - `weather_logs` row inserted
   - Email received for Chicago
   - `workflow_logs` has a run record

### B) Multi-city test (bonus)
1. Set `cities = ["Chicago", "New York", "San Francisco"]`
2. Execute workflow
3. Confirm:
   - 3 iterations
   - 3 rows in `weather_logs` (one per city)
   - 1 row in `workflow_logs` summarizing the run
   - 3 emails (one per city)

### C) Validate Supabase logs
In Supabase:
- Open `weather_logs` and confirm city-level rows
- Open `workflow_logs` and confirm run-level rows with status and counts

### D) Validate alert rules
To validate each alert type, temporarily adjust thresholds:
- For **heat**: set `heat_c = -100` to force heat alert
- For **frost**: set `frost_c = 100` to force frost alert
- For **precipitation**: set city to one currently raining (or temporarily override condition in normalization code)

Then execute and confirm email + DB reflect expected `alert_type`.

### E) Test retries (Weather API)
To confirm retry behavior:
1. Temporarily use an invalid API key OR throttle network
2. Execute workflow
3. Observe the Weather API node attempt up to 3 retries with ~1000 ms delay
4. Restore the correct API key

### F) Test “Supabase insert fails but email still sends” (weather logs)
To simulate DB failure while still sending emails:
1. Temporarily set an incorrect `SUPABASE_KEY`
2. Ensure the `weather_logs` insert node has **Continue On Fail** enabled
3. Execute workflow
4. Confirm:
   - `weather_logs` insert shows failure
   - Emails still send for each city
   - `workflow_logs` records a **partial** or **failed** status with error summary
5. Restore correct key

### G) Test Error Workflow (Event Trigger)
To trigger an error and confirm error notifications:
1. Force a hard failure (e.g., remove required API key; disable the Weather API node; or throw an error in a Code node)
2. Execute the main workflow
3. Confirm:
   - main workflow fails
   - error workflow triggers
   - `workflow_logs` receives an error record
   - an error email is received with debugging context

---

## 10) Submission Checklist (Deliverables)
- [ ] Exported n8n workflow JSON: `daily_weather_automation.json`
- [ ] Exported error workflow JSON: `workflow_error_notifier.json`
- [ ] `README.md` (this file)
- [ ] Screenshot(s) of workflow(s) (PNG)

---

## 11) Troubleshooting

### No emails received
- Verify SMTP credentials
- If Gmail: confirm App Password and port/SSL/TLS settings
- Check spam folder

### Supabase insert failing
- Confirm URL ends with `/rest/v1/weather_logs` or `/rest/v1/workflow_logs`
- Confirm `apikey` + `Authorization: Bearer ...`
- If using anon key: ensure RLS policies allow inserts

### Wrong alert_type
- Confirm units are `metric` (°C)
- Confirm condition normalization is case-insensitive
- Confirm thresholds in Config node match intended values
