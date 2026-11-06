# FortyGuard — Street-Level Heat Safety Assistant

An AI-powered early warning system for extreme heat, built for the **FortyGuard Hackathon**.

Most weather apps give you a single, city-wide number. But heat doesn't work that way — the temperature on a shaded street and an open construction site can differ by several degrees, and that difference matters for someone's safety.

This project combines a **conversational Telegram assistant** with an **autonomous monitoring workflow** to deliver hyper-local, street-level heat safety — both on request and proactively.

---

## 🎥 Demo

A short demo video showing the full user flow (conversation → subscription → automatic alert) is available here: **[https://drive.google.com/file/d/1uZ8bOzBfVvpRrXEncEvbPiryduklQQQQ/view?usp=drivesdk]**

---

## 🧩 What it does

### 1. Conversational Assistant (`smart Temperature.json`)
A Telegram bot that:
- Greets the user and asks for their name, location, and role (outdoor worker, resident, pedestrian, student, etc.)
- Accepts a location as **typed text** *or* a **shared map pin/venue** — either way, it resolves both a readable address and precise coordinates using forward/reverse geocoding.
- Pulls hyper-local microclimate data (actual temperature, feels-like temperature, humidity, air quality index) for that exact location.
- Gives safety tips tailored to the user's specific role.
- Lets the user subscribe to automatic heat alerts, saving their info to a Google Sheet.
- Replies fully in the user's language (Arabic/English, dynamically detected).

### 2. Autonomous Monitoring Workflow (`My_workflow_2.json`)
Runs on a schedule (every 15 minutes) and:
- Reads all subscribed users from the Google Sheet.
- Fetches current conditions for each subscriber's saved location.
- Compares the current temperature against a safety threshold **and** the last recorded reading — an alert is only sent when conditions are both above the safety threshold *and* still rising, to avoid repeat spam for the same heat event.
- Sends a proactive warning message directly to the subscriber via Telegram when conditions turn dangerous.
- Updates the recorded temperature in the sheet after every check, whether or not an alert was sent.

### 3. Alert Follow-up Bot (`FortyGuard - Alert Bot Handler.json`)
A second Telegram bot (the one that sends alerts) that also handles follow-up questions from users after they've received a warning — e.g. "what should I do now?" or "is it safe to keep working?" — using conversation memory so it understands references like "the warning" or "the temperature" without the user repeating context.

### 4. Weather Data Sub-workflow (`My_Sub-workflow.json`)
A shared sub-workflow called by both bots. It fetches a baseline temperature reading from Open-Meteo, then submits it to the FortyGuard API to retrieve full microclimate and environmental parameters (feels-like temperature, humidity, air quality, etc.) for the requested coordinates.

---

## 🏗️ Architecture

```
                    ┌─────────────────────────┐
                    │   User (Telegram)       │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │  smart Temperature       │  ← conversational assistant
                    │  (name, location, role,  │
                    │   report, subscription)  │
                    └───────────┬─────────────┘
                                │ calls
                    ┌───────────▼─────────────┐
                    │   My Sub-workflow        │  ← Open-Meteo + FortyGuard API
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Google Sheet           │  ← stores subscribers
                    └───────────┬─────────────┘
                                │ read every 15 min
                    ┌───────────▼─────────────┐
                    │  Monitoring Workflow     │  ← proactive checks
                    └───────────┬─────────────┘
                                │ if unsafe
                    ┌───────────▼─────────────┐
                    │  Alert Bot Handler       │  ← sends warning +
                    │                          │    answers follow-ups
                    └──────────────────────────┘
```

---

## ⚙️ Tech Stack

- **n8n** — workflow orchestration (all logic below is built as n8n workflows)
- **OpenRouter** — LLM access for the conversational agents
- **Telegram Bot API** — user-facing interface (two bots: main assistant + alerts)
- **Google Sheets API** — subscriber storage
- **Open-Meteo API** — baseline weather data
- **FortyGuard API** — hyper-local microclimate & environmental parameters

---

## 🔧 Setting this up yourself

These workflow files were exported directly from n8n and are **not plug-and-play** — a few things are tied to the original author's accounts and need to be reconnected:

1. **Import order:** import `My_Sub-workflow.json` first, then `smart_Temperature.json`, `My_workflow_2.json`, and `FortiGuard - Alert Bot Handler.json`.
2. **Credentials:** each Telegram, Google Sheets, and OpenRouter node needs to be reconnected to your own credentials — the credential IDs in these files only resolve inside the original author's n8n instance.
3. **Google Sheet:** create your own sheet with these columns: `name`, `type`, `address`, `latitude`, `longitude`, `chat_id`, `last_temperature`. Update the Document ID in the Google Sheets nodes to point to your sheet.
4. **Sub-workflow link:** after importing `My_Sub-workflow.json`, re-select it in the "Call Sub-workflow" / "FortyGuard Tool" node inside `smart_Temperature.json`, since the workflow ID reference won't carry over automatically.
5. **FortyGuard API key:** replace `YOUR_API_KEY` in `My_Sub-workflow.json` with a valid FortyGuard API key.
6. **Two Telegram bots:** this project uses two separate bot tokens — one for the main conversational assistant, one for sending alerts and handling follow-ups. Create both via [@BotFather](https://t.me/BotFather) and wire each into its respective workflow.

---

## 💡 Why this approach

Most heat-safety tools stop at "here's the current temperature." This project goes further by:
- Resolving **precise, street-level location data** rather than city-wide averages, regardless of whether the user types an address or shares a pin.
- Tailoring advice to the person's **actual activity**, not a generic warning.
- Closing the loop with **proactive monitoring** — the system reaches out the moment conditions become dangerous, instead of waiting for the user to ask.

Built end-to-end for the FortyGuard Hackathon.

---

## 📡 Sample FortyGuard API Request & Response

Here's a real request made from the `My_Sub-workflow.json` sub-workflow, and the actual response returned by the FortyGuard API for a location in Cairo, Egypt:

**Request** — `POST https://api.fortyguard.com/v1/env_params`
```json
{
  "latitude": 30.058376,
  "longitude": 31.326881,
  "temperature": 25.7,
  "date_time": {
    "filter_type": 1,
    "start_date": "2026-08-29",
    "start_time": "00:00"
  }
}
```

**Response** — `GET https://api.fortyguard.com/v1/status/{activity_id}`
```json
{
  "error": false,
  "status_code": 200,
  "message": "Completed",
  "data": {
    "activity_id": "0c8007bc-16ef-414b-b2fa-881b3803f9eb",
    "status": "Completed",
    "result": {
      "metadata": {
        "timezone": "GMT+2",
        "timezone_offset_hours": 2,
        "time_range": {
          "start": "2026-08-29T00:00:00+02:00",
          "end": "2026-08-29T00:00:00+02:00",
          "interval": "1h",
          "count": 1
        }
      },
      "locations": [
        {
          "lat": 30.0625,
          "lon": 31.3125,
          "elevation": 61,
          "temperature": 25.7,
          "parameters": {
            "heat_index_celsius": [26.7],
            "apparent_temperature_celsius": [31.7],
            "relative_humidity_percent": [63.7],
            "precipitation_mm": [0],
            "cloud_cover_octas": [0],
            "wet_bulb_temperature_celsius": [23.4],
            "air_quality:idx": [73.6],
            "air_quality_pm2p5:idx": [73.6],
            "air_quality_pm10:idx": [33.2],
            "air_quality_no2:idx": [7.6],
            "aqi_us_co": [1.6],
            "air_quality_o3:idx": [41.5],
            "air_quality_so2:idx": [16.3],
            "methane_ppb": [2213.7],
            "co2_ppm": [444]
          }
        }
      ]
    }
  }
}
```

The bot extracts `apparent_temperature_celsius` (the "feels-like" value) as the primary number shown to users and used for alert threshold comparisons, alongside humidity and air quality for the full safety report.

---

## ⚠️ What doesn't work yet / known limitations

- **No standalone web demo:** this project is Telegram-native by design (a conversational bot + a background monitoring workflow), so there is no separate web URL to click — it needs to be run as n8n workflows connected to a Telegram bot. See setup steps above.
- **Alert follow-up bot uses a lightweight LLM:** the alert bot's free-tier model sometimes gives a generic greeting instead of a context-aware answer on the very first follow-up message after an alert, before conversation memory "warms up." A second message in the same conversation is handled correctly.
- **No spam-prevention beyond "still rising" check:** the monitoring workflow currently avoids repeat alerts by only firing when the temperature is both above the safety threshold *and* higher than the last recorded reading. It does not yet implement a fixed cooldown window (e.g. "don't alert the same person more than once per hour").
- **Single shared Google Sheet as the subscriber database:** works well for a hackathon-scale demo, but would need to move to a proper database for production-scale usage.
- **Manual credential setup required:** as noted above, all API keys and OAuth credentials must be reconnected manually after importing the workflows — this is an n8n export limitation, not a design choice.
