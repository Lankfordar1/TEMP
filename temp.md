# Notes:


## Claude Skills
Skills are one of four parts:

* CLAUDE.md in the project root. This holds project facts: the segments, data sources, definitions (IC, ROIC, NOPAT), and rules such as “never overwrite raw files.” Claude Code loads it at the start of every session. 
* A mapping file (CSV or YAML). It maps each source row label to a standard field and a segment. This is the most important file, and you review it by hand.
* Python scripts, one per task: clean, populate the model, build the charts.
* Skills for the repeatable procedures. 
  * Project skills go in .claude/skills/<skill-name>/SKILL.md, and you can run one directly by typing /skill-name. 

### Example Folder Layout

```
sotp/
  CLAUDE.md
  config/mapping.csv
  data/raw/        (raw files, untouched)
  data/clean/
  scripts/01_clean.py
  scripts/02_populate_model.py
  scripts/03_charts.py
  output/
  .claude/skills/monthly-refresh/SKILL.md
```
---
## API ex.

```python
python
import pandas as pd
import fds.sdk.FactSetEstimates as fe
import fds.sdk.FactSetFundamentals as ff
from fds.sdk.FactSetEstimates.api import consensus_api
from fds.sdk.FactSetEstimates.models import RollingConsensusRequest
from fds.sdk.FactSetFundamentals.api import fact_set_fundamentals_api
from fds.sdk.FactSetFundamentals.models import *
from fds.sdk.utils.authentication import ConfidentialClient

client = ConfidentialClient("app-config.json")
ids = ["TMO-US", "DHR-US"]
TAX = 0.21
W = 0.125  # share of FY1 still remaining at the valuation date

# 1) Consensus EBIT for FY1, FY2, FY3
with fe.ApiClient(fe.Configuration(fds_oauth_client=client)) as c:
    req = RollingConsensusRequest(
        ids=ids, metrics=["EBIT"], periodicity="ANN",
        relative_fiscal_start=1, relative_fiscal_end=3, currency="USD")
    resp = consensus_api.ConsensusApi(c).get_rolling_consensus_for_list(req)
est = pd.DataFrame(resp.to_dict()["data"])
ebit = est.pivot_table(index="request_id", columns="relative_period", values="mean")

# 2) Last reported invested capital
with ff.ApiClient(ff.Configuration(fds_oauth_client=client)) as c:
    req = FundamentalsRequest(data=FundamentalRequestBody(
        ids=IdsBatchMax30000(ids), metrics=Metrics(["FF_INVEST_CAP"]),
        periodicity=Periodicity("ANN"), currency="USD"))
    resp = fact_set_fundamentals_api.FactSetFundamentalsApi(c) \
        .get_fds_fundamentals_for_list(req).get_response_200()
ic = pd.DataFrame(resp.to_dict()["data"]).groupby("request_id")["value"].last()

# 3) Time-weight into NTMA / STMA, then ROIC
ntma = W * ebit[1] + (1 - W) * ebit[2]
stma = W * ebit[2] + (1 - W) * ebit[3]
roic_2y = ((ntma + stma) / 2) * (1 - TAX) / ic
```
---

### file cleaning skill ex

#### Setup
 * Save the cleaning code as a .py script, not a notebook. A script runs with one command, while a notebook needs extra tooling to run unattended.
 * Create .claude/skills/clean-file/SKILL.md with the content below.
 * Open Claude Code in the project folder and type /clean-file.

markdown
---
name: clean-file
description: Clean Excel file. Use when the
  user says "clean the file" or adds a new file to data/raw.
---

1. Find the newest .xlsx file in data/raw/.
2. Run: python scripts/01_clean.py <that file>
3. Do not edit the script or the raw file.
4. Report: rows read, rows written, any unmapped row labels,
   and whether segments sum to the consolidated total.

---
## Damodaran Tax Rates by Country
```python
"""Pull this year's statutory corporate income tax rates from the OECD.

Source: OECD Corporate Income Tax Rates Database
        (dataflow OECD.CTP.TPS, DSD_TAX_CIT@DF_CIT), via the OECD SDMX API.

Usage:  python corporate_tax_rates.py
Needs:  pip install pandas requests
"""
import io
from datetime import date

import pandas as pd
import requests

COUNTRIES = {"JPN": "Japan", "USA": "United States", "ITA": "Italy"}
YEAR = date.today().year

URL = "https://sdmx.oecd.org/public/rest/data/OECD.CTP.TPS,DSD_TAX_CIT@DF_CIT"
PARAMS = {
    "startPeriod": YEAR - 1,          # one year back, in case this year is not published yet
    "format": "csvfilewithlabels",    # codes plus readable labels
}


def main() -> None:
    resp = requests.get(URL, params=PARAMS, timeout=60)
    resp.raise_for_status()
    df = pd.read_csv(io.StringIO(resp.text))

    needed = {"REF_AREA", "MEASURE", "TIME_PERIOD", "OBS_VALUE"}
    missing = needed - set(df.columns)
    if missing:
        # The OECD changed the layout: show what came back so the filter can be fixed.
        raise SystemExit(f"Missing columns {missing}. Columns returned: {list(df.columns)}")

    df = df[df["REF_AREA"].isin(COUNTRIES)].copy()
    df["TIME_PERIOD"] = df["TIME_PERIOD"].astype(int)

    # Use this year where published, otherwise the latest year available per country.
    df = df[df["TIME_PERIOD"] <= YEAR]
    latest = df.groupby("REF_AREA")["TIME_PERIOD"].transform("max")
    df = df[df["TIME_PERIOD"] == latest]

    label_col = "Measure" if "Measure" in df.columns else "MEASURE"
    df["Country"] = df["REF_AREA"].map(COUNTRIES)

    # 1) Every rate measure, so you can see what the OECD reports.
    all_rates = (
        df[["Country", "TIME_PERIOD", "MEASURE", label_col, "OBS_VALUE"]]
        .drop_duplicates()
        .sort_values(["Country", "MEASURE"])
    )
    print("All measures returned:\n")
    print(all_rates.to_string(index=False))

    # 2) The one to use for WACC: combined central + sub-central statutory rate.
    combined = df[df[label_col].astype(str).str.contains("combined", case=False)]
    combined = combined[~combined[label_col].astype(str).str.contains("small", case=False)]
    print("\nCombined statutory rate (use this for the after-tax cost of debt):\n")
    if combined.empty:
        print("No measure labelled 'combined' found. Pick the right row from the table above.")
    else:
        out = combined[["Country", "TIME_PERIOD", label_col, "OBS_VALUE"]].drop_duplicates()
        out = out.rename(columns={"TIME_PERIOD": "Year", label_col: "Measure", "OBS_VALUE": "Rate (%)"})
        print(out.to_string(index=False))


if __name__ == "__main__":
    main()
```
---

## OECD Tax Rates by Country
```python
"""Pull corporate marginal tax rates by country from Damodaran (NYU Stern).

Source: https://pages.stern.nyu.edu/~adamodar/pc/datasets/countrytaxrates.xls
        Underlying data is the KPMG corporate tax rate survey.
        The file carries two columns: the headline corporate rate and a rate
        adjusted for the global minimum tax (Pillar Two).

Usage:  python damodaran_tax_rates.py
Needs:  pip install pandas requests xlrd
        (xlrd is required for .xls; openpyxl does NOT read this format)
"""
import io

import pandas as pd
import requests

URL = "https://pages.stern.nyu.edu/~adamodar/pc/datasets/countrytaxrates.xls"
HEADERS = {"User-Agent": "Mozilla/5.0"}  # the site rejects some default agents


def load() -> pd.DataFrame:
    resp = requests.get(URL, headers=HEADERS, timeout=60)
    resp.raise_for_status()
    book = pd.ExcelFile(io.BytesIO(resp.content))
    print(f"Sheets in file: {book.sheet_names}\n")

    # The data sheet usually carries a few title rows above the header.
    # Scan the first 10 rows for the one that starts the country column.
    raw = book.parse(book.sheet_names[0], header=None)
    header_row = None
    for i in range(min(10, len(raw))):
        row = raw.iloc[i].astype(str).str.strip().str.lower()
        if row.str.startswith("country").any():
            header_row = i
            break
    if header_row is None:
        raise SystemExit(f"No header row found. First rows:\n{raw.head(10)}")

    df = book.parse(book.sheet_names[0], header=header_row)
    df.columns = [str(c).strip() for c in df.columns]
    return df


def main() -> None:
    df = load()
    print(f"Columns: {list(df.columns)}\n")

    country_col = df.columns[0]
    df[country_col] = df[country_col].astype(str).str.strip()

    # Keep every country; drop blank and footnote rows at the bottom of the sheet.
    out = df[df[country_col].ne("") & df[country_col].ne("nan")]

    pd.set_option("display.max_rows", None)
    print(out.to_string(index=False))
    print(f"\n{len(out)} rows.")

    # Rates may arrive as 0.22 or as 22.0 depending on the vintage — check before use.
    print("Note: confirm whether rates are decimals or percentages before using them.")


if __name__ == "__main__":
    main()
```

---
## Repo Outline and file types 
# How a Python + Claude Code repo is laid out

A reference guide: what the common file types are for, a sample repo that uses agents,
subagents, skills and Python scripts, and how the pieces connect.

---

## 1. The file types

### Files that configure the repository itself

| File | What it is | Who reads it | Committed to git? |
|---|---|---|---|
| `.gitignore` | A list of file patterns git should never track: secrets, data, caches, outputs | git | Yes |
| `.gitattributes` | Per-file git rules, e.g. line endings or "treat `*.xlsx` as binary" | git | Yes |
| `.gitkeep` | An empty placeholder. Git doesn't track empty folders, so this keeps one in the repo | git | Yes |
| `README.md` | The front page: what the project does and how to run it | People (GitHub/GitLab show it on the repo page) | Yes |
| `LICENSE` | Legal terms for reuse | People | Yes |

### Python project files

| File | What it is | Who reads it | Committed? |
|---|---|---|---|
| `pyproject.toml` | The modern single home for a Python project: name, version, dependencies, and tool settings (pytest, ruff, black...) | pip, build tools, linters | Yes |
| `requirements.txt` | A plain list of packages to `pip install -r`. Older, simpler alternative to the dependency list in `pyproject.toml` | pip | Yes |
| `uv.lock` / `poetry.lock` | Exact pinned versions of every package, so every machine installs the same thing | uv / poetry | Yes |
| `.python-version` | Which Python version the project expects (e.g. `3.11`) | pyenv, uv | Yes |
| `*.py` | Python code: a **script** you run, or a **module** that other code imports | Python | Yes |
| `__init__.py` | Marks a folder as a Python package so `import mypackage.thing` works | Python | Yes |
| `__main__.py` | Lets a package run with `python -m mypackage` | Python | Yes |
| `conftest.py` | Shared setup (fixtures) for tests | pytest | Yes |
| `__pycache__/`, `*.pyc` | Compiled bytecode Python writes automatically to load faster | Python | **No**, regenerated |
| `.venv/` | The project's isolated set of installed packages | Python | **No**, recreated from the dependency list |
| `*.ipynb` | A Jupyter notebook: code, output and notes in one file | Jupyter, VS Code | Usually yes, outputs cleared |

### Settings and data formats

| Extension | Format | Typical use | Notes |
|---|---|---|---|
| `.env` | `KEY=value` lines | Secrets and machine-specific settings (API keys, passwords, paths) | **Never committed.** Code reads it at runtime |
| `.env.example` | Same, with placeholders | Shows which keys a new user must supply | Committed; no real values |
| `.yaml` / `.yml` | YAML: indentation-based, allows comments | Human-edited config (pipeline settings), CI pipelines, Docker Compose | Easy to read; indentation errors are the common pitfall |
| `.toml` | TOML: `[sections]` and `key = value` | `pyproject.toml`, tool configs | Stricter than YAML, harder to break |
| `.json` | JSON: braces and quotes, **no comments** | Machine-written config, API payloads, Claude Code settings | Every tool can read it |
| `.ini` / `.cfg` | Old-style `[section]` + `key=value` | Older tool configs (`setup.cfg`, `pytest.ini`) | Being replaced by `pyproject.toml` |
| `.md` | Markdown: plain text with light formatting | Docs, READMEs, **and all Claude Code instructions** (CLAUDE.md, agents, skills) | Readable raw or rendered |
| `.j2` / `.jinja` | Jinja templates | HTML or text with `{{ placeholders }}` that code fills in | |
| `.csv` / `.parquet` / `.xlsx` | Data files | Inputs and outputs | Usually gitignored: large, often private |

**Choosing between YAML, TOML and JSON:**
- **YAML:** settings people edit by hand and want to comment.
- **TOML:** Python tooling (`pyproject.toml`).
- **JSON:** things programs write, or tools that require it.

### Automation files

| File | What it does |
|---|---|
| `.github/workflows/*.yml` | GitHub Actions: run tests or deploy on every push |
| `.gitlab-ci.yml` | GitLab's equivalent, at the repo root |
| `.pre-commit-config.yaml` | Checks that run automatically before each commit (formatting, linting, no secrets) |
| `Makefile` | Short aliases for long commands (`make test`) |
| `Dockerfile` | Recipe to package the project into a container |

### Claude Code files

| File | What it is | Scope |
|---|---|---|
| `CLAUDE.md` | Standing instructions the main session reads at startup: project rules, conventions, commands. It is guidance, not enforced | Shared with the team (committed) |
| `CLAUDE.local.md` | Your personal instructions for this project | You only (gitignored) |
| `.claude/settings.json` | Enforced settings: tool permissions, environment variables, hooks | Shared (committed) |
| `.claude/settings.local.json` | Your personal overrides of the above | You only (gitignored) |
| `.claude/agents/<name>.md` | A **subagent**: a specialist with its own instructions and tool list that the main session can hand a task to | Shared |
| `.claude/skills/<name>/SKILL.md` | A **skill**: a packaged procedure loaded only when relevant, or run directly as `/<name>`. Its folder can hold scripts and templates | Shared |
| `.claude/commands/<name>.md` | Older format for a `/command`; still works, but skills now cover the same job | Shared |
| `.mcp.json` | Connections to MCP servers (external tools and data) for this project. Lives at the repo **root**, not in `.claude/` | Shared |
| `~/.claude/...` | The same kinds of files in your home folder apply to **all** your projects | You only |

### Agents, subagents and skills

- **The agent** is the main Claude Code session you talk to. `CLAUDE.md` shapes how it behaves.
- **A subagent** is a separate worker the agent delegates to. It works in its own context with its own instructions and tools, and returns a result. Use one when a task is self-contained and would clutter the main conversation: a researcher, a reviewer, a step owner.
- **A skill** is a set of instructions (plus optional scripts) that any agent loads when the task calls for it: how to make a chart in house style, how to write the weekly summary. Only its short description is loaded until it's needed.
- **A Python script** does deterministic work: fetching data, fitting a model. Agents and skills *run* scripts; they don't replace them.

---

## 2. Sample repo: `sales-forecaster`

A small project that pulls sales data, builds features, fits a model, and writes a report.
It's driven by Claude Code with two subagents and two skills.

```
sales-forecaster/
├── README.md                      # what it is, how to run it
├── LICENSE
├── CLAUDE.md                      # main agent's standing instructions (committed)
├── CLAUDE.local.md                # your personal notes for Claude (gitignored)
├── .gitignore                     # keeps secrets, data, caches, outputs out of git
├── .env                           # SALES_API_KEY=...  (gitignored, never committed)
├── .env.example                   # SALES_API_KEY=<API_KEY>  (committed template)
├── .mcp.json                      # MCP servers for this project (e.g. a database)
├── .python-version                # 3.11
├── pyproject.toml                 # package name, dependencies, pytest/ruff settings
├── requirements.txt               # optional flat list for pip install -r
├── .pre-commit-config.yaml        # checks run before each commit
├── .gitlab-ci.yml                 # GitLab CI: run tests on push
├── .github/
│   └── workflows/
│       └── tests.yml              # GitHub Actions equivalent
│
├── .claude/                       # everything Claude Code reads for this project
│   ├── settings.json              # permissions, env vars, hooks (committed)
│   ├── settings.local.json        # your overrides (gitignored)
│   ├── agents/                    # SUBAGENTS: one markdown file each
│   │   ├── data-collector.md      #   fetches and validates data; may run ingest
│   │   └── reviewer.md            #   read-only: checks results, flags problems
│   └── skills/                    # SKILLS: one folder each, SKILL.md required
│       ├── house-chart/
│       │   ├── SKILL.md           #   how to draw charts in house style -> /house-chart
│       │   ├── style.yaml         #   colours and fonts the skill uses
│       │   └── scripts/
│       │       └── plot.py        #   helper the skill tells Claude to run
│       └── weekly-summary/
│           ├── SKILL.md           #   how to write the weekly note -> /weekly-summary
│           └── template.md        #   the layout to fill in
│
├── config/
│   └── settings.yaml              # run settings people edit: dates, target, model
│
├── src/
│   └── sales_forecaster/          # THE PACKAGE: importable code
│       ├── __init__.py            # makes this folder a package
│       ├── __main__.py            # enables: python -m sales_forecaster
│       ├── ingest.py              # pull data from the API / files
│       ├── features.py            # clean and build model inputs
│       ├── model.py               # fit and score
│       └── report.py              # write the HTML report
│
├── scripts/                       # entry points you run by hand
│   ├── run_pipeline.py            # ingest -> features -> model -> report
│   └── backfill.py                # one-off: reload history
│
├── notebooks/
│   └── explore.ipynb              # scratch analysis (outputs cleared before commit)
│
├── tests/
│   ├── conftest.py                # shared fixtures (sample data)
│   ├── test_features.py
│   └── test_model.py
│
├── data/
│   ├── raw/.gitkeep               # inputs land here (contents gitignored)
│   └── processed/.gitkeep
├── outputs/                       # reports and charts (gitignored)
│
├── .venv/                         # installed packages (gitignored)
└── **/__pycache__/                # compiled bytecode (gitignored)
```

### Why the code is split into `src/` and `scripts/`

- `src/sales_forecaster/` is a **package**: reusable functions with no side effects when imported. Tests import it, scripts import it, notebooks import it.
- `scripts/` are thin **entry points** that call the package in order. Keeping them thin means the logic is tested once, in the package.
- `pip install -e .` (driven by `pyproject.toml`) makes the package importable from anywhere in the repo.

---

## 3. What a few of these files contain

### `.gitignore`
```gitignore
# secrets
.env
.env.*
!.env.example
CLAUDE.local.md
.claude/settings.local.json

# environments and caches
.venv/
__pycache__/
*.pyc
.pytest_cache/

# data and outputs
data/raw/*
data/processed/*
!data/**/.gitkeep
outputs/
```
`!` means "except": the `!.env.example` line keeps the template tracked even though `.env.*` is ignored.

### `.env` and `.env.example`
```dotenv
# .env  (never committed)
SALES_API_KEY=sk_live_8f2...

# .env.example  (committed)
SALES_API_KEY=<API_KEY>
```

### `pyproject.toml`
```toml
[project]
name = "sales-forecaster"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["pandas>=2.0", "scikit-learn>=1.4", "pyyaml>=6.0", "requests>=2.31"]

[project.optional-dependencies]
dev = ["pytest>=8", "ruff>=0.5"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### `config/settings.yaml`
```yaml
run_name: q4_forecast
start_date: 2022-01-01
target: net_sales          # the Y variable
models: [linear, random_forest]
```

### `CLAUDE.md`
```markdown
# Project instructions
You run the sales forecast. Order: ingest -> features -> model -> report.
- Delegate data pulls to the data-collector subagent; send results to reviewer.
- Use the house-chart skill for every chart.
- Never print or commit .env. Run `pytest -q` after code changes.
Commands: `python scripts/run_pipeline.py`, `pytest -q`
```

### `.claude/agents/reviewer.md` (a subagent)
```markdown
---
name: reviewer
description: Checks model results and reports for errors, leakage and overfitting. Use after each run.
tools: Read, Grep, Glob
---
You review; you never edit files. Compare train vs test fit, check for variables that
restate the target, and list problems in order of severity.
```
The frontmatter between the `---` lines is YAML: `name`, when to use it (`description`), and which tools it may use (`tools`). The body is its instructions.

### `.claude/skills/house-chart/SKILL.md` (a skill)
```markdown
---
name: house-chart
description: Draw charts in the company house style. Use for any chart or plot.
---
1. Load colours and fonts from style.yaml in this folder.
2. Run scripts/plot.py with the data file and chart type.
3. Title says the finding; subtitle gives units; add a source line.
```
The folder name becomes the command `/house-chart`. Claude also loads the skill on its own when a task matches the description.

### `.claude/settings.json`
```json
{
  "permissions": {
    "allow": ["Bash(pytest:*)", "Bash(python scripts/*)"],
    "deny": ["Read(./.env)"]
  },
  "env": { "PYTHONUTF8": "1" }
}
```

---

## 4. How it fits together when you run it

```
you: "run the Q4 forecast"
  |
  v
MAIN AGENT  (reads CLAUDE.md, enforced by .claude/settings.json)
  |-- delegates --> data-collector SUBAGENT --runs--> scripts/run_pipeline.py --stage ingest
  |                                                     uses src/sales_forecaster/ingest.py
  |                                                     reads .env (API key), config/settings.yaml
  |                                                     writes data/raw/
  |-- runs -------> scripts/run_pipeline.py --stage model,report
  |                   writes outputs/report.html
  |-- loads ------> house-chart SKILL  --runs--> .claude/skills/house-chart/scripts/plot.py
  |-- delegates --> reviewer SUBAGENT (read-only) --> returns a list of issues
  v
you get: the report path + the reviewer's findings
```

### The rules of thumb

1. **Secrets in `.env`, settings in YAML, code in `.py`, instructions in `.md`.**
2. **If it can be regenerated, don't commit it:** caches, `.venv`, outputs and usually data.
3. **Shared and committed vs personal and gitignored:** `settings.json` / `CLAUDE.md` are for everyone; `settings.local.json` / `CLAUDE.local.md` are for you.
4. **Logic in the package, entry points in `scripts/`, judgement in agents, procedures in skills.**

---

## 5. Mapping this to the ML Modeling Project

| Sample repo | ML Modeling Project |
|---|---|
| `CLAUDE.md` | `CLAUDE.md`: the orchestrator; imports `docs/PROJECT_BRIEF.md` |
| `.claude/agents/` | six step owners: `01-ingestion.md` ... `06-reporting.md` |
| `.claude/skills/` | none yet. A report-style skill would be a natural first one |
| `.claude/commands/` | `run-pipeline.md` (the older command format) |
| `config/settings.yaml` | `config/pipeline.yaml` + `config/decisions/<run>.yaml` + `config/styles/` |
| `src/sales_forecaster/` | `src/mlpipe/` |
| `scripts/` | `scripts/step1_ingest.py` ... `step6_report.py`, `run_pipeline.py` |
| `data/raw/` | `ingestion/<folder>/` (inputs) and `data/raw/` (ingested) |
| `outputs/` | `runs/<run>/` and `deploy/` |

---

## Peer Comps Basic Outline
# Segment peer comps: how it fits together

**Goal:** plot each internal business segment against its public peers on metrics such as
EV/IC and ROIC − WACC, and fill an existing HTML report with the results.

**Why it needs three sources:** FactSet only has your company as a whole. Segment-level
numbers come from two internal files:

| Need | Source | Grain |
|---|---|---|
| Peer financials and market data (EV, invested capital, ROIC, WACC inputs) | FactSet: API, an MCP connector if your firm has one, or an Excel export | one row per **ticker × period** |
| Segment accounting (ROIC, invested capital, NPBT, taxes, interest income and expense) | Internal FP&A file, prepped by `FPNA_CLEAN.py` | one row per **segment × period** |
| Segment valuation (WACC, EV = PV of FCF + PV of terminal value, other DCF outputs) | The consultant's DCF output, built from the same FP&A file | one row per **segment × period** |

On every chart your segments appear next to their peers as `MYC_1`, `MYC_2`, `MYC_3`.

---

## 1. Repo layout

```
peer-comps/
├── CLAUDE.md                      # main agent: order of operations, rules below
├── .env / .env.example            # FACTSET_USERNAME / FACTSET_API_KEY (real values gitignored)
├── .gitignore                     # .env, data files, outputs
├── pyproject.toml
├── .claude/
│   ├── settings.json              # e.g. allow running scripts, deny reading .env
│   ├── agents/                    # SUBAGENTS
│   │   ├── factset-collector.md   #   pulls peer data, checks coverage per ticker
│   │   ├── fpna-prep.md           #   runs FPNA_CLEAN.py, explains the tie-out
│   │   └── reviewer.md            #   read-only: ties, outliers, definition mismatches
│   └── skills/                    # SKILLS (run on request)
│       ├── refresh-comps/SKILL.md #   /refresh-comps: full refresh, sources -> HTML
│       └── comps-chart/SKILL.md   #   /comps-chart: one chart, house style, export SVG
│
├── config/                        # everything you'd change by hand; no code edits
│   ├── peers.yaml                 # your ticker, segments, each segment's peer list + label
│   ├── segment_map.yaml           # FP&A sub-parts -> reported segment (A + B -> Segment 1)
│   └── metrics.yaml               # metric definitions: EV, IC, NOPAT, period basis
│
├── ingestion/                     # one folder per source (picked up automatically)
│   ├── factset_peers/fetch.py     # script source: FactSet API for every ticker in peers.yaml
│   ├── fpna/                      # script source
│   │   ├── FPNA_CLEAN.py          #   your existing prep script
│   │   └── <FP&A workbook>.xlsx   #   this month's file (gitignored)
│   └── consultant_dcf/            # file source
│       ├── <consultant DCF>.xlsx  #   (gitignored)
│       └── source.yaml            #   which cells/columns hold WACC, EV, PV FCF, PV TV
│
├── src/comps/                     # the package: logic, no hard-coded names
│   ├── rollup.py                  # sub-parts -> segments, leaf rows only
│   ├── tieout.py                  # rolled-up segments must equal the file's totals
│   ├── join.py                    # FP&A + DCF on segment; stack peers + segments
│   ├── labels.py                  # MYC_1, MYC_2 ... from peers.yaml
│   └── metrics.py                 # EV/IC, ROIC - WACC, any ratio in metrics.yaml
│
├── scripts/
│   ├── build_comps.py             # sources -> one tidy comps table
│   └── render_report.py           # comps table -> your HTML template
├── templates/
│   └── peer_comps.html            # your finished HTML, with the data swapped for a hook
├── outputs/                       # peer_comps_<period>.html, charts as SVG, tables (gitignored)
└── tests/
    ├── test_rollup_tieout.py      # A + B = X; segments sum to total
    └── test_metrics.py            # EV/IC and spread on a known example
```

---

## 2. How it fits together when you run it

```
you: "/refresh-comps for Q3 2026"
  |
  v
MAIN AGENT  (reads CLAUDE.md; config/ says who the peers and segments are)
  |
  |-- delegates --> factset-collector SUBAGENT
  |                   runs ingestion/factset_peers/fetch.py
  |                   reads config/peers.yaml (tickers), .env (FactSet credentials)
  |                   writes data/raw/factset_peers.csv            one row per ticker x period
  |                   reports: any ticker with missing fields, stale prices, currency
  |
  |-- delegates --> fpna-prep SUBAGENT
  |                   runs ingestion/fpna/FPNA_CLEAN.py on this month's FP&A workbook
  |                   keeps LEAF rows only (drops the subtotals that would double-count)
  |                   rolls sub-parts up with config/segment_map.yaml (A + B -> Segment 1)
  |                   TIE-OUT: rolled-up segments vs the file's own totals, within tolerance
  |                   writes data/raw/fpna_segments.csv            one row per segment x period
  |                   STOPS AND ASKS if the tie-out fails or a sub-part has no mapping
  |
  |-- runs -------> step 1 reads ingestion/consultant_dcf/ (file source, source.yaml)
  |                   writes data/raw/consultant_dcf.csv           WACC, EV, PV FCF, PV TV per segment
  |
  |-- runs -------> scripts/build_comps.py
  |                   joins FP&A + DCF on segment (same segment_map, so names agree)
  |                   stacks peers + segments into ONE table; labels MYC_1, MYC_2, MYC_3
  |                   computes metrics from config/metrics.yaml (EV/IC, ROIC - WACC ...)
  |                   writes outputs/comps_table.csv
  |
  |-- loads ------> comps-chart SKILL (house style)
  |-- runs -------> scripts/render_report.py
  |                   fills templates/peer_comps.html with the comps table
  |                   writes outputs/peer_comps_Q3_2026.html + each chart as SVG for slides
  |
  |-- delegates --> reviewer SUBAGENT (read-only)
  |                   checks: tie-out passed, every peer present, metric outliers,
  |                   definition mismatches (e.g. IC incl. vs excl. goodwill)
  v
you get: the HTML report, the comps table, the tie-out log, and the reviewer's flags
```

### What the combined table looks like

One row per label per period, whatever the source. The charts only ever read this table.

| label | type | segment | period | ev | invested_capital | roic | wacc | ev_ic | roic_minus_wacc | ev_basis |
|---|---|---|---|---|---|---|---|---|---|---|
| ABC | peer | Segment 1 | Q3 2026 | 12,400 | 6,100 | 11.2% | 8.9% | 2.03 | 2.3 pts | market |
| XYZ | peer | Segment 1 | Q3 2026 | 8,050 | 5,300 | 7.4% | 9.6% | 1.52 | −2.2 pts | market |
| MYC_1 | segment | Segment 1 | Q3 2026 | 9,700 | 4,200 | 13.1% | 8.4% | 2.31 | 4.7 pts | DCF |
| ABC | peer | Segment 2 | Q3 2026 | 12,400 | 6,100 | 11.2% | 8.9% | 2.03 | 2.3 pts | market |
| MYC_2 | segment | Segment 2 | Q3 2026 | … | … | … | … | … | … | DCF |

*(The numbers are illustrative.)* A peer appears once per segment it's benchmarked against.
A chart for Segment 1 filters `segment == "Segment 1"`, so the peers and `MYC_1` sit on the
same axes.

### The two config files that drive it

```yaml
# config/peers.yaml - who is compared with whom
own_ticker: MYC
segments:
  Segment 1: {label: MYC_1, peers: [ABC, XYZ]}
  Segment 2: {label: MYC_2, peers: [ABC, XYZ]}
  Segment 3: {label: MYC_3, peers: [ABC, XYZ]}     # peer lists can differ by segment
show_consolidated: false                             # true = also plot MYC (whole company) for reference
```

```yaml
# config/segment_map.yaml - how FP&A sub-parts roll up
Segment A: Segment 1       # A and B are two parts of Segment 1 (the "Segment X" case)
Segment B: Segment 1
Segment C: Segment 2
Segment D: Segment 3
consultant_names:          # if the DCF file names segments differently
  "Seg 1 - Optical": Segment 1
tie_out_tolerance: 0.5     # $mm; rolled-up total vs the file's reported total
```

Adding a peer, a segment or a metric is a config edit. Nothing in `src/` names a ticker or
a segment.

---

## 3. Where it should stop and ask

| Situation | Why it can't guess |
|---|---|
| A sub-part in the FP&A file isn't in `segment_map.yaml` | Mapping it to the wrong segment silently moves profit between segments |
| The tie-out fails | Usually a subtotal kept as a leaf, or a new line item; the fix needs judgement |
| A peer is missing a field in FactSet | Drop the peer, use an older period, or fill from another source: your call |
| Segment names in the DCF file don't match | The consultant's labels change; the mapping must be confirmed |
| Periods don't line up (FactSet LTM vs an FP&A forecast year) | Determines what's actually being compared; set once in `metrics.yaml` |

---

## 4. Rules of thumb for this deliverable

1. **Define the metric once, apply it to both sides.** EV, invested capital, NOPAT and the
   tax rate must mean the same thing for peers and segments. Examples: goodwill in or out
   of IC, operating leases, cash netted or not. Write it in `metrics.yaml`; `metrics.py`
   reads it.
2. **Tie out before you chart.** Rolled-up segments must equal the file's company total.
   As a second check, the sum of segments should be broadly consistent with FactSet's
   consolidated MYC figures, allowing for corporate and eliminations.
3. **Label the EV basis.** Peers' EV is market value; a segment's EV is the consultant's
   DCF value. EV/IC for `MYC_1` is "what the DCF implies" next to "what the market pays".
   Mark it on the chart (`ev_basis` column) so nobody reads it as like-for-like.
4. **Ratios travel, levels don't.** EV/IC and ROIC − WACC are currency-neutral. If you
   chart EV or IC levels across peers, convert them to one currency first. A WACC computed
   in a peer's local currency is fine inside its own spread, not compared raw across currencies.
5. **Labels are data.** `MYC_1` comes from `peers.yaml`. Renaming a segment or adding
   `MYC_4` never touches code.
6. **Keep your HTML; feed it.** Replace the hard-coded numbers in `peer_comps.html` with
   one hook, either `{{ placeholders }}` or a single embedded data block the charts read.
   `render_report.py` fills the hook from the comps table each period.
7. **Secrets in `.env`, data files gitignored, definitions in config, logic in `src/`.**

