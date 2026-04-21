# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SpiderFoot is an open source intelligence (OSINT) automation tool written in Python 3.7+. It integrates with 200+ data sources through a modular plugin architecture and includes a web UI, CLI, and correlation engine for analyzing scan results.

## Build and Run Commands

### Installation
```bash
pip3 install -r requirements.txt
```

### Running the Web UI
```bash
python3 ./sf.py -l 127.0.0.1:5001
```

### Running CLI Scans
```bash
# Start a scan with specific modules
python3 ./sf.py -s TARGET -m mod1,mod2,...

# List available modules
python3 ./sf.py -M

# Run correlation rules against a scan
python3 ./sf.py -C scanID

# Use the interactive CLI
python3 ./sfcli.py
```

### Testing

Install test dependencies:
```bash
pip3 install -r test/requirements.txt
```

Run unit and integration tests (excludes module integration tests):
```bash
./test/run
```

Run a single test file:
```bash
python3 -m pytest test/unit/test_spiderfoot.py
```

Run a single test:
```bash
python3 -m pytest test/unit/test_spiderfoot.py::TestSpiderFoot::test_method_name
```

Run all tests including module integration tests:
```bash
python3 -m pytest -n auto --flake8 --dist loadfile --durations=5 --cov-report html --cov=. .
```

Run only module integration tests:
```bash
python3 -m pytest -n auto --flake8 --dist loadfile --durations=5 --cov-report html --cov=. test/integration/modules/
```

Run acceptance tests (requires web server running on port 5001):
```bash
# Terminal 1
python3 ./sf.py -l 127.0.0.1:5001

# Terminal 2
cd test/acceptance
pip3 install -r requirements.txt
robot --variable BROWSER:Firefox --outputdir results scan.robot
```

### Linting
```bash
# Configuration in setup.cfg (flake8 section); max line length is 120
python3 -m flake8 .
```

## Architecture Overview

### Core Components

**Entry Points:**
- `sf.py`: Main entry point for both web UI (CherryPy-based) and CLI scans
- `sfcli.py`: Interactive CLI interface for SpiderFoot
- `sfwebui.py`: Web UI implementation (CherryPy routes and handlers)

**Core Libraries:**
- `sflib.py`: `SpiderFoot` class with common utilities (HTTP requests, DNS resolution, data validation, geo-location, etc.)
- `sfscan.py`: `SpiderFootScanner` class that orchestrates scan execution, module loading, and event flow
- `spiderfoot/`: Package containing core classes and utilities

**Database:**
- `spiderfoot/db.py`: `SpiderFootDb` class for SQLite operations
- Database path: `{data_path}/spiderfoot.db`
- Schema includes: scan instances, events/results, correlation results, configuration, logs

### Event-Driven Architecture

SpiderFoot uses a **publisher/subscriber event model**:

1. **SpiderFootEvent** (`spiderfoot/event.py`): Represents discovered data
   - Properties: `eventType`, `data`, `module`, `sourceEvent`, `confidence`, `visibility`, `risk`
   - Events form a chain: each event references its `sourceEvent` (the event that triggered it)
   - Events are hashed (SHA256) for uniqueness tracking

2. **SpiderFootPlugin** (`spiderfoot/plugin.py`): Base class for all modules
   - Modules subscribe to specific event types
   - When a module produces events, it calls `notifyListeners()` to publish to subscribers
   - Each module declares `meta` (name, summary, categories), `opts` (configuration), and `optdescs`
   - Key methods: `setup()`, `enrichTarget()`, `handleEvent()`, `watchedEvents()`, `producedEvents()`

3. **Event Flow:**
   - Scanner creates ROOT event from target
   - Modules process events they're subscribed to via `handleEvent()`
   - Modules emit new events via `notifyListeners()` which triggers subscribed modules
   - ThreadPool (`spiderfoot/threadpool.py`) manages concurrent module execution

### Module System

**Location:** `modules/` directory (200+ modules)

**Naming Convention:** `sfp_<module_name>.py`

**Module Structure:**
```python
class sfp_modulename(SpiderFootPlugin):
    meta = {
        'name': "Module Name",
        'summary': "What it does",
        'flags': [],  # e.g., ["slow", "apikey", "tool"]
        'useCases': ["Footprint", "Investigate", "Passive"],
        'categories': ["DNS", "Search Engines", etc.],
        # Required for modules that integrate with an external data source:
        'dataSource': {
            'website': "https://example.com",
            'model': "FREE_NOAUTH_UNLIMITED",  # or FREE_AUTH_LIMITED, COMMERCIAL_ONLY, etc.
            'references': ["https://example.com/api"],
            'description': "What the data source provides."
        }
    }

    opts = {}  # Default configuration options
    optdescs = {}  # Option descriptions

    results = None   # Tracks seen data; populated by self.tempStorage() in setup()
    errorState = False  # Set to True on fatal errors to skip further handleEvent() calls

    def setup(self, sfc, userOpts=dict()):
        self.sf = sfc
        self.results = self.tempStorage()  # dict-like; used for deduplication
        for opt in list(userOpts.keys()):
            self.opts[opt] = userOpts[opt]

    def watchedEvents(self):
        # Return list of event types this module subscribes to

    def producedEvents(self):
        # Return list of event types this module can produce

    def handleEvent(self, event):
        # Check self.errorState and return early if True
        # Check self.results for duplicates before processing
        # Process incoming events and emit new events via self.notifyListeners()
```

**Special Modules:**
- `sfp__stor_db.py`: Stores events to database
- `sfp__stor_stdout.py`: Outputs events to stdout

### Correlation Engine

**Location:** `correlations/` directory (37+ YAML rules)

**Purpose:** Post-scan analysis to identify patterns, anomalies, and security issues

**Architecture:**
- `spiderfoot/correlation.py`: `SpiderFootCorrelator` class
- Rules are YAML files defining: collections, aggregations, analysis, and headline templates
- Runs after scan completion (60 second timeout)
- Results stored in `tbl_scan_correlation_results` and `tbl_scan_correlation_results_events`

**Rule Structure:**
```yaml
id: rule_id
version: 1
meta:
  name: "Rule Name"
  description: "What it detects"
  risk: INFO|LOW|MEDIUM|HIGH
collections:
  - collect:
      - method: exact|regex
        field: type|module|data
        value: "match value"
aggregation:
  field: data|type|module  # Group results by this field
analysis:
  method: threshold|outlier|first_collection_only|match_all_to_first_collection
  # method-specific options
headline: "Result headline with {field} placeholders"
```

**Field Prefixes:** Use `source.`, `child.`, or `entity.` to reference related events in collections/aggregations/analysis

**Creating Rules:**
1. Copy `correlations/template.yaml` to `correlations/<rule_id>.yaml`
2. Edit rule definition
3. Restart SpiderFoot to load the new rule

See `correlations/README.md` for detailed documentation.

### Data Helpers

**SpiderFootHelpers** (`spiderfoot/helpers.py`):
- Data path management
- Username/password wordlist loading
- Domain/TLD parsing
- Data sanitization
- GEXF graph building for visualizations

**SpiderFootTarget** (`spiderfoot/target.py`):
- Represents scan target with aliases
- Target types: IP, domain, hostname, netblock, ASN, email, phone, username, person name, Bitcoin address

## Key Patterns and Conventions

### Configuration System

- Global config: `sfConfig` dict in `sf.py` (overridden by DB-stored config)
- Module-specific config: Each module's `opts` dict
- Config options prefixed by type: `_` for SpiderFoot core options, no prefix for module options
- Database stores per-scan and global configuration in `tbl_config` and `tbl_scan_config`

### Logging

- Uses Python `logging` module
- Custom `SpiderFootPluginLogger` preserves caller context
- Multiprocess logging via queue: `logger.py` provides `logListenerSetup` and `logWorkerSetup`
- Logs stored in `tbl_scan_log` table

### Threading

- `SpiderFootThreadPool` (`spiderfoot/threadpool.py`) manages module execution
- Default: 3 concurrent module threads (`_maxthreads` option)
- Each module runs in its own thread when processing events

### Data Validation

Common patterns in `sflib.py`:
- `validIP()`, `validIP6()`: IP address validation
- `validEmail()`: Email validation
- `validPhoneNumber()`: Phone number validation
- `isDomain()`, `isHostname()`: Domain/hostname validation
- `urlFQDN()`: Extract FQDN from URL

### HTTP Requests

- Primary method: `SpiderFoot.fetchUrl()` in `sflib.py`
- Honors SOCKS proxy settings (`_socks*` config options)
- Configurable User-Agent (`_useragent` option, can load from file with `@` prefix)
- Timeout: `_fetchtimeout` option (default: 5 seconds)
- SSL verification disabled for OSINT purposes

### DNS Resolution

- Custom DNS server support via `_dnsserver` option
- Common DNS methods in `sflib.py`: `resolveHost()`, `resolveIP()`, `resolveIP6()`
- Public Suffix List used for TLD parsing (`_internettlds` option)

## Development Workflow

### Adding a New Module

1. Create `modules/sfp_yourmodule.py` based on existing module structure
2. Define `meta`, `opts`, `optdescs` dictionaries
3. Implement `setup()`, `watchedEvents()`, `producedEvents()`, `handleEvent()`
4. Test with: `python3 ./sf.py -s TARGET -m sfp_yourmodule`
5. Add tests in `test/unit/modules/` and `test/integration/modules/`

### Modifying the Web UI

- Templates: `spiderfoot/templates/` (Mako templates)
- Static files: `spiderfoot/static/` (JS, CSS, images)
- Routes: `sfwebui.py` (CherryPy routes)
- Note: Web UI uses custom doc root path configurable via `_sfdocroot` option

### Database Schema Changes

- Schema defined in `SpiderFootDb.createSchemaQueries` in `spiderfoot/db.py`
- Migration logic in `SpiderFootDb.__init__()`
- Always test with fresh DB: delete `spiderfoot.db` and restart

## Important Notes

- **Security:** SpiderFoot intentionally disables SSL verification for OSINT data collection
- **Rate Limiting:** Modules should implement rate limiting for API calls
- **API Keys:** Many modules support API keys; store in module `opts`
- **Event Types:** Returned by `SpiderFootDb.eventTypes()` method in `spiderfoot/db.py` (e.g., `INTERNET_NAME`, `IP_ADDRESS`, `EMAILADDR`)
- **Target Types:** Must match one of the supported types in `SpiderFootTarget` validation
- **Python Version:** Requires Python 3.7+
- **External Tools:** Some modules call external tools (nmap, dnstwist, etc.) - check module flags for `tool` type

## Docker Support

- `Dockerfile`: Standard deployment
- `Dockerfile.full`: Includes external scanning tools
- `docker-compose.yml`, `docker-compose-dev.yml`, `docker-compose-full.yml`: Various deployment configurations
