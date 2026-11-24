# LLM Text-to-SQL Agent over SEC Filings

**Project LANTERN, Lab 4B: Ask the Panel** · DAMG 7245, Northeastern University, Fall 2026

**Team:** Pranav Waghmare, Preksha Praveen, Pradyumna Reddy Cherla

A natural-language query interface over SEC BDC filings. An analyst types a question; a LangGraph
agent asks Gemini for ONE SQL query as a Pydantic object, a read-only MCP server runs it, and the
answer comes back with the rows and the SQL behind it. Failed queries are repaired automatically.

Built on a real panel: 72 BDC filings, 21,944 current holdings, period 2025-12-31.
Model: `gemini-3.5-flash-lite`, recorded once and replayed from `cs4b/outputs/cache/` (no key needed).

**Demo video (under 10 minutes):** [Watch on Google Drive](https://drive.google.com/file/d/1mfk4bGIFduw5CiG_ecTTq2W524BoX6Py/view?usp=sharing)

## Architecture

![Architecture: one read-only path to the data](cs4b/docs/architecture.png)

- **One path to the data.** Nothing in the agent or the app imports `sqlite3` or opens the database;
  the server opens it read-only with an authorizer and runs every query through a SQL gate
  (one statement, row cap, timeout).
- **Rules twice.** The agent's Pydantic contract refuses bad SQL before any call; the server's gate
  is the authority.
- **Closed catalog.** Four READ tools: `list_tables`, `describe_table`, `sample_values`, `run_query`.
  Every call and every refusal is written to an audit ledger.
- **Repair, capped.** A refused or failed query goes back to the model with its error code and the
  failed SQL, at most 3 model calls per question; a budget guard stops runaway cost.
- **Follow-ups.** LangGraph's checkpointer keeps each conversation's history (question + SQL), so
  "which of those..." questions work.

## Setup and reproducibility (no API key needed)

```
cd cs4b
python -m venv .venv
source .venv/bin/activate              # Windows PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
python tests/run_tests.py              # 26/26 kit tests
python tools/build_db.py               # rebuilds data/panel.sqlite from data/real/
pytest -q tests/test_server.py         # the MCP server: catalog, SELECT, refusals
python agent/run.py --mode replay      # answers the ten questions from the cache
PYTHONPATH=src python -m askpanel.check   # 12/12   (PowerShell: $env:PYTHONPATH = "src")
streamlit run app/streamlit_app.py     # the chat app, replay by default
```

Record mode (only to re-record): set `GEMINI_API_KEY` (a free Google AI Studio key) and run
`python agent/run.py --mode record`. The key is never committed.

## What is where (inside `cs4b/`)

| File | What it is |
|---|---|
| `sql/holdings_v.sql` | The view the model reads (one row per current holding, `price`, `is_debt`) |
| `tools/clean_panel.py` | Drops unfunded commitments, keeps written-off loans at fair value 0 |
| `agent/contract.py`, `tests/test_contract.py` | The Pydantic contract and its refusal tests |
| `services/mcp/server.py`, `tests/test_server.py` | The read-only MCP query server and its tests |
| `agent/mcp_client.py` | One long-lived MCP session for the synchronous agent and app |
| `agent/model.py`, `agent/graph.py`, `agent/run.py` | Gemini behind record/replay, the LangGraph agent, the runner |
| `app/streamlit_app.py` | The chat app |
| `questions/questions.yaml`, `outputs/questions/`, `outputs/cache/` | Ten questions, saved and replayable |
| `outputs/economics.md`, `tools/economics.py` | Unit economics and the script behind every number |
| `outputs/failures.md` | Four failures, each with its fix |

## The ten questions

| id | category | question | outcome | calls | cost |
|---|---|---|---|---|---|
| q01 | lookup | What is the largest single holding by fair value in Fidus Investment Corp's current schedule? | answered, repaired (invalid column `adsch`) | 2 | $0.00115 |
| q02 | lookup | Which current holding at Blackstone Secured Lending Fund has the highest PIK rate, and what is that rate? | answered | 1 | $0.00062 |
| q03 | aggregation | What share of Antares Strategic Credit Fund's current fair value is first lien debt? | answered (also returns the sibling fund) | 1 | $0.00069 |
| q04 | aggregation | How many current holdings does Morgan Stanley Direct Lending Fund report, and what is their total fair value? | answered | 1 | $0.00061 |
| q05 | cross_bdc | Which five BDCs have the largest total fair value in their current schedules? | answered | 1 | $0.00060 |
| q06 | cross_bdc | Which ten BDCs have the lowest average price across their current debt holdings? | answered | 1 | $0.00061 |
| q07 | follow_up (q06) | Which holdings pull the lowest of those averages down the most? | answered, repaired (two timeouts) | 3 | $0.00241 |
| q08 | over_time | How did Kennedy Lewis Capital Co's total fair value and number of holdings change from the prior schedule to the current one? | answered | 1 | $0.00074 |
| q09 | over_time | Which five BDCs grew their total fair value the most, in dollars, from the prior schedule to the current one? | answered | 1 | $0.00094 |
| q10 | decline | Which of Golub Capital BDC 4's borrowers will default next year? | declined correctly | 1 | $0.00043 |

Totals: $0.00881 for 13 model calls; median SQL time 17 ms (see `cs4b/outputs/economics.md`).

## Using the app

For each question the app shows the answer, the SQL, the rows as a table, the cost and number of
model calls, and the trace (every graph step, including repairs). In replay mode a question is
answered only if it matches a recorded request exactly, history included: click
**New conversation**, then paste a question from the table above (ask q07 right after q06).

## MCP Inspector

The server in the MCP Inspector: the four tools, a SELECT on the real panel, and a write refused
by the gate.

![Inspector: run_query on the real panel](cs4b/docs/inspector.png)
![Inspector: DROP TABLE refused (not_select)](cs4b/docs/inspector_refusal.png)

## Known limitations

- **Sibling funds (q03).** The name match also catches ANTARES STRATEGIC CREDIT FUND II LLC; the
  answer shows the two funds as separate rows rather than one mixed total.
- **q07's scope.** "The lowest of those averages" means the single lowest BDC, but the SQL takes
  holdings from all ten low-average BDCs.
- **Undrawn commitments.** Some partly drawn loans still show low prices, because the panel does
  not say how much of the principal is drawn.
- **Lien type and industry.** `investment_type` and `industry` are empty in this panel; lien type
  is matched inside the holding's description.

## Tech stack

Python · LangGraph · Google Gemini API · Model Context Protocol (MCP) · Pydantic v2 · SQL / SQLite ·
Streamlit · pandas · PyArrow · pytest · Git/GitHub

Public SEC data only; every person in the scenario is fictional.
