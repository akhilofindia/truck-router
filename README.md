<div align="center">

# 🚚 Truck router

**Weather-aware truck route planner: compare routes by the wind, rain, and snow a driver will actually hit.**

![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5.6-3178C6?logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-6-646CFF?logo=vite&logoColor=white)
![Django](https://img.shields.io/badge/Django-5.1-092E20?logo=django&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![Leaflet](https://img.shields.io/badge/Leaflet-1.9-199900?logo=leaflet&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)
![No API keys](https://img.shields.io/badge/API%20keys-none%20required-brightgreen)

</div>

---

## 📖 Overview

Truck router is a React + Django app that helps a truck driver pick the safest route. It weighs the weather along the way, the load being hauled, and the trip time.

The driver enters an **origin**, a **destination**, a **departure time**, and a **load weight**. The app then:

1. Asks a truck routing service for up to **three distinct routes**.
2. Places checkpoints along each route every **10, 25, or 50 miles** and estimates when the truck reaches each one.
3. Looks up the **forecast for the hour the truck arrives** at each checkpoint, not the weather right now.
4. Scores every route and **recommends the one with the least dangerous weather**.

## ✨ Features

- 🗺️ **Color-coded routes.** Each route and checkpoint is drawn in the color of its risk level.
- ⏱️ **Arrival-time forecasts.** ETAs follow the router's own maneuver times, so slow stretches aren't treated as highway speed.
- ⚖️ **Load-aware wind limits.** Heavy loads hit no-travel and severe thresholds at lower wind speeds.
- 🌡️ **48-hour forecast heatmap.** A slider plays the corridor forecast from hour 0 to 48. All hours load with the plan, so dragging the slider makes no new network requests.
- 🔁 **Fallbacks built in.** If the truck router is down, ordinary driving routes are used and labeled as not truck-restricted.
- 🔑 **No API keys.** Every external service is free and public.

## 🧱 Tech Stack

| Layer | Tools |
| --- | --- |
| Frontend | React 18, TypeScript, Vite, Leaflet / React-Leaflet, custom canvas heatmap |
| Backend | Django 5, Django REST Framework, Requests |
| Serving | Gunicorn (gthread), Docker multi-stage build |
| Hosting | Render (Docker blueprint) |
| Tests | pytest |

## 📂 Project Structure

```text
.
├── backend/
│   ├── config/              # Django settings, URLs, WSGI/ASGI
│   └── planner/
│       ├── geocode.py       # Place-name search
│       ├── routing.py       # Valhalla truck routes, OSRM fallback, alternates
│       ├── sampling.py      # Checkpoints and ETAs along a route
│       ├── weather.py       # Hourly forecasts (NWS + MET Norway)
│       ├── risk.py          # Wind / rain / snow bands and load rules
│       ├── scoring.py       # Mile totals, average risk, ranking
│       ├── planning.py      # Builds a full plan and the heatmap
│       ├── views.py         # API endpoints
│       └── tests/           # pytest suite
├── frontend/
│   └── src/
│       ├── components/      # PlannerForm, MapView, RouteList
│       ├── map/HeatmapLayer.ts
│       ├── api.ts           # API client
│       └── riskColors.ts    # Shared risk legend
├── Dockerfile               # Builds the frontend, then serves it with the API
├── render.yaml              # Render blueprint
├── requirements.txt
└── requirements-dev.txt
```

## 🚀 Getting Started

### Prerequisites

- Python **3.12+**
- Node.js **22+**

### 1. Clone

```bash
git clone https://github.com/akhilofindia/geo-routing.git
cd geo-routing
```

### 2. Backend (terminal 1)

```bash
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements-dev.txt
.venv\Scripts\python backend\manage.py runserver
```

> On macOS / Linux, use `.venv/bin/python` and forward slashes.

### 3. Frontend (terminal 2)

```bash
cd frontend
npm install
npm run dev
```

Open **http://127.0.0.1:5173**. The Vite dev server proxies `/api` to Django on port `8000`.

### 🐳 Run with Docker

One container serves the built frontend and the API on a single port:

```bash
docker build -t truck-route-planner .
docker run -p 8000:8000 truck-route-planner
```

Then open **http://localhost:8000**.

## 🧪 Running Tests

From the `backend` folder:

```bash
..\.venv\Scripts\python -m pytest
```

## ⚙️ Configuration

All settings are optional environment variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `DJANGO_SECRET_KEY` | dev-only key | Django secret key. **Set this in production.** |
| `DJANGO_DEBUG` | `1` (`0` in Docker) | Debug mode |
| `ALLOWED_HOSTS` | `*` | Comma-separated allowed hosts |
| `FRONTEND_DIST` | `frontend/dist` | Built frontend folder that Django serves |
| `PORT` | `8000` | Port gunicorn binds to in Docker |

## 🧮 How a Trip Is Scored

The weather at each checkpoint comes from the forecast hour that contains the truck's estimated arrival. Wind, rain, and snow are rated separately, and the checkpoint takes the **worst** of the three.

| Condition | 🟢 Low | 🟡 Moderate | 🟠 High | 🔴 Severe | ⛔ No travel |
| --- | --- | --- | --- | --- | --- |
| Wind | < 25 mph | 25–34 | 35–44 | 45–54 | ≥ 55 mph |
| Rain | < 0.10 in/hr | 0.10–0.25 | 0.25–0.50 | 0.50–1.00 | > 1.00 |
| Snow | < 0.5 in/hr | 0.5–1.0 | 1.0–2.0 | 2.0–3.0 | > 3.0 |

A value exactly on the line between two bands takes the higher band. Rain at 1.00 in/hr and snow at 3.0 in/hr still count as severe.

### Load weight (wind only)

- **≥ 55 mph:** no travel for any load.
- **45–54 mph** with a load over **30,000 lb:** no travel.
- **35–44 mph** with a load over **40,000 lb:** severe.

### Ranking

Each stretch between two checkpoints counts toward the worse of its two ends. Checkpoints the forecast doesn't reach are marked unknown and left out of the mile totals.

**Average risk** is weighted by distance: Low 1, Moderate 2, High 3, Severe 4, No travel 5.

The recommended route is the first one after sorting by:

1. Fewest **no-travel** miles
2. Fewest **severe** miles
3. Fewest **high-risk** miles
4. Lowest **average risk**
5. Shortest **travel time**

No-travel miles come first, so a route with a banned stretch can never beat a route that is only severe.

## 🔌 API Reference

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/api/health` | Health check |
| `GET` | `/api/geocode?q=Dallas` | Search for a place by name |
| `POST` | `/api/plan` | Plan, score, and rank routes |

### `POST /api/plan`

```json
{
  "origin": "Dallas, Texas",
  "destination": {
    "label": "Oklahoma City, Oklahoma",
    "latitude": 35.4676,
    "longitude": -97.5164
  },
  "departure": "2026-10-10T15:00:00-05:00",
  "load_lb": 42000,
  "sample_miles": 25
}
```

| Field | Rules |
| --- | --- |
| `origin`, `destination` | A place name, or an object with coordinates from the geocoder |
| `departure` | ISO 8601 time **with a time zone** |
| `load_lb` | Load weight in pounds |
| `sample_miles` | `10`, `25`, or `50` |

## 🌐 External Services

| Purpose | Service | Notes |
| --- | --- | --- |
| Map tiles | Esri World Street Map | No key needed |
| Geocoding | [Open-Meteo Geocoding](https://open-meteo.com/en/docs/geocoding-api) | |
| Truck routing | [Valhalla](https://valhalla.openstreetmap.de/) (public) | The load is converted to metric tons. Fair-use demo server, which usually limits a route to about 1,500 km. |
| Routing fallback | [OSRM](https://project-osrm.org/) (public) | Driving routes, labeled as not truck-restricted |
| Weather (US) | [National Weather Service](https://www.weather.gov/documentation/services-web-api) | Hourly wind, rain, and snow grids |
| Weather (worldwide) | [MET Norway Locationforecast](https://api.met.no/) | Covers points outside the US and any NWS gaps |

If the truck router returns fewer than three distinct routes, the app also asks for a toll-avoiding route, a route that avoids highways more, and, if still needed, a route pushed away from the middle of the fastest one.

The heatmap samples the corridor about every 20 miles, plus points about 12 miles to either side. All 49 forecast hours come back with the plan, and the slider only changes which hour the canvas draws.

## ☁️ Deployment (Render)

[`render.yaml`](render.yaml) defines a free Render Docker web service with a health check on `/api/health` and a generated `DJANGO_SECRET_KEY`.

1. Push this repo to GitHub.
2. In Render, choose **New → Blueprint** and select the repo.
3. Render builds the [`Dockerfile`](Dockerfile), sets `PORT`, and starts gunicorn.

> ⏳ **Cold starts:** on the free plan, the first request after the service has been idle can take up to a minute while it wakes up and the routing and weather services respond.

## ⚠️ Known Limitations

- The public Valhalla and OSRM servers are shared demo servers, so they can be slow or rate-limited.
- Valhalla usually rejects routes longer than about 1,500 km.
- Forecasts only cover a limited time ahead. Checkpoints reached after that are marked unknown.
- Gunicorn runs one worker with a 40-second timeout, which suits a free-tier host better than heavy traffic.

<details>
<summary><b>🎥 Demo walkthrough script (5–10 min)</b></summary>

1. **0:00** Explain the page: three truck routes, scored by the weather at the truck's arrival time, the load, and the trip length.
2. **0:40** Enter Dallas → Oklahoma City, a departure a few hours ahead, 42,000 lb, and a 25-mile sample. Submit.
3. **2:00** Show the three colored routes, the checkpoint dots, and the recommendation. Point out the no-travel, severe, and high-risk miles on the cards.
4. **3:30** Open a checkpoint and read its wind, rain, snow, risk level, and ETA.
5. **4:30** Click another route to bring its line and checkpoints to the front.
6. **5:20** Drag the forecast slider from hour 0 to 48. Point out that it sends no new requests.
7. **6:40** Switch to a 10-mile sample, or raise the load, and run the plan again. Explain the load-based wind rules.
8. **8:00** Show `risk.py` and `scoring.py` (risk bands and ranking), then `HeatmapLayer.ts` (canvas redraw).
9. **9:00** Mention the Valhalla distance limit and the OSRM fallback.

</details>

---

<div align="center">
Built with ❤️ by <a href="https://github.com/akhilofindia">Akhil</a>
</div>
