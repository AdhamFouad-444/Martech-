# Martech- / Viaffinity RevOps Command Center

Single master dashboard for Viaffinity's HubSpot portal (48485556): deals, leads, contacts,
companies, meetings, calls, emails, tasks, sequences, workflows, marketing activity, plus
deliverability monitoring for viaffinity.com and getviaffinity.com.

Three questions at a glance:

1. What is happening across pipeline and GTM right now, and what should a rep do next?
2. Can we trust the data?
3. Are our domains healthy enough to keep sending?

## Stack

- **API + sync engine:** Python 3.12, FastAPI, APScheduler (port 8000)
- **Frontend:** React 18, TypeScript, Vite (port 5173)
- **Store:** SQLite (WAL) — full local history, no external DB
- **Run:** `docker compose up` or `make dev`

## Docs

- [docs/PLAN.md](docs/PLAN.md) — architecture, auth model, sync engine, per-tab build plan,
  phases, risks, and the access checklist needed to start.

## Delivery

Three phases, each with a demo: Phase 1 = Revenue & GTM tab, Phase 2 = Data Integrity tab,
Phase 3 = Deliverability tab. Phase 0 is an access/feasibility spike.
