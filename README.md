# Agentic SQL Chatbot

A local weather chatbot powered by a multi-node LangGraph agent, Ollama, and an MCP data server. Ask natural-language questions about weather across 8 US cities — the agent fetches live data on demand, writes SQL, self-corrects on failure, and answers in plain English.

```
hottest day last summer in Chicago?
coldest night in New York last January?
will it rain in Miami tomorrow?
what's the forecast for Atlanta this week?
which days had rain last month in Seattle?
```

**Supported cities:** Atlanta, Chicago, New York, Los Angeles, Houston, Seattle, Miami, Denver

---

## Architecture

| File | Role |
|---|---|
| `chatbot.py` | CLI loop — input, agent call, answer + MCP metrics |
| `agent.py` | LangGraph graph — planner → enricher → schema → actor → judge → answer |
| `city_data_server.py` | MCP server — Tools, Resources, Prompts, Middleware |
| `mcp_client.py` | Sync MCP client — persistent connection, caching, session metrics |
| `database.py` | SQLite helpers — schema, sampling, querying |
| `llm_helper.py` | Ollama wrapper |
| `benchmark.py` | Automated MCP benchmark — cache impact, error demo, primitives discovery |
| `data_sources.py` | Open-Meteo HTTP fetch logic |

### MCP Protocol

All three MCP primitives are implemented:

| Primitive | Implementation | Purpose |
|---|---|---|
| **Tools** | `plan_data_load`, `check_coverage`, `load_weather`, `load_csv` | Data fetching and routing |
| **Resources** | `weather://schema`, `weather://tables` | Live DB schema and table inventory |
| **Prompts** | `extreme_value_query`, `trend_overview_query`, `specific_date_query`, `comparison_query`, `aggregation_query` | Server-managed SQL scaffolds injected into the actor |

The client adds:
- **Persistent connection** — server subprocess spawns once per session (~750ms saved per subsequent call)
- **Caching** — tools, resources, and prompts cached with location-scoped invalidation
- **SessionMetrics** — p50/p95 latency, cache hit rate, time saved, error rate

The server adds:
- **Middleware** — `ErrorHandlingMiddleware` wraps every tool; exceptions return `{"error": "..."}` instead of crashing the enricher

---

## Agentic Workflow

```
User query
    │
    ▼
┌─────────┐
│ PLANNER │  LLM parses intent and identifies which tables to query
└────┬────┘
     │
     ▼
┌──────────┐ ◄─────────────────────────────────────┐
│ ENRICHER │  MCP: plan → coverage check → load     │ (reload: missing date range)
└────┬─────┘                                        │
     │                                              │
     ▼                                              │
┌────────┐                                          │
│ SCHEMA │  MCP: weather://schema + SQL Prompt       │
└───┬────┘                                          │
    │                                               │
    ▼                                               │
┌───────┐ ◄──────────────────┐                     │
│ ACTOR │  LLM writes SQL     │ (retry, up to 5x)  │
└───┬───┘                    │                     │
    │                        │                     │
    ▼                        │                     │
┌───────┐  fail (bad SQL) ───┘                     │
│ JUDGE │  fail (missing data) ────────────────────┘
└───┬───┘
    │ pass
    ▼
┌──────────────┐
│ FINAL ANSWER │  LLM synthesizes rows into plain English
└──────────────┘
```

**Planner** — Outputs a JSON plan: intent, tables, and strategy. Does not decide what data to load.

**Enricher** — Calls `plan_data_load` to parse the query, `check_coverage` to check if the date range is in SQLite, and `load_weather` only if data is missing.

**Schema** — Reads the `weather://schema` Resource and fetches a matching Prompt scaffold (one of five SQL templates keyed by intent). Both are injected into the actor.

**Actor** — Receives schema, sample rows filtered to the loaded city, the plan, and any prior failed attempts. Outputs raw SQL, immediately executed.

**Judge** — Deterministic checks first (syntax error, zero rows). If rows returned, a lightweight LLM call checks semantic correctness against the result data — not the SQL form. Routes back to the actor (up to 5 retries) or to the enricher if a required date range is missing from the DB.

**Final Answer** — Converts result rows into a plain-English answer grounded strictly in the data.

---

## Running

**Prerequisites:** Python 3.10+, [Ollama](https://ollama.com) running locally with a model pulled (`ollama pull llama3.1`)

```bash
pip install -r requirements.txt
python chatbot.py
```

Override the model: `OLLAMA_MODEL=llama3.2 python chatbot.py`

---

## Running with Docker

```bash
docker compose up --build
docker compose exec chatbot python chatbot.py
```

Copy `.env.example` → `.env` to override `OLLAMA_MODEL` or `OLLAMA_HOST`. CSV files in `./data/` are mounted into the container for use with `load_csv`.

---

## Running the Benchmark

```bash
python benchmark.py
```

Clears the database, runs 5 questions about the same dataset, and shows progressive cache acceleration. Discovers all three MCP primitives and demos `ErrorHandlingMiddleware` catching a bad city name.

---

## Data Sources

| Source | API | Coverage |
|---|---|---|
| Open-Meteo Forecast | `api.open-meteo.com/v1/forecast` | Next 16 days |
| Open-Meteo Archive | `archive-api.open-meteo.com/v1/archive` | 1940–yesterday |

No API keys required. Data is fetched on demand and cached in `cityops.sqlite` for the session.
