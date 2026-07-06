# 🌾 AgriMargin

### The Missing Economic Layer for Smallholder Crop Decisions

**A multi-agent AI system that doesn't just diagnose crop disease — it decides whether treating it is worth the money, then safely acts on that decision.**

*Built for Kaggle's 5-Day AI Agents: Intensive Vibe Coding Capstone — Track: Agents for Good*

---

[![Track](https://img.shields.io/badge/Track-Agents%20for%20Good-2e7d32?style=flat-square)]()
[![Built with](https://img.shields.io/badge/Built%20with-Google%20ADK-4285F4?style=flat-square)]()
[![Status](https://img.shields.io/badge/Status-Working%20Prototype-orange?style=flat-square)]()

> The underlying decision logic and dealer-order flow also transfer directly to the **Agents for Business** track if a cooperative or agri-input company deploys it.

## Table of Contents

- [The Problem](#the-problem)
- [Why Agents, Specifically](#why-agents-specifically)
- [Solution & Architecture](#solution--architecture)
- [Key Concepts Demonstrated](#key-concepts-demonstrated)
- [What's Real vs. What's a Stub](#whats-real-vs-whats-a-stub-read-this-before-you-demo-it)
- [Quick Start](#quick-start)
- [Real-World Value & Path to Income](#real-world-value--path-to-income)
- [Honest Limitations](#honest-limitations)
- [Why This, and Not Another Diagnosis App](#why-this-and-not-another-diagnosis-app)
- [Repository Structure](#repository-structure)

---

## The Problem

A smallholder farmer with a sick crop today has to stitch together information from several disconnected places: a photo-ID app tells them what disease it might be, a weather app or their own intuition tells them if conditions favor its spread, and a separate market app or a trip to the mandi tells them the current price. **Nothing puts those three pieces of information together and answers the question that actually determines their income:** is treating this worth the money, given how close I am to harvest and what the crop will sell for?

That gap has a real cost. Farmers either over-spend on treatment that doesn't pay for itself, or under-treat a genuinely high-risk outbreak because they never ran the numbers. Existing tools (Plantix-style photo diagnosis apps, government mandi-price apps, generic weather apps) each solve one piece of this and stop. **None of them close the loop into a recommendation and an action.**

## Why Agents, Specifically

This isn't a single-model classification problem — it's a small research-and-decide workflow: gather evidence from three different domains (plant pathology, meteorology, market economics), reason over all of it together, and only then, optionally, take a real-world action (placing an order) that has to be gated by human approval.

That's exactly the shape of problem multi-agent orchestration is for: specialist agents that each do one job well, coordinated by an orchestrator that knows the right sequence and can hand off between them mid-conversation, with a tool-execution layer that can be paused for confirmation before anything irreversible happens.

## Solution & Architecture

AgriMargin is built on **Google's Agent Development Kit (ADK)** as five specialist agents behind one orchestrator:

| # | Agent | Role |
|---|---|---|
| 1 | **Diagnosis Agent** | Identifies the pest/disease from a farmer's photo (or symptom description in text-only testing) using Gemini's native multimodal reasoning, and reports a confidence level rather than a false-certain answer. |
| 2 | **Weather-Risk Agent** | Pulls a short-term forecast and computes a disease-spread risk index from temperature and humidity. The risk heuristic (higher humidity + moderate-warm temperatures favor fungal spread) is real, working logic — not a placeholder. |
| 3 | **Market-Intelligence Agent** | Fetches the current price and 7-day trend for the crop at the farmer's local market. This is the natural slot for an MCP server integration. |
| 4 | **Economic-Advisor Agent** ⭐ | **The core innovation.** Combines the diagnosis, the risk level, and the market price into a genuine cost-benefit calculation — expected value of yield saved by treating, versus the cost of treatment, adjusted for how close harvest is. Outputs: `spray` / `spray (marginal)` / `don't spray` / `don't spray — too close to harvest`. |
| 5 | **Action Agent** | Only engages if the recommendation is to spray and the farmer wants to act on it. Drafts an order message to a local input dealer, and only *sends* it after explicit human confirmation. Can also schedule a reapplication reminder. |

See [`docs/architecture.svg`](docs/architecture.svg) for the full diagram.

```
agrimargin_orchestrator (root Agent)
 ├── diagnosis_agent        — pest/disease ID from photo or symptoms
 ├── weather_risk_agent     — forecast → disease-spread risk index
 ├── market_intel_agent     — current crop price + trend
 ├── economic_advisor_agent — spray / wait / sell recommendation + math
 └── action_agent           — drafts + (confirmed) sends dealer order,
                               schedules reapplication reminder
```

The orchestrator is a standard ADK `Agent` with `sub_agents=[...]`; it uses ADK's built-in delegation (`transfer_to_agent`) to route a farmer's message to the right specialist in sequence, rather than a hand-coded if/else pipeline. This means the conversation stays natural — a farmer can ask a follow-up question and the orchestrator decides which specialist should answer it, instead of forcing a rigid form-fill flow.

## Key Concepts Demonstrated

| Concept | Where | Notes |
|---|---|---|
| **Multi-agent system (ADK)** | `agents.py` | 5 specialist `Agent`s + 1 orchestrator with `sub_agents`, verified against the real installed `google-adk==2.3.0` API — not written from memory. The graph builds and its structure was tested. |
| **Security features** | `agents.py`, `tools.py` | `send_dealer_order` and `schedule_reapplication_reminder` are registered with `require_confirmation=True`. ADK will not execute either without an explicit human "yes" — the agent physically cannot place an order or send a message unattended. |
| **MCP Server** | `market_intel_agent` in `agents.py` | Currently a local Python stub function so the pipeline is runnable end-to-end without external credentials. See [Quick Start](#quick-start) for exactly how to swap it for a real `MCPToolset` connection. We're transparent about this rather than faking a live MCP connection for the demo. |
| **Agent skills / Antigravity / Deployability** | Video | Demonstrated live in the walkthrough video rather than in static code. |

## What's Real vs. What's a Stub — Read This Before You Demo It

| Function | Status |
|---|---|
| `compute_disease_risk()` | ✅ **Real logic.** A genuine (simplified) humidity/temperature heuristic. Tune thresholds with an agronomist per crop. |
| `compute_economics()` | ✅ **Real logic.** Genuine cost-vs-expected-loss math. This is the core novel piece. |
| `get_weather_forecast()` | ⚠️ **Stub.** Returns seeded fake data. Swap for a real weather API. |
| `get_market_price()` | ⚠️ **Stub.** Returns seeded fake data. Swap for a real mandi/market API or MCP server. |
| `send_dealer_order()` | 🔒 **Simulated send**, gated by `require_confirmation=True`. Swap the body for a real WhatsApp Business Cloud API / Twilio call. |
| `schedule_reapplication_reminder()` | 🔒 **Simulated**, also confirmation-gated. Swap for a real calendar/SMS integration. |

Every stub is labeled `STUB` / `SIMULATED` in its return value on purpose — so it's never ambiguous in a demo which numbers are real.

## Quick Start

```bash
pip install -r requirements.txt
cp .env.example .env      # then edit .env with your key
export GOOGLE_API_KEY=your_key_here   # free key: https://aistudio.google.com/apikey
python main.py --demo     # runs one canned example end to end
python main.py            # interactive mode
```

Or open [`AgriMargin_Kaggle_Capstone.ipynb`](AgriMargin_Kaggle_Capstone.ipynb) directly in Google Colab for a fully self-contained, runnable version with the writeup and video script included.

### Swapping in a real MCP server

Right now `market_intel_agent` calls a local Python stub function. To make this a genuine MCP integration:

1. `pip install "google-adk[mcp]"`
2. Stand up (or find) an MCP server that exposes your region's market-price data as a tool.
3. In `agents.py`, replace `FunctionTool(get_market_price)` with an `MCPToolset` pointed at that server.
4. Do the same for the dealer-order integration in `action_agent` if you want outbound messages to go through an MCP server rather than a direct API call.

## Real-World Value & Path to Income

This isn't designed to stop at the competition. Three concrete monetization paths exist without inventing new business models:

- **Dealer commission** — local agri-input dealers pay a small referral fee for orders routed to them through the Action Agent.
- **FPO/cooperative subscription** — Farmer Producer Organizations pay a flat fee for their member base to use the system, instead of per-farmer pricing that smallholders can't absorb.
- **B2B data licensing** — anonymized, consented crop-health and spray-decision data has real value to agri-input companies and crop insurers for underwriting, a market that already exists today.

## Honest Limitations

- The disease-risk and economics heuristics need review by an agronomist per crop before the numbers are trusted for real farm decisions.
- Diagnosis accuracy on real-world phone photos (bad lighting, blur, wrong crop part) hasn't been validated against a labeled photo set yet.
- The market-price and weather integrations are stubs, clearly labeled as such, pending a real API or MCP server connection.
- No real payment/ordering flow exists — `send_dealer_order` is simulated. Wiring an actual dealer network in is the largest remaining piece of work, and it's a partnerships problem as much as an engineering one.

## Why This, and Not Another Diagnosis App

The honest pitch here isn't *"we built a better plant-disease classifier"* — diagnosis-only tools already exist and do that job reasonably well. The contribution is the layer nobody has built yet: an agent that takes that diagnosis and asks the question a farmer actually cares about — *is this worth spending money on, right now, given my harvest date and today's price* — and only then, with explicit permission, does something about it. That's a genuinely different product, built the way multi-agent systems are supposed to be used: specialists that each do one job well, coordinated by an orchestrator, gated by a human at the one point where real money moves.

## Repository Structure

```
agrimargin/
├── agents.py                        # 5 specialist agents + orchestrator
├── tools.py                         # function tools (real logic + labeled stubs)
├── main.py                          # CLI runner
├── requirements.txt
├── .env.example
├── AgriMargin_Kaggle_Capstone.ipynb # full submission notebook (code + writeup + video script)
└── docs/
    ├── architecture.svg
    ├── KAGGLE_WRITEUP.md
    ├── VIDEO_SCRIPT.md
    └── SUBMISSION_CHECKLIST.md
```

---

<p align="center"><i>Built for smallholder farmers, not for the leaderboard.</i></p>
