# Health Dashboard — Data Sync
# Run this prompt in Cowork (Claude Desktop) whenever you want to sync data.
# Connectors required: Oura MCP, GetFast MCP

---

## ARCHITECTURE OVERVIEW (read this if you have no prior context)

This system has three components that work together:

1. **Google Sheet** (ID: 1p0WzqzwyYNFEAVUvENlttlAjdAjn__bpzjLurTSo4oY)
   The Sheet is the live data backend. It has these tabs:
   - `oura` — readiness, sleep scores, HRV, RHR, steps, activity calories (from Oura MCP)
   - `sleep_detail` — nightly sleep stage hours, HRV, RHR (from Oura MCP)
   - `breathing` — breathing disturbances per hour during sleep, with `source` column.
     Oura rows: source="Oura", populated by this sync.
     Apple Watch rows are NOT written here — AW data lives in hardcoded `AH_BD` in the HTML.
     The dashboard reads this tab at load time and splits rows by source:
     Oura rows → `AH_BD_OURA` dict, Apple Watch rows → merged into `AH_BD`.
   - `strava` — workout activities (from GetFast/Strava MCP)
   - `apple_health` — daily AH metrics: weight, SpO2, VO2max, resting energy (entered from CSV exports)
   - `weight` — individual weight readings with date (entered from CSV exports or scale)
   - `meta` — sync timestamps written by Claude after each sync run
   Row 1 = headers, Row 2 = units/descriptions (skip both when reading). Data starts row 3.

2. **Google Apps Script webhook**
   URL: https://script.google.com/macros/s/AKfycbywzN7JAqHaOUt7JU8f_UOeMJ7g3lwH_lqnKALE4f6QDoU2W0WGYVnXgnPesJx4KObF/exec
   Accepts POST requests with {"tab": "<name>", "rows": [...]}. Upserts by date column —
   existing rows are updated in place, new dates are appended. Safe to re-run.

3. **HTML dashboard** (https://connorm-19.github.io/tools/health_dashboard_v1.html)
   A static file hosted on GitHub Pages. It fetches live data from the Sheet on every page load
   using the Sheets API (API key: AIzaSyBs8NEvKaYXLom0a6ixwM2Q1dFo2rEtN10). The HTML merges
   Sheet data on top of hardcoded historical data and falls back to hardcoded data if the fetch
   fails. The dashboard is NOT updated by running this sync — it reads the Sheet automatically.

   Key hardcoded data objects in the HTML (updated manually when needed):
   - `RAW.weight` — historical weight readings going back to 2023
   - `AH_BD` — Apple Watch breathing disturbances by date (Oct 2024–present, 460 days)
   - `AH_BD_OURA` — Oura Ring BDI full history (Oct 2024–Apr 2026, 539 days, hardcoded)
     Both AH_BD and AH_BD_OURA are updated by providing a standalone CSV to Claude.
     The Sheet `breathing` tab (Oura rows) also merges into AH_BD_OURA at runtime,
     so recent synced data fills in automatically without HTML edits.
   - `AH_SLEEP` — sleep stage hours
   - `LIVE_AH` — recent Apple Health daily metrics (Mar–Apr 2026)

**Your job in this sync prompt:** populate the Sheet via the webhook (Steps 1–3 below).
**You never touch the HTML file** when running this sync.
**You never create new Sheet tabs** — the tabs listed above are fixed and permanent.

**CRITICAL — NO HTML PATCHES DURING SYNC:**
Never write NEW_AH2, NEW_WT2, NEW_AH, NEW_OURA, NEW_STRAVA, or any live patch blocks to
the HTML during a sync run. Apple Health CSV data (resting_kcal, active_kcal, rhr, weight)
belongs in the Sheet `apple_health` and `weight` tabs — not hardcoded into the HTML. If an
AH CSV is attached alongside this sync prompt, follow the WHEN AN APPLE HEALTH CSV IS
PROVIDED section and post it to the Sheet after confirmation. Never add it to the HTML.
The Sheet is the live backend. The HTML reads it automatically on every page load.
No HTML edits and no git push are needed as part of a sync.

---

## WHEN AN APPLE HEALTH CSV IS PROVIDED

Sometimes Connor will attach an Apple Health export CSV alongside this prompt (filename like
`HealthAutoExport-YYYY-MM-DD-YYYY-MM-DD.csv`). If a CSV is attached:

**Do NOT post Apple Health data to the webhook automatically.**
Apple Health data goes into the `apple_health` and `weight` tabs of the Sheet, but these tabs
are populated manually by Connor — not by this sync prompt.

**Instead, do this:**
1. Parse the CSV to extract the following fields, aggregated per day (sum energy, average others):
   - date (from Date/Time column, take the date part only)
   - weight_lbs (Weight (kg) × 2.20462, rounded to 1 decimal — use average if multiple readings)
   - resting_kcal (Resting Energy (kJ) ÷ 4.184, summed across hourly rows)
   - active_kcal (Active Energy (kJ) ÷ 4.184, summed across hourly rows)
   - spo2 (Blood Oxygen Saturation (%), averaged)
   - hrv_ms (Heart Rate Variability (ms), averaged)
   - rhr (Resting Heart Rate (count/min), averaged)
   - vo2max (VO2 Max (ml/(kg·min)), averaged — often sparse)
   - sleep_total_h (Sleep Analysis [Total] (hr), summed)
   - sleep_deep_h (Sleep Analysis [Deep] (hr), summed)
   - sleep_rem_h (Sleep Analysis [REM] (hr), summed)
   - breathing_dist (Breathing Disturbances (count) — already per hour, average)

2. Present the parsed data as a clean table in your response so Connor can review it.

3. Tell Connor: "This data is ready for manual entry into the apple_health and weight tabs of
   the Sheet. Would you like me to post it to the webhook?" Then wait for confirmation before
   posting anything from the Apple Health CSV.

4. If Connor confirms, POST to tab "apple_health" and "weight" using the same webhook pattern
   as Step 3 below. The weight tab row format is: {"date": "YYYY-MM-DD", "weight_lbs": <number>}.
   The apple_health tab row format matches the fields above.

This two-step approach (parse → confirm → post) prevents accidental overwrites of manually
entered data and avoids creating spurious rows.

---

## COMPUTE SESSION & TIMING

The POST step uses GetFast compute (httpx). The session takes 30-90 seconds to start.
Follow this exact sequence:

1. Call GetFast:compute_start_session
2. Call GetFast:compute_session_status every 15 seconds until status == "active"
3. Once status == "active", immediately call GetFast:compute_execute with the full POST code
4. Do NOT let the session sit idle after activation — execute within 30 seconds or it may drop

If compute_execute returns "Session is not active", start over from step 1.
Do all data collection (Oura + Strava + AH parsing) BEFORE starting the compute session,
so you can fire the POST immediately once the session is active.

---

## DUPLICATE DATES — NO ISSUE

The webhook upserts by date. If a date already exists in the Sheet, its row is overwritten
with the new values. If it doesn't exist, a new row is appended. Running the sync twice
on the same day is safe — it will update existing rows, not create duplicates.

---

## APPLE HEALTH TAB — REQUIRED

The HTML dashboard DOES read the apple_health tab on every page load and merges it into
RAW.ah (Apple Health daily metrics). The tab must exist in the Sheet with these columns:

  date, weight_lbs, resting_kcal, active_kcal, spo2, hrv_ms, rhr, vo2max,
  sleep_total_h, sleep_deep_h, sleep_rem_h, breathing_dist

Row 1 = headers, Row 2 = units/descriptions (e.g. "lbs", "kcal", "%", etc.), data from row 3.
If the tab doesn't exist yet, create it in the Sheet manually with those headers before syncing.

---

STEP 1 — Pull Oura data (last 14 days through today)

Use the Oura MCP tools for the date range (today minus 14 days) through today:

- oura:get_daily_readiness — readiness_score, contributors (hrv_balance, rhr_score,
  recovery_index, body_temperature), temperature_deviation, temperature_trend_deviation
- oura:get_daily_sleep — sleep_score, total sleep, deep, rem, core hours, latency
  NOTE: oura:get_daily_sleep returns a summary score only, not stage durations.
  Use oura:get_sleep (not get_daily_sleep) to get the full sleep object with
  deep_sleep_duration, rem_sleep_duration, light_sleep_duration, latency, average_breath,
  average_hrv. When multiple sleep periods exist for a day (long_sleep + nap), use only
  the record with type="long_sleep" for the primary sleep metrics.
- oura:get_daily_activity — activity_score, active_calories, steps
- oura:get_daily_spo2 — spo2_percentage.average, breathing_disturbance_index

NOTE: get_daily_activity can return very large payloads and may be saved to disk.
If saved to disk, read with jq:
  jq '[.[].text | fromjson | .data[]] | map({
    date: .day,
    activity_score: .score,
    activity_cal_active: .active_calories,
    activity_steps: .steps
  })' <path-to-file>

Combine all Oura fields by date into this format for the `oura` tab:
{
  "date": "YYYY-MM-DD",
  "readiness_score": <number>,
  "readiness_label": <"optimal"|"good"|"fair"|"pay_attention">,
  "sleep_score": <number>,
  "hrv": <number, ms>,
  "rhr": <number, bpm>,
  "temp_deviation": <number, °C>,
  "temp_trend_deviation": <number, °C>,
  "resp_rate": <number, rpm>,
  "spo2_avg": <number, %>,
  "activity_score": <number>,
  "activity_cal_active": <number, kcal>,
  "activity_steps": <number>,
  "sleep_total_h": <number>,
  "sleep_deep_h": <number>,
  "sleep_rem_h": <number>,
  "sleep_core_h": <number>,
  "sleep_latency_min": <number>
}

Readiness label thresholds: 85–100 = "optimal", 70–84 = "good", 60–69 = "fair", <60 = "pay_attention"
HRV and RHR come from get_daily_readiness contributors (hrv_balance, resting_heart_rate).
SpO2 avg comes from get_daily_spo2 (spo2_percentage.average).

Also produce rows for the `sleep_detail` tab:
{
  "date": "YYYY-MM-DD",
  "total_h": <number>,
  "deep_h": <number>,
  "rem_h": <number>,
  "core_h": <number>,
  "resp_rate_rpm": <number>,
  "source": "Oura"
}

Also produce rows for the `breathing` tab (skip date if data unavailable):
{
  "date": "YYYY-MM-DD",
  "disturbances_per_hr": <number>,
  "source": "Oura"
}

NOTE: The `source` field is critical. The dashboard splits the breathing tab by source on load:
- source="Oura" rows → `AH_BD_OURA` (shown as Oura Ring bars in the breathing chart)
- source="Apple Watch" rows → `AH_BD` (shown as Apple Watch bars)
Always use exactly "Oura" as the source string for Oura BDI rows.

---

STEP 2 — Pull Strava data (last 14 days through today)

Use GetFast:list_activities with start_date, end_date, source: strava.

For each activity produce one row for the `strava` tab:
{
  "date": "YYYY-MM-DD",
  "name": <string>,
  "type": <"Run"|"Ride"|"Hike"|"Walk"|"AlpineSki"|"Other">,
  "distance_km": <number, 2 decimal places>,
  "duration_min": <number, 1 decimal place>,
  "pace_min_km": <string "M:SS" for runs, null otherwise>,
  "avg_hr": <number or null>,
  "max_hr": <number or null>,
  "elev_gain_m": <number or null>
}

Activity type mapping: running→Run, cycling→Ride, hiking→Hike, walking→Walk,
alpine_skiing→AlpineSki, all others→Other.
pace_min_km from GetFast is decimal (e.g. 5.5) — convert to "M:SS" (e.g. "5:30").
Multiple activities on the same day = multiple rows.

---

STEP 3 — POST all data to webhook via GetFast compute

Use GetFast:compute_execute to POST. Never use bash_tool for HTTP — it runs in a sandboxed
environment without internet access. Always use compute_execute for network requests.

IMPORTANT: The compute environment has httpx installed, NOT requests. Always use httpx.
Always call compute_start_session first, then check compute_session_status and wait for
status=="active" before calling compute_execute.

CRITICAL — POST ALL TABS IN A SINGLE compute_execute CALL:
The compute session drops silently if there is any delay between activation and execution,
or if multiple sequential calls are made. Always combine all tab POSTs (oura, sleep_detail,
breathing, strava, meta) into one single compute_execute call using a helper function.
Never split across multiple compute_execute calls. Use timeout=45 (not 30) to handle
slower webhook responses.

Pattern:
  import httpx, datetime
  from zoneinfo import ZoneInfo

  WEBHOOK = "https://script.google.com/macros/s/AKfycbywzN7JAqHaOUt7JU8f_UOeMJ7g3lwH_lqnKALE4f6QDoU2W0WGYVnXgnPesJx4KObF/exec"

  def post(tab, rows):
      r = httpx.post(WEBHOOK, json={"tab": tab, "rows": rows}, timeout=45, follow_redirects=True)
      print(f"{tab}: {r.status_code} {r.text[:100]}")

  post("oura", oura_rows)
  post("sleep_detail", sleep_detail_rows)
  post("breathing", breathing_rows)
  post("strava", strava_rows)
  post("meta", [{
      "date": datetime.date.today().isoformat(),
      "synced_at": datetime.datetime.now(ZoneInfo("America/Vancouver")).isoformat(timespec="seconds"),
      "synced_by": "Claude.ai"
  }])

Post to exactly these tabs (no others, no new tabs):
  "oura", "sleep_detail", "breathing", "strava", "meta"

---

STEP 4 — Summary

Report:
- Rows posted per tab (appended vs updated from webhook response)
- Any errors
- Key metrics for the most recent date: readiness score, sleep score, HRV, RHR, workouts

---

NOTES:
- Leave fields null if unavailable — never guess
- Dates always YYYY-MM-DD
- The 14-day window fills any gaps automatically — safe to run every few days
- Do NOT fetch Apple Health or weight data from connectors — only Oura and Strava
- Do NOT create new Sheet tabs — the tab names above are fixed
- The HTML dashboard reads the Sheet automatically — no HTML changes needed after syncing
- REMINDER: If Claude ever proposes writing NEW_AH, NEW_WT, or any JavaScript data patches
  to the HTML file as part of a sync, that is a mistake — reject it and ask Claude to post
  the data to the Sheet instead
- The `weight` Sheet tab uses column header `wt_lbs` (not `weight_lbs`) — always use `wt_lbs` as
  the key when posting weight rows: {"date": "YYYY-MM-DD", "wt_lbs": <number>}
- The breathing chart on the Sleep tab shows Apple Watch as colour-coded bars and Oura Ring
  as a purple line with coloured dots overlaid. Both use teal/amber/red severity thresholds.
  Apple Watch data (AH_BD) is hardcoded — update by providing a standalone AW breathing CSV.
  Oura BDI data (AH_BD_OURA) is also hardcoded for full history, AND flows through the Sheet
  `breathing` tab for recent data — so the last 14 days update automatically each sync run.
  The chart shows the last 45 days by default.

## UPDATING HARDCODED BREATHING DATA IN THE HTML

The breathing chart has two hardcoded dicts in health_dashboard_v1.html that need updating
when Connor provides new CSV exports.

**Apple Watch (`AH_BD`) — standalone AW breathing CSV:**
- Filename pattern: `Breathing_Disturbances-YYYY-MM-DD-YYYY-MM-DD.csv`
- Column format: Date/Time, Breathing Disturbances (count), Sources
- One reading per night; average any dates with multiple readings
- Replace the `const AH_BD = {...};` line in the HTML
- Do NOT post Apple Watch breathing data to the Sheet breathing tab

**Oura Ring (`AH_BD_OURA`) — full Oura BDI history:**
- Pull via `oura:get_daily_spo2` for the full date range (Oct 2024–present)
- Deduplicate by day (average if multiple entries per day), skip null values
- Replace the `const AH_BD_OURA = {...};` line in the HTML
- Note: recent Oura BDI also syncs automatically via the Sheet `breathing` tab each run,
  so HTML updates are only needed to extend the full historical baseline

**After either HTML update:**
- git add health_dashboard_v1.html
- git commit -m "update breathing disturbance history"
- git push
- The compute session / webhook are NOT involved — this is a direct HTML file edit only

## ALTERNATE POSTING METHOD IF COMPUTE SESSION FAILS

NOTE: The browser console fallback does NOT work from claude.ai — CORS blocks it from
external origins. It only works if you open the dashboard URL in your own browser and
run fetch() from the DevTools console on that page directly. Claude cannot trigger this
remotely.

If compute keeps failing, the correct fix is: collect all data first, start a fresh
compute session, wait for active status, then POST everything in one single
compute_execute call (see STEP 3 above). Do not attempt multiple calls.

For all html changes, if explicitly asked, the file is located at /Users/connormccracken/Documents/tools/health_dashboard_v1.html. If making changes to html file, let me know and also tell me the terminal command to push these updates live.
