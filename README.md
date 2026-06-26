# Route SLA Intelligence Platform
### ET · MOE School Bus Operations · UAE · 2026

> An end-to-end intelligent operations platform combining data engineering, BI engineering, AI-generated insights, and real-time traffic correlation — built to reduce SLA breaches across 22 stations and hundreds of MOE school bus routes.

Disclaimer: This project uses mock/sample data for demonstration purposes only. No real, confidential, or client data has been included.

---

## The Problem

ET operates MOE school bus routes across the UAE under strict SLA agreements:
- **60 minutes** maximum for regular MOE routes
- **45 minutes** maximum for KG routes

When routes consistently exceed these thresholds, students are delayed, drivers are under pressure, and contracts are at risk. The challenge isn't detecting breaches — it's understanding *why* they're happening and *which ones* need immediate action versus routine monitoring.

Standard dashboards show you what went wrong. This platform tells you why — and what to do about it.

---

## What makes this different

Most BI dashboards stop at visualisation. This platform is built across four engineering disciplines:

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENGINEERING LAYERS                           │
├──────────────────┬──────────────────────────────────────────────┤
│ Data Engineering │ Oracle ADW → Power BI Dataflow → DAX model   │
│ BI Engineering   │ Custom HTML visuals · DAX-generated JS · SLA │
│ AI Engineering   │ Claude API · multi-turn narratives · forecast│
│ Traffic Layer    │ Google Maps Traffic API · congestion index   │
└──────────────────┴──────────────────────────────────────────────┘
```

---

## Architecture

```
Oracle ADW (Route source system)
        │
        ▼
Power BI Dataflow (scheduled refresh)
        │
        ├── DAX Measures
        │     ├── SLA Breach Detection (consecutive 5-day logic)
        │     ├── HTML/JS Visual Generation (dynamic strings)
        │     ├── Station Summary Header (auto-fill, filter-aware)
        │     ├── Active Filters Summary (live slicer state)
        │     └── Trend Measures (weekly + monthly)
        │
        ├── HTML Content Visual (custom Power BI visual)
        │     └── DAX string → full HTML + CSS + JS in iframe
        │
        ├── Python AI Pipeline (post-refresh trigger)
        │     ├── Reads breach_export.csv from Power BI
        │     ├── Calls Claude claude-sonnet-4-6 per station
        │     ├── Multi-turn conversation → executive summary
        │     └── Writes breach_narratives.csv → linked table
        │
        └── Traffic Correlation Pipeline
              ├── Google Maps Traffic API → congestion index per route
              ├── Joins to breach data on Station + Date + Time window
              ├── Pearson correlation: traffic index vs trip duration
              └── Feeds traffic density heatmap in dashboard
```

---

## Core Logic: Consecutive Breach Detection

The central algorithmic challenge — identifying routes in persistent distress, not one-off incidents.

```dax
-- Tag each (Route, Date) as breached only if ALL trips that day were Overrun
VAR _TaggedDates =
    ADDCOLUMNS( _RouteDates,
        "@IsDateBreach",
            VAR _Total  = COUNTROWS( FILTER( _AllTrips, [Routename]=_R && [Trip Date]=_D ) )
            VAR _Breach = COUNTROWS( FILTER( _AllTrips, [Routename]=_R && [Trip Date]=_D && [KPI]="Overrun" ) )
            RETURN IF( _Total > 0 && _Total = _Breach, 1, 0 )
    )

-- A route is Critical if ALL of its last 5 available trip dates breached
VAR _Last5BreachCount =
    SUMX(
        TOPN( 5,
            FILTER( _TaggedDates, [Routename] = _R ),
            [Trip Date], DESC
        ),
        [@IsDateBreach]
    )

VAR _IsCritical = [@Last5BreachCount] >= 5 && [@AvailDates] >= 5
```

**Why this matters:** a route that ran Mon/Tue/Thu/Fri/Mon and breached all five is correctly flagged Critical — calendar gaps (Wednesday, weekends, public holidays) are irrelevant. This is operational pattern detection, not calendar arithmetic.

---

## Traffic Correlation

The key question beyond *what* breached is *why*. The traffic layer answers this.

```python
# For each breached route on a breach date, fetch the traffic
# congestion index for the route's operational time window
def get_traffic_index(route_coords, trip_date, departure_time):
    url = (
        f"https://maps.googleapis.com/maps/api/directions/json"
        f"?origin={route_coords['start']}"
        f"&destination={route_coords['end']}"
        f"&departure_time={to_unix(trip_date, departure_time)}"
        f"&traffic_model=best_guess"
        f"&key={API_KEY}"
    )
    response = requests.get(url).json()
    duration_in_traffic = response['routes'][0]['legs'][0]['duration_in_traffic']['value']
    duration_no_traffic  = response['routes'][0]['legs'][0]['duration']['value']
    return round((duration_in_traffic / duration_no_traffic) * 100)

# Pearson correlation across all breached routes
from scipy.stats import pearsonr
r, p = pearsonr(breach_df['traffic_index'], breach_df['avg_duration'])
# Result: r = 0.87 — strong positive correlation
```

This gives route planners a crucial distinction:
- **High traffic correlation** → infrastructure/timing problem → adjust departure times or reroute
- **Low traffic correlation** → operational problem → check stop sequence, driver schedule, vehicle capacity

---

## AI Pipeline: Multi-Turn Narrative Generation

```python
# Per-station narrative
def generate_station_narrative(station, routes_df, traffic_df):
    prompt = build_prompt(station, routes_df, traffic_df)
    return client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=300,
        messages=[{"role": "user", "content": prompt}]
    ).content[0].text

# Cross-station executive summary using conversation history
def generate_executive_summary(station_narratives):
    conversation = [
        {"role": "user",      "content": format_all_narratives(station_narratives)},
        {"role": "assistant", "content": "Reviewed. Ready for your question."},
        {"role": "user",      "content": "Write a 2-sentence executive summary for senior management..."}
    ]
    return client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=200,
        messages=conversation
    ).content[0].text
```

---

## Dashboard Features

| Feature | Implementation | Engineering Layer |
|---|---|---|
| Station header strip (22 stations) | DAX `grid-template-columns: repeat(auto-fill,minmax(180px,1fr))` | BI |
| Breach toggle (Critical / Total) | DAX-embedded `showView()` JavaScript | BI |
| Dynamic stations · schools counter | JS `counts` object pre-baked at render time | BI |
| Consecutive 5-day breach logic | `TOPN(5) + SUMX(@IsDateBreach)` | BI |
| Breached dates only (Total view) | `FILTER(TOPN(5,...), [@IsDateBreach]=1)` | BI |
| Bus numbers per route | `DISTINCT(SELECTCOLUMNS(FILTER(...KPI="Overrun"...),"@Bus",[Bus]))` | BI |
| School ID + Name lookup | `LOOKUPVALUE(Schools[School Name],Schools[School ID],_Sc)` | BI |
| AI breach narratives per station | Python → Claude API → CSV → Power BI linked table | AI |
| Multi-turn executive summary | Conversation history pattern in Anthropic SDK | AI |
| Traffic congestion index | Google Maps Directions API with `departure_time` | Traffic |
| Traffic vs duration scatter plot | Pearson r = 0.87 correlation · Chart.js | Analytics |
| Hourly traffic density heatmap | Route × Hour × Day · colour-mapped severity | Traffic |
| Breach heat map (route × date) | DAX matrix → HTML table with conditional colouring | BI |
| Weekly + monthly trend charts | Calendar-relationship-aware DAX measures + Chart.js | BI/Analytics |
| AI forecast overlay on trend | Dashed green line projected from last data point | AI |
| Expandable station → school → route | Three-level accordion with animated arrows | BI |
| Root cause per route | Traffic-correlated + manual classification | Analytics |
| Filter summary bar | `ISFILTERED() + CONCATENATEX()` per slicer | BI |

---

## SLA Thresholds

| Route Type | Max Duration | KPI Flag |
|---|---|---|
| MOE Regular | 60 mins | `MOE Trip Duration > 60` → "Overrun" |
| MOE KG | 45 mins | `MOE Trip Duration > 45` → "Overrun" |

Trip duration:
- **HOMETOSCHOOL** — first pick-up → school arrival
- **SCHOOLTOHOME** — school departure → last drop-off

---

## Stack

| Layer | Technology |
|---|---|
| Source system | Oracle ADW (ET Route) |
| Data model | Power BI Desktop · DAX |
| HTML visuals | HTML Content Visual · DAX-generated HTML/CSS/JS |
| AI narratives | Python 3.11 · Anthropic SDK (`claude-sonnet-4-6`) |
| Traffic data | Google Maps Directions API |
| Correlation | Python · pandas · scipy |
| Charts | Chart.js 4.4.1 |
| Pipeline orchestration | Power Automate (post-refresh trigger) |

---

## Setup

### Power BI
```
1. Open Route_SLA_Monitor.pbix
2. Update Oracle ADW connection string
3. Import breach_narratives.csv as linked table
4. Refresh
```

### AI + Traffic Pipeline
```bash
pip install anthropic pandas scipy requests

export ANTHROPIC_API_KEY=your_key
export GOOGLE_MAPS_API_KEY=your_key

python Route_pipeline.py
```

---

## Project Structure

```
Route-sla-intelligence/
├── README.md
├── Route_pipeline.py           # AI + traffic pipeline
├── Route_breach_narratives.py  # Standalone AI narrative generator
├── Route_traffic_correlation.py# Traffic API + correlation analysis
├── sample_data/
│   ├── breach_export.csv          # Anonymised sample
│   ├── breach_narratives.csv      # Sample AI output
│   └── traffic_index.csv          # Sample traffic data
├── measures/
│   ├── SLA_Breach_Visual_v4.dax
│   ├── Station_Summary_Header.dax
│   ├── Critical_Routes_This_Week.dax
│   ├── Active_Filters_Summary.dax
│   └── Notes_Page.dax
└── mockup/
    └── Route_v4_mockup.html    # Interactive dashboard preview
```

---

## Roadmap

- [ ] **Real-time alerts** — Power Automate → Teams notification when route hits day 4 of consecutive breaches
- [ ] **Predictive breach scoring** — ML model trained on trip duration + traffic patterns to flag routes before they breach
- [ ] **Route replanning co-pilot** — LLM-assisted stop resequencing based on breach history + traffic index
- [ ] **Live traffic integration** — replace static congestion index with real-time feed during operational hours
- [ ] **Natural language query** — semantic search across breach history: *"which KG routes in Al Hail breached more than 3 times in June?"*

---

## About

Built as part of a deliberate transition from BI Engineering toward AI Engineering — demonstrating that the distance between analytics and AI is bridged by the same foundation: understanding data deeply enough to build things that are genuinely useful.

The traffic correlation layer is the piece I'm most proud of. It shifts the platform from *reporting what happened* to *explaining why* — which is the difference between a dashboard and an intelligence system.

---

*ET · Route 2026 · For demonstration purposes*
