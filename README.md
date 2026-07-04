# PowerAlert GH

A crowdsourced power outage tracking bot built on Telegram. Report outages, check your area's status, and know whether it's just you or the whole grid.

## Why

Unplanned power outages are a common problem in many regions, but there's often no simple, community-driven way to know if an outage is localized or widespread. PowerAlert GH lets anyone report an outage in their area and lets others check recent reports before assuming it's just their home.

## Features

- **/report** — Log a power outage in your area
- **/status** — View recent outage/restoration reports for any area
- **/restored** — Mark power as back on in /subscribe/subscribe** — Get notified automatically when someone reports an outage in an area you c/help
- **/help** — List all availableAdmin alertsmin alerts** — Reports and restorations are pushed in real time to admin accounts for monitoring

## TePython 3**Python 3** — corepython-telegram-botlegram-bot** — Telegram Bot APSQLite- **SQLite** — lightweight local database for reports and subsRailway **Railway** — cloud hosting, running 24/7 independent of any local machine

## How It Works

1. A user reports an outage via /report, providing an area name.
2. The report is saved to a local SQLite database with a timestamp.
3. Anyone subscribed to that area is notified automatically.
4. Admins receive a real-time alert with the report details.
5. Anyone can check /status for an area to see all recent activity — both outages and restorations — most recent first.

## Setup

1. Clone this repo
2. Install dependencies:


pip install -r requirements.txt
3. Create a bot via [@BotFather](https://t.me/BotFather) on Telegram and get your API token
4. Set the BOT_TOKEN environment variable with your token
5. Run:


python bot.py
## Deployment

This bot is deployed on [Railway](https://railway.app), reading its token from an environment variable rather than hardcoding it, so it can run continuously without exposing credentials in the codebase.

## Roadmap / Ideas

- Web dashboard showing live outage reports on a map
- Better area-name matching (handle typos, alternate spellings)
- /myreports command to view your own report history
