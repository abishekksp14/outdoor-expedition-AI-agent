Autonomous Outdoor Expedition Planner

An autonomous AI agent (AutoGPT/Devin-style) that plans an outdoor activity from a single high-level goal. It independently checks weather, air quality, and historical climate data across multiple candidate locations, replans on its own when nothing fits your constraints, and self-verifies its own conclusion before answering.


Give it one sentence. It plans, investigates, adapts, and reports back,completely on its own.



Why This Project

Most "AI agent" demos are a single API call wrapped in a chat interface. This performs plan → act → observe → replan → self-verify loop, where the model decides on its own:


which locations and dates to check
what to do when nothing meets the stated criteria
when to double-check its own conclusion before finalizing it


Built to explore how far a small tool-calling agent can go on real, live, free data  and to hit (and fix) the real engineering problems that come with building on a hosted LLM API in production.

What It Actually Does

Given a goal like:


"Compare Denver and Boulder, Colorado. Find me the best day in the next 7 days for a hike. I want max temp under 80°F, less than 30% chance of rain, wind under 20mph, and good or moderate air quality."



The agent will:


Resolve every candidate location to coordinates (self-healing — retries automatically if a "City, State" query fails)
Pull real forecasts for every location across the requested window
Check air quality for the most promising days — not exhaustively, selectively, based on what's worth checking
Score every day against your stated constraints
If nothing fits, autonomously adapt to try a longer window, check other candidate locations, loosen exactly one constraint (and say which), or suggest a different activity better suited to the actual conditions
Self-verify its pick with an hour-by-hour breakdown, and cross-check sunrise/sunset so it never suggests starting in the dark
Pull a 5-year historical baseline for genuine context ("this is 10°F hotter than typical for this date")
Report a final recommendation with an honest account of every compromise it made along the way


Architecture

The intelligence lives almost entirely in the system prompt, not in hand-written control flow. The Python side only provides plain, "dumb" tools; the model decides which to call, in what order, how many times, and when to stop.

┌─────────────────────────────────────────────┐
│   One goal, plain English                   │
└───────────────────┬───────────────────────────┘
                     ▼
┌─────────────────────────────────────────────┐
│   Gemini 2.5 Flash (gemini-genai SDK)        │
│   — decides which tool to call, and when     │
└───────────────────┬───────────────────────────┘
                     ▼
        ┌────────────┴────────────┐
        ▼                         ▼
┌───────────────┐        ┌────────────────┐
│  7 Tools       │        │  Open-Meteo     │
│  (pure Python) │◄──────►│  (4 free APIs)  │
└───────────────┘        └────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│   Final recommendation + full trace          │
│   (shown via Gradio UI or ipywidgets)        │
└─────────────────────────────────────────────┘

Tools

ToolPurposeresolve_locationCity name → coordinates. Self-corrects "City, State" queries that fail.get_daily_forecastMax/min temp, rain chance, wind, UV, sunrise/sunset for up to 16 days.get_hourly_forecast_for_dayHour-by-hour breakdown — used to self-verify a day the daily average made look good (or bad).get_air_quality_forecastUS AQI for a specific day — relevant for physical activity, not just weather.get_historical_baselineReal 5-year average for the same calendar date, for genuine "is this unusual?" context.score_day_for_activityPure arithmetic, checks a day's numbers against stated constraints.suggest_alternative_activityIf nothing fits, proposes an activity actually suited to the real conditions.

All tool parameters are flat scalars (strings, floats, ints) — never nested objects. This is a deliberate design choice: LLM function-calling is unreliable at reconstructing nested structures from memory, and asking a model to pass back a whole object it saw earlier is one of the most common causes of malformed tool calls.

Tech Stack

PieceWhyGoogle ColabFree, zero local setup, one click to a shareable link.google-genaiGemini's current official SDK, provides native function/tool calling.gemini-2.5-flashFree-tier eligible, fast, strong enough for structured multi-tool reasoning.Open-Meteo (Geocoding, Forecast, Air Quality, Historical Archive APIs)Real, live data, zero signup, zero API key, ever.GradioUI + a public shareable link straight out of Colab.ipywidgetsZero-external-dependency fallback UI (no tunnel required).

Running It


Open the notebook in Google Colab.
Get a free Gemini API key: https://aistudio.google.com/apikey
Store it in Colab Secrets (🔑 icon in the sidebar) under the name GOOGLE_AI_API_KEY this keeps it out of the notebook file entirely, so it's safe to commit.
Run all cells top to bottom.
Use the Gradio UI (open the public link in a new tab, not the embedded Colab preview) or the ipywidgets fallback cell.


Engineering Challenges Along the Way

Building this surfaced several real, non-obvious problems worth documenting:


SDK deprecation mid-project — google.generativeai was fully sunset by Google; migrated to google-genai.
Silent tool-call budget cap — automatic function calling defaults to 10 tool-call turns per message. A rich multi-location investigation can genuinely need more; the fix (maximum_remote_calls=30) is one line, but diagnosing why the agent returned no answer at all took real debugging.
MALFORMED_FUNCTION_CALL errors — caused by tool parameters typed as nested dict objects. Fixed by redesigning every tool signature to take flat scalar parameters instead.
Type coercion mismatch — Gemini passes numbers as floats regardless of a Python type hint; an API expecting a clean integer (forecast_days) rejected 7.0 with a 400 error. Type hints are documentation, not enforcement — tools that forward values into strict external APIs need to defensively cast them.
429 vs. 503 errors — rate-limit (quota) errors and server-overload errors look similar but need different handling; retry-with-backoff now covers both, with a final message that tells you which one you actually hit.
Geocoding query format quirks — Open-Meteo's geocoder doesn't reliably handle "City, State" as one query; resolve_location now retries automatically with just the city name.


Possible Extensions


Cache repeated API calls within a single run (avoid re-fetching the same location/date twice)
A compare_activities tool, letting the agent choose between multiple activity types per day, not just one
Persist past goals/results to disk so the agent remembers previous runs across sessions
