# Robin → SpiderFoot Integration Plan

## Overview

Integrate Robin's AI-powered dark web OSINT pipeline into SpiderFoot as a set of REST API endpoints served by the existing CherryPy web server. After integration, the `robin/` folder is deleted.

---

## Background

### What Robin Does

Given a freeform query (e.g. `"ransomware group BlackCat"`), Robin runs a 5-step pipeline:

1. **Refine** — LLM optimizes the query into a ≤5-word dark web search string
2. **Search** — queries 16 `.onion` search engines in parallel via Tor SOCKS5 proxy
3. **Filter** — LLM selects the top 20 most relevant results from all collected links
4. **Scrape** — fetches and extracts text from those 20 `.onion` pages via Tor
5. **Summarize** — LLM produces a structured analyst report using a preset prompt

Supported LLM providers: OpenAI, Anthropic (Claude), Google Gemini, Ollama, LlamaCPP, OpenRouter.

### Why Robin's Search Engines Don't Overlap Much with SpiderFoot

SpiderFoot has 4 dark web modules: `sfp_ahmia`, `sfp_torch`, `sfp_onionsearchengine`, `sfp_onioncity`.
Robin uses 16 `.onion` search engines directly via Tor. Only Ahmia overlaps (partially — SpiderFoot uses the clearweb version, Robin uses the `.onion` address). The other 15 engines are not covered by SpiderFoot at all.

Additionally, SpiderFoot modules are **target-driven** (search for mentions of a specific scan target). Robin is **query-driven** (freeform investigation with LLM analysis). They serve different purposes.

---

## Architecture Decision

- Robin's core logic is moved into a new package: `spiderfoot/robin_osint/`
- 4 new endpoints are added to `sfwebui.py` following SpiderFoot's flat URL pattern
- Since Tor search + LLM calls take 30–120 seconds, `/robin_investigate` is **asynchronous** — it starts a background thread and returns a `job_id` immediately
- The caller polls `/robin_status?job_id=<id>` for the result
- **Note:** CherryPy's default dispatcher maps method names directly to URLs (underscores stay as underscores, no path nesting). `def robin_models` → `/robin_models`, not `/robin/models`. This is consistent with every existing SpiderFoot endpoint (`/startscan`, `/scanstatus`, `/scanopts`, etc.)
- Robin's Streamlit UI is **not** ported — replaced entirely by JSON API endpoints

---

## Step 1 — Create `spiderfoot/robin_osint/` Package

Create the directory `spiderfoot/robin_osint/` with the following 7 files:

### `spiderfoot/robin_osint/__init__.py`
Empty file — marks the directory as a Python package.

---

### `spiderfoot/robin_osint/config.py`
**Source:** `robin/config.py` — no changes needed.

Loads API keys from `.env` file or environment variables using `python-dotenv`.

```python
# Keys loaded:
OPENAI_API_KEY
ANTHROPIC_API_KEY
GOOGLE_API_KEY
OLLAMA_BASE_URL
OPENROUTER_BASE_URL
OPENROUTER_API_KEY
LLAMA_CPP_BASE_URL
```

---

### `spiderfoot/robin_osint/search.py`
**Source:** `robin/search.py` — no changes needed.

Contains:
- `SEARCH_ENGINES` — list of 16 `.onion` search engine URL templates
- `get_tor_session()` — creates a `requests.Session` routed through `socks5h://127.0.0.1:9050`
- `fetch_search_results(endpoint, query)` — scrapes one search engine for links
- `get_search_results(refined_query)` — runs all 16 engines concurrently via `ThreadPoolExecutor`, deduplicates results

---

### `spiderfoot/robin_osint/scrape.py`
**Source:** `robin/scrape.py` — no changes needed.

Contains:
- `scrape_single(url_data)` — fetches a single `.onion` URL via Tor, extracts text via BeautifulSoup
- `scrape_multiple(urls_data)` — scrapes up to 16 URLs concurrently, truncates each to 2000 chars

---

### `spiderfoot/robin_osint/llm_utils.py`
**Source:** `robin/llm_utils.py` — 2 changes:

1. Fix import path:
   - `from config import ...` → `from spiderfoot.robin_osint.config import ...`

2. Remove streaming callbacks (only needed for Streamlit UI):
   - Remove `BufferedStreamingHandler` class
   - Change `_common_llm_params` to `{"temperature": 0, "streaming": False}`

Contains:
- `_llm_config_map` — maps model name strings to LangChain class + constructor params
- `get_model_choices()` — returns list of available models based on configured API keys
- `resolve_model_config(model_choice)` — resolves model name to config dict
- `fetch_ollama_models()` — queries local Ollama for available models
- `fetch_llama_cpp_models()` — queries local llama.cpp for available models

---

### `spiderfoot/robin_osint/llm.py`
**Source:** `robin/llm.py` — 2 changes:

1. Fix import paths:
   - `from llm_utils import ...` → `from spiderfoot.robin_osint.llm_utils import ...`
   - `from config import ...` → `from spiderfoot.robin_osint.config import ...`

2. Replace `openai.RateLimitError` with `Exception` in `filter_results()` (removes hard `openai` dependency in the catch block).

Contains:
- `get_llm(model_choice)` — instantiates the LangChain LLM object
- `refine_query(llm, user_input)` — LLM call: optimize query for dark web search
- `filter_results(llm, query, results)` — LLM call: select top 20 relevant links
- `generate_summary(llm, query, content, preset, custom_instructions)` — LLM call: produce analyst report
- `PRESET_PROMPTS` — 4 analyst prompt templates:
  - `threat_intel` (default)
  - `ransomware_malware`
  - `personal_identity`
  - `corporate_espionage`

---

### `spiderfoot/robin_osint/health.py`
**Source:** `robin/health.py` — 1 change:

1. Fix import paths:
   - `from search import ...` → `from spiderfoot.robin_osint.search import ...`
   - `from llm import ...` → `from spiderfoot.robin_osint.llm import ...`
   - `from llm_utils import ...` → `from spiderfoot.robin_osint.llm_utils import ...`

Contains:
- `check_tor_proxy()` — tests TCP connection to `127.0.0.1:9050`, returns status + latency
- `check_llm_health(model_choice)` — sends a minimal `"Say OK"` prompt to the LLM, returns status + latency + provider name
- `check_search_engines()` — pings all 16 `.onion` engines via Tor concurrently, returns per-engine status

---

## Step 2 — Add Job Store to `SpiderFootWebUi.__init__`

Robin's pipeline is too slow for a synchronous HTTP response. Add an in-memory job store inside `__init__` in `sfwebui.py`:

```python
import threading
import uuid

# Inside SpiderFootWebUi.__init__, after existing setup:
self._robin_jobs = {}
self._robin_jobs_lock = threading.Lock()
```

`_robin_jobs` maps `job_id (str)` → `dict` with keys:
- `status` — `"running"` | `"done"` | `"error"`
- `result` — the markdown report string (when done)
- `error` — error message string (when error)

---

## Step 3 — Add 4 New Endpoints to `sfwebui.py`

Add the following imports at the top of `sfwebui.py`:

```python
import threading
import uuid
from spiderfoot.robin_osint.llm_utils import get_model_choices
from spiderfoot.robin_osint.llm import get_llm, refine_query, filter_results, generate_summary
from spiderfoot.robin_osint.search import get_search_results
from spiderfoot.robin_osint.scrape import scrape_multiple
from spiderfoot.robin_osint.health import check_tor_proxy, check_search_engines
```

All 4 endpoints are added inside the `SpiderFootWebUi` class after `scanelementtypediscovery`.

---

### Endpoint 1: `GET /robin_models`

Returns available LLM models based on configured API keys.

```
GET /robin_models

Response 200:
{
  "models": ["claude-sonnet-4-5", "gpt-4.1", "gemini-2.5-flash", ...]
}
```

Implementation:
```python
@cherrypy.expose
@cherrypy.tools.json_out()
def robin_models(self):
    try:
        return {"models": get_model_choices()}
    except Exception as e:
        return self.jsonify_error('500', str(e))
```

---

### Endpoint 2: `GET /robin_health`

Returns Tor proxy status and reachability of all 16 search engines.

```
GET /robin_health

Response 200:
{
  "tor": {"status": "up", "latency_ms": 120, "error": null},
  "engines": [
    {"name": "Ahmia",     "status": "up",   "latency_ms": 850, "error": null},
    {"name": "OnionLand", "status": "down", "latency_ms": null, "error": "timeout"},
    ...
  ]
}
```

Implementation:
```python
@cherrypy.expose
@cherrypy.tools.json_out()
def robin_health(self):
    try:
        return {
            "tor": check_tor_proxy(),
            "engines": check_search_engines()
        }
    except Exception as e:
        return self.jsonify_error('500', str(e))
```

---

### Endpoint 3: `POST /robin_investigate`

Starts the full Robin pipeline in a background thread. Returns a `job_id` immediately.

```
POST /robin_investigate
Content-Type: application/x-www-form-urlencoded

Parameters:
  query   (str, required)  — the investigation query
  model   (str, required)  — LLM model name (from /robin_models)
  preset  (str, optional)  — threat_intel | ransomware_malware | personal_identity | corporate_espionage
                             default: threat_intel
  custom  (str, optional)  — additional analyst instructions appended to the preset prompt

Response 200:
{
  "job_id": "a3f9c2d1",
  "status": "running"
}

Response 400 (missing params):
{
  "error": "query and model are required"
}
```

Background thread pipeline:
1. `llm = get_llm(model)`
2. `refined = refine_query(llm, query)`
3. `raw_results = get_search_results(refined)` — searches 16 engines via Tor
4. `top_results = filter_results(llm, refined, raw_results)` — LLM picks top 20
5. `scraped = scrape_multiple(top_results)` — fetch content from top 20 links
6. `content = "\n\n".join(f"{url}\n{text}" for url, text in scraped.items())`
7. `report = generate_summary(llm, query, content, preset, custom)`
8. Store `{"status": "done", "result": report}` in `self._robin_jobs[job_id]`

On any exception: store `{"status": "error", "error": str(e)}`.

---

### Endpoint 4: `GET /robin_status`

Polls the status and result of a job started by `/robin_investigate`.

```
GET /robin_status?job_id=a3f9c2d1

Response 200 — still running:
{
  "job_id": "a3f9c2d1",
  "status": "running"
}

Response 200 — completed:
{
  "job_id": "a3f9c2d1",
  "status": "done",
  "result": "## Investigation Report\n\n### Input Query: ...\n\n..."
}

Response 200 — failed:
{
  "job_id": "a3f9c2d1",
  "status": "error",
  "error": "Tor proxy unreachable: Connection refused"
}

Response 404 — unknown job:
{
  "error": "Job not found"
}
```

---

## Step 4 — Update `requirements.txt`

The following packages are already in SpiderFoot's `requirements.txt` and do **not** need to be added:
- `beautifulsoup4` ✓
- `pysocks` ✓
- `requests` ✓

Add the following new dependencies:

```
# Robin OSINT - LLM integration
python-dotenv>=1.0.0,<2
langchain-core>=0.3.0,<0.4
langchain-openai>=0.3.0,<0.4
langchain-anthropic>=0.3.0,<0.4
langchain-google-genai>=2.0.0,<3
langchain-ollama>=0.3.0,<0.4
```

---

## Step 5 — Delete `robin/` Folder

After verifying all endpoints work, delete the entire `robin/` directory from the repository root. Nothing in SpiderFoot's existing codebase references it.

---

## File Change Summary

| Action | Path | Notes |
|--------|------|-------|
| Create | `spiderfoot/robin_osint/__init__.py` | Empty package marker |
| Create | `spiderfoot/robin_osint/config.py` | Copied from `robin/config.py`, no changes |
| Create | `spiderfoot/robin_osint/search.py` | Copied from `robin/search.py`, no changes |
| Create | `spiderfoot/robin_osint/scrape.py` | Copied from `robin/scrape.py`, no changes |
| Create | `spiderfoot/robin_osint/llm_utils.py` | Fix imports + remove streaming callbacks |
| Create | `spiderfoot/robin_osint/llm.py` | Fix imports + replace `openai.RateLimitError` |
| Create | `spiderfoot/robin_osint/health.py` | Fix imports |
| Modify | `sfwebui.py` | Add job store to `__init__` + 4 new endpoints |
| Modify | `requirements.txt` | Add 6 LangChain packages |
| Delete | `robin/` | Entire folder removed after integration |

---

## What Is NOT Integrated

| Robin Component | Reason Excluded |
|----------------|-----------------|
| `ui.py` (Streamlit UI) | Replaced by JSON API endpoints |
| `entrypoint.sh` | SpiderFoot has its own entrypoint |
| `Dockerfile` | SpiderFoot has its own Docker setup |
| `.streamlit/` config | Streamlit-specific, not needed |
| Investigation save-to-file | Out of scope for API integration |

---

## API Usage Example (curl)

```bash
# 1. Check available models
curl http://127.0.0.1:5001/robin_models

# 2. Check Tor + engine health
curl http://127.0.0.1:5001/robin_health

# 3. Start an investigation
curl -X POST http://127.0.0.1:5001/robin_investigate \
  -d "query=BlackCat ransomware infrastructure" \
  -d "model=claude-sonnet-4-5" \
  -d "preset=ransomware_malware"
# Returns: {"job_id": "a3f9c2d1", "status": "running"}

# 4. Poll for result
curl http://127.0.0.1:5001/robin_status?job_id=a3f9c2d1
# Returns: {"job_id": "a3f9c2d1", "status": "done", "result": "..."}
```

---

## Prerequisites

- **Tor** must be running locally with SOCKS5 proxy on `127.0.0.1:9050`
  - Linux/WSL: `sudo apt install tor && sudo service tor start`
  - Mac: `brew install tor && brew services start tor`
- At least one LLM API key must be set in `.env` or environment variables
- Python 3.7+ (already required by SpiderFoot)
