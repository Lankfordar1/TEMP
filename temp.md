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

---
Repo Decoder Skill
---
---
name: "databricks-repo-decoder"
description: "Use when asked to explain, map, reverse engineer, or repurpose a cloned Python/Databricks GitLab repo: flags Databricks code, builds a flow chart and plain-English code walkthrough."
---

# Databricks Repo Decoder

Purpose: let the user understand a cloned Python repository (usually from GitLab, usually built for Databricks) without opening every file, so they can replicate or repurpose the core logic outside Databricks. This is informational only.

## Ground rules (read first)

- **Read only.** Never edit, move, or delete anything in the cloned repo. "Commenting out" happens in the copy of the code shown in the HTML output, not in the repo files.
- **Run nothing from the repo.** No scripts, notebooks, tests, installs, CI, or jobs. Reading files and searching text is all this skill needs.
- **No contact with Databricks or any remote.** No Databricks CLI, SDK, or API calls, no tokens, no git fetch, pull, or push. Nothing is pulled from or posted to Databricks.
- **Create only the one HTML file below, in a temp folder.** Anything else (a cleaned copy of the code, converted scripts, new files) waits until the user explicitly asks.
- **Do not print secret values** (tokens, passwords, keys) found in the repo. Name the variable and where it lives.
- **Do not guess.** If a file was not read, or behavior depends on something outside the repo (a table, a job setting, another repo), say so.

## Step 1: Map the repo

1. Read `README`, `pyproject.toml` / `setup.py` / `requirements.txt`, `.gitlab-ci.yml`, and any job or workflow definitions (`databricks.yml`, bundle resources, job JSON or YAML).
2. Find the entry points: job tasks, notebooks (files starting with `# Databricks notebook source`), `main.py`, CLI commands.
3. Work out the order of operations from job task order and dependencies, `%run` and `dbutils.notebook.run` chains, imports, and function calls.

If the run order cannot be confirmed from the repo, show the best-supported order and label it as inferred.

## Step 2: Flag Databricks-specific code

Look for:

| Type | Examples |
|---|---|
| Notebook markers | `# Databricks notebook source`, `# COMMAND ----------`, `# MAGIC`, `%run`, `%sql`, `%pip` |
| Databricks utilities | `dbutils.secrets`, `dbutils.widgets`, `dbutils.fs`, `dbutils.notebook`, `display()` |
| Reading data | `spark.read`, `spark.table`, `spark.sql("SELECT ...")`, DBFS or mount paths (`dbfs:/`, `/dbfs/`, `/mnt/`), `catalog.schema.table` names |
| Writing data | `.write`, `.saveAsTable`, `INSERT` / `MERGE` / `CREATE TABLE`, Delta table operations, `dbutils.fs.put` / `cp` / `rm` |
| Platform services | `databricks-sdk`, `databricks-connect`, MLflow tracking or model registry, job and cluster configs, CI deploy steps |

For each flagged section record:

- File and line range
- What it does
- Why it is there
- Direction: **PULLS IN** data, **PUSHES OUT** data, or **runtime only**
- What the rest of the code needs from it (so the user knows what to replace)

Plain PySpark transformation logic (joins, filters, aggregations) is core project logic. Keep it and translate it, and note that it needs a Spark session to run.

## Step 3: Build the repo map

Create a temp folder (`mktemp -d` or the OS temp directory) and write one file there: `repo_map.html`. Make it self-contained: inline CSS and a small inline script, no external scripts, fonts, or CDN links, so opening it loads nothing from the network.

Write for a reader who does not work with code or data. The page should make sense to them without opening any technical section.

### Look and feel: Obsidian graph view

- Follow Obsidian's default theme: dark and light via `prefers-color-scheme`, neutral grays, one purple accent, system sans-serif font.
- **Graph:** circles joined by thin lines, drawn as inline SVG with a fixed layout that fits a phone screen.
  - Group the run into 4 to 7 **stages**. Each stage is a large lettered circle (A, B, C...) with a short label under it, joined in run order by arrows.
  - Each **step** is a small numbered circle attached to its stage.
  - Mark steps with a colored ring and a legend: Databricks, uses the internet, saves a file locally.
  - Tapping a circle highlights it and its links, fades the rest, and opens its note.
- **Notes:** one per stage and step, shown one at a time beside or below the graph, styled like an Obsidian note: title, `#tag` pills, purple internal links, callout boxes, and previous/next links that walk the run in order.
- If scripts are disabled, all notes should show in order.

### Page order

1. Title and two plain sentences on what the repo does.
2. Three status chips with counts: Databricks code, uses the internet, saves files locally.
3. The graph and the note panel. The opening note is "Start here": the whole run as one line per stage.
4. "Read in order": an outline of every stage and step as links.
5. "What leaves or lands on your computer", including a summary of every flagged Databricks section from Step 2.
6. "Not confirmed from the code" and a one-paragraph scope note.

### What each step note contains

- **Title:** the step number and 3 to 5 plain words (an action, not a function name).
- **What happens:** one to three short sentences. No jargon; if a technical term is needed, explain it in the same sentence.
- **If it goes wrong:** one sentence in a callout, only when the step has a branch or error path.
- **Good to know:** a callout for anything the reader would act on. Every Databricks section gets one: what it does, why it is there, and what would need to replace it.
- **For the technical reader** (collapsed): file and line ranges, short technical notes, and the original code with every Databricks-specific line commented out and prefixed `# [DATABRICKS - DISABLED]`.

For notebooks, the `# COMMAND ----------` cell breaks are natural step boundaries. Group small related cells into one step.

For a large repo (roughly 30 or more scripts), map one main path first (the main entry point or the README's example), say what was left out, then ask which other scripts to add.

## Step 4: Hand off and clean up

- Give the user the file (or its path) with a short summary: what the repo does, how many Databricks sections were flagged, and anything that could not be determined.
- Then stop and wait for direction. Do not start rewriting or converting code.
- Delete the temp folder when the user says they are done, or at the end of the session. If deletion cannot be confirmed, tell the user the path so they can remove it.

---
#Segment Vals

# Segment Valuation Reference

Working reference for segment-level DCF, peer comparison, and return-based
valuation metrics. Formulas, variable definitions, source mapping, and the
reasoning behind each choice.

All figures in this document are illustrative placeholders. No source data is
reproduced here.

---

## 1. Source labels

| Label | Description | Cadence |
|---|---|---|
| `GS_INPUT` | Third-party sum-of-parts valuation file. Segment DCF and peer multiples. | Quarterly |
| `FPNA` | Internal financial planning file. Segment P&L and selected balance sheet lines, actuals plus forecast. | Monthly |
| `MDS` | Market data service. Peer consensus estimates and reported fundamentals. | Daily |
| `TAX_SRC` | Published statutory corporate income tax rates by jurisdiction. | Annual |

`GS_INPUT` is the format being replicated. `FPNA` is the input feed. `MDS`
supplies everything on the peer side. The objective is to reproduce `GS_INPUT`
logic locally on the `FPNA` cadence.

---

## 2. Variable glossary

| Symbol | Definition | Source |
|---|---|---|
| $R_t$ | Revenue, period $t$ | `FPNA` |
| $E_t$ | EBITDA, period $t$ | `FPNA` |
| $D_t$ | Depreciation and amortization | `FPNA` |
| $EBIT_t$ | Earnings before interest and tax | `FPNA` |
| $\tau$ | Tax rate | `FPNA` / assumption |
| $NOPAT_t$ | Net operating profit after tax, unlevered | Derived |
| $NWC_t$ | Net working capital balance | `FPNA` |
| $C_t$ | Capital expenditure | `FPNA` |
| $UFCF_t$ | Unlevered free cash flow | Derived |
| $w$ | WACC | Assumption |
| $K_e$ | Cost of equity | Assumption |
| $m$ | Exit multiple, EV/EBITDA | Assumption |
| $g$ | Perpetual growth rate | Derived or assumed |
| $s$ | Stub fraction — share of the first forecast year after the valuation date | Derived from date |
| $\phi$ | Mid-year offset: `0.5` if mid-period convention, `0` if year-end | Convention switch |
| $n$ | Number of explicit forecast years after the stub | Model setting |
| $IC_t$ | Invested capital | `FPNA` |
| $ROIC_t$ | Return on invested capital | Derived |
| $EV$ | Enterprise value | Derived |
| $TV$ | Terminal value | Derived |
| $COV$ | Current operations value | Derived |
| $FGV$ | Future growth value | Derived |

---

## 3. Unlevered free cash flow

The build runs from EBITDA down. Every line is derivable from `FPNA` plus one
tax assumption.

$$EBIT_t = E_t - D_t$$

$$NOPAT_t = EBIT_t \times (1 - \tau)$$

$$\Delta NWC_t = NWC_{t-1} - NWC_t$$

$$UFCF_t = NOPAT_t + D_t + \Delta NWC_t - C_t$$

### Why unlevered

Interest is excluded deliberately. Interest is tax-deductible, so it lowers cash
tax paid, but the discount rate already captures that benefit through the
after-tax cost of debt. Including the interest shield in the cash flows as well
would count it twice.

**Consequence for the tax line:** apply $\tau$ to $EBIT$, not to pre-tax income.
Where a unit carries no interest the two are identical; where it does, the
`FPNA` tax line will be lower than the model's by $\tau \times$ net interest.
That difference is expected, not an error.

### Working capital sign

$\Delta NWC_t$ is the *change in the balance*, signed so that an increase in
working capital is a use of cash. Confirm the sign convention in `FPNA` before
wiring it up — payables stored as negatives invert the result, and the error is
silent.

---

## 4. Discounting

Two independent inputs, both often equal to 0.5. Keep them in separate cells.

| Input | Meaning | Mid-period on | Mid-period off |
|---|---|---|---|
| $s$ | Share of year 1 after the valuation date | 0.5 | 0.5 |
| $\phi$ | How early within a period cash arrives | 0.5 | 0 |

$s$ depends only on the valuation date and never changes with the convention.

**Discount period, stub year:**

$$t_0 = s \times (1 - \phi)$$

**Discount period, year $k$ (where $k = 1 \ldots n$):**

$$t_k = s + k - \phi$$

**Terminal period:**

$$t_{TV} = s + n$$

**Discount factor:**

$$DF_t = \frac{1}{(1 + w)^{t}}$$

**Present value of the stub year** — only the post-valuation-date portion counts:

$$PV_0 = UFCF_0 \times s \times DF_{t_0}$$

**Present value of each full year:**

$$PV_k = UFCF_k \times DF_{t_k}$$

---

## 5. Terminal value and enterprise value

Terminal-year figures extrapolate the final forecast year at a constant rate,
holding margins flat.

$$g_{term} = \frac{R_n}{R_{n-1}} - 1$$

$$E_{term} = R_n \times (1 + g_{term}) \times \frac{E_n}{R_n}$$

$$TV = E_{term} \times m$$

$$PV_{TV} = \frac{TV}{(1 + w)^{t_{TV}}}$$

$$EV = \sum_{k=0}^{n} PV_k + PV_{TV}$$

### Why the exit multiple and not a perpetuity

Both appear. The exit multiple is the *input*; the perpetuity rate it implies is
reported as a *check*. That ordering matters: an exit multiple is observable
from peers, a perpetual growth rate is not. Reporting the implied rate exposes
whether the chosen multiple embeds a growth assumption that survives scrutiny.

---

## 6. Implied perpetuity growth

Solve Gordon for $g$ given a terminal value that came from a multiple.

Starting point:

$$TV = \frac{F \times (1 + g)}{w - g}$$

where $F$ is terminal-year UFCF adjusted for intra-period timing:

$$F = UFCF_{term} \times (1 + w)^{\phi}$$

Rearranged:

$$g = \frac{TV \times w - F}{TV + F}$$

**Note on $\phi$:** under the year-end convention $\phi = 0$, so the adjustment
factor becomes 1 and $F = UFCF_{term}$. It is not 1 year — an exponent of 1
would imply cash arriving a full year early.

### Reading the result

Compare the output against long-run nominal GDP growth. A rate materially above
it means the exit multiple is carrying an assumption the business is unlikely to
sustain in perpetuity. This is the single most useful sanity check on a
multiple-based terminal value.

---

## 7. Implied forward multiple

$$\text{EV/EBITDA}_{fwd} = \frac{EV}{E_1}$$

Use the **first full forecast year**, not the terminal year. $EV$ is a present
value; terminal EBITDA sits $n$ years out. Dividing one by the other mixes
periods and produces a number that describes nothing.

Three distinct multiples, easily confused:

| Multiple | Formula | Role |
|---|---|---|
| Implied forward | $EV / E_1$ | Output. Compare to peers. |
| Exit | $m$ | Input assumption. |
| Terminal | $TV / E_{term}$ | Equals $m$ by construction. No information. |

A forward multiple above the exit multiple is normal when near-term growth
exceeds perpetual growth.

---

## 8. Source mapping

Three categories. Only the first changes on refresh.

**Pulled from `FPNA`:**

| Model line | Notes |
|---|---|
| Revenue | |
| Gross profit | |
| EBITDA | Use the unallocated line, not the fully-allocated one, if replicating `GS_INPUT` |
| D&A | |
| Capex | |
| NWC components | Receivables, inventory, payables |
| Invested capital | Check whether it is a formula or a hardcode |
| Tax rate | Point the model at this cell rather than hardcoding |

**Assumptions (one cell each):** $w$, $m$, valuation date, $s$, $\phi$, $n$.

**Calculated:** everything in sections 3 through 7.

### Layout guidance

Put `FPNA` inputs on a dedicated tab, laid out in the same row order as the
source file, so a monthly refresh is a paste rather than a re-map. Keep a
reconciliation row comparing each derived figure against the `FPNA` equivalent
where one exists; expected differences (the tax line above) should be documented
as expected, with a stated cause.

### Allocation layers

`FPNA`-style files often carry two EBITDA lines:

$$EBITDA_{allocated} = EBITDA_{unallocated} - \text{corporate charges}$$

Unallocated is before the unit's share of central costs. Peers report after all
their corporate costs, so **fully allocated is the like-for-like basis for peer
comparison**, while unallocated is typically what a sum-of-parts values, with
corporate costs deducted separately as a stub.

Check whether allocations persist across the whole forecast. Where they stop
after the actuals, the forecast years are not fully allocated regardless of
which line is labelled as such.

---

## 9. Peer estimate windows

`MDS` typically exposes three rolling windows:

| Window | Meaning |
|---|---|
| `LTMA` | Last twelve months, time-weighted |
| `NTMA` | Next twelve months, time-weighted |
| `STMA` | Second twelve months, time-weighted |

These are time-weighted blends of two adjacent fiscal years, sized to a 12-month
period. For a December year-end at valuation date $v$:

$$W = \text{fraction of the current fiscal year remaining at } v$$

$$NTMA = W \times FY_1 + (1 - W) \times FY_2$$

### Why rolling beats fixed fiscal years

1. Peers with non-December year-ends align automatically — no manual calendarization.
2. The window moves with the valuation date, so the metric does not go stale between refreshes.
3. `NTMA` closely tracks the first full forecast year, which keeps the multiple consistent with section 7.

**To apply the same basis to internal segments,** weight the forecast fiscal
years with the same $W$. Derive $W$ from the valuation date in a single cell.

**Verification:** pull a known entity's `NTMA` alongside its $FY_1$ and $FY_2$
annual consensus and confirm the weights reproduce it before applying them
anywhere else.

---

## 10. Forward growth rates

$$CAGR_{2y} = \left( \frac{V_{t+2}}{V_{t}} \right)^{1/2} - 1$$

General form, where $N$ is the number of compounding periods between the two
observations:

$$CAGR = \left( \frac{V_{end}}{V_{start}} \right)^{1/N} - 1$$

**Spreadsheet form**, deriving $N$ from the header row so the window can shift
without breaking:

```
=IFERROR((End/Start)^(1/($EndYear-$StartYear))-1,"")
```

Guard against a non-positive base when applying this to earnings rather than
revenue:

```
=IF(Start<=0,"n/m",(End/Start)^(1/($EndYear-$StartYear))-1)
```

### Choosing the window

"Forward" means the base is the current year, not a prior actual. Use $FY_1$ to
$FY_3$ on both sides. The rolling windows in section 9 cannot produce a
two-year *forward* span: `STMA` is only one year past `NTMA`, so `LTMA` to
`STMA` straddles one year back and one year forward.

**Coverage caveat:** $FY_3$ estimates are materially thinner than $FY_1$. Count
how many peers return a value before committing to the window; fall back to a
one-year forward rate and relabel the axis if coverage is poor.

---

## 11. Peer UFCF from estimates

Where consensus free cash flow is available, convert to unlevered:

$$UFCF = FCF + (I_{exp} - I_{inc}) \times (1 - \tau)$$

The adjustment is **after tax**. Consensus FCF is already net of tax, and that
tax was reduced by the interest deduction. Adding back gross interest leaves the
shield in the number and overstates UFCF by $\tau \times$ net interest.

For a net-cash entity the adjustment is negative and UFCF falls below FCF. That
is correct.

### When interest estimates are missing

Build from EBIT instead, which avoids interest entirely:

$$UFCF = E - (EBIT \times \tau) - C - \Delta NWC$$

This is the standard build with $D$ substituted out via $D = E - EBIT$, so no
D&A estimate is needed. EBITDA, EBIT and sales are among the best-covered
estimate items, so coverage gaps are rare.

**Approximating $\Delta NWC$** where consensus does not cover working capital —
hold it at a constant share of sales:

$$\Delta NWC \approx \frac{NWC_{reported}}{R_{reported}} \times (R_{est} - R_{prior})$$

Assumes working capital efficiency holds flat. Reasonable over one to two years.

**Checks before use:** confirm the sign convention on interest expense; pull all
components on the same period basis; confirm the vendor's FCF definition matches
operating cash flow less capex.

---

## 12. Tax rate for the cost of debt

$$K_{d,\,after\text{-}tax} = K_d \times (1 - \tau)$$

Use the **statutory** rate by jurisdiction, not an effective rate computed from
reported taxes over pre-tax income.

### Reasoning

| | Effective rate | Statutory rate |
|---|---|---|
| What it measures | Taxes actually recognized, including deferrals, credits, carryforwards, one-offs | The rate applying to the next dollar of taxable income |
| Stability | Swings year to year | Stable |
| Relevance to the shield | Indirect | Direct — the interest deduction saves tax at the marginal rate |

Effective rates introduce entity-specific accounting noise into a
cross-sectional comparison. Some entities return near-zero effective rates from
timing differences alone, which inflates their after-tax cost of debt.

**Magnitude:** the WACC impact is $\text{debt weight} \times K_d \times
\Delta\tau$. At a 20% debt weight and a 5% cost of debt, a 20-point tax rate
change moves WACC by roughly 0.2 points. Worth correcting; rarely reorders a
peer set.

### Implementation notes

- Use the **combined** central plus sub-central statutory rate.
- Apply a 15% floor to approximate the global minimum tax regime for
  low-rate jurisdictions.
- Domicile is a proxy for where income is earned. For an entity domiciled in a
  low-rate jurisdiction but operating elsewhere, use the operating jurisdiction.
- Loss-making entities cannot currently use the deduction. Flag rather than adjust.

---

## 13. Segment enterprise value

A segment has no observable market capitalization, so $EV$ must be constructed.
Two routes, used for different purposes.

**Route A — model value:**

$$EV_{model} = \sum PV(UFCF) + PV_{TV}$$

This is the DCF output from section 5. Note that it is **not independent of
peers**: $m$ typically comes from peer multiples, so solving for implied growth
using $EV_{model}$ returns the assumption you put in. Circular.

**Route B — peer-implied value:**

$$EV_{peer} = \text{median}(\text{EV/EBITDA})_{peers} \times E_{segment}$$

Use the **median**, not the mean. EV/EBITDA distributions are right-skewed; a
single peer with temporarily depressed EBITDA produces an extreme multiple that
drags the mean several turns.

Use Route B wherever the question is "what is the market pricing in," and Route
A wherever the question is "what does our forecast support." Where they diverge,
the gap is the finding.

### Comparability adjustments

Before plotting a segment against peers:

| Issue | Adjustment |
|---|---|
| Cost basis | Peer EV reflects all corporate costs. Use fully-allocated segment EBITDA, or deduct the PV of corporate costs from $EV_{peer}$. |
| Date | Model value is as of the valuation date; peer EV is as of the pull date. Align one to the other. |
| Invested capital definition | Confirm whether the peer source builds IC from the financing side (debt plus equity, including goodwill and cash) or the operating side. A narrower internal definition inflates segment EV/IC. |
| Period | Use the same point for IC on both sides. |

**Label model-derived positions explicitly.** A segment's position on a peer
chart reflects model assumptions, not an observed price.

---

## 14. Return-based valuation

### The value driver identity

$$\frac{EV}{IC} = \frac{ROIC - g}{w - g}$$

Derivation, from $EV = NOPAT \times (1 - g/ROIC) / (w - g)$ and
$NOPAT = ROIC \times IC$:

$$\frac{EV}{IC} = \frac{ROIC \times (1 - g/ROIC)}{w - g} = \frac{ROIC - g}{w - g}$$

Solved for $g$:

$$g = \frac{(EV/IC) \times w - ROIC}{(EV/IC) - 1}$$

Where:

$$ROIC_t = \frac{NOPAT_t}{IC_{t-1}} \qquad IC = NWC + \text{net fixed assets} + \text{goodwill and intangibles}$$

### Why this over the alternatives

| Alternative | Limitation |
|---|---|
| EV/EBITDA alone | Ignores capital intensity. Two units at the same multiple are not comparable if one employs twice the capital. |
| EV/IC alone | No benchmark. The ratio cannot say whether a given level is justified. |
| Empirical iso-curves (max / median / min of the sample) | Descriptive. They move when the sample changes, so a unit appears to re-rate when only the peer set did. |
| DCF output alone | Circular where the exit multiple came from peers. Cannot test its own input. |

The identity supplies the benchmark the ratio lacks: the EV/IC that a given
$ROIC$, $w$ and $g$ justify.

### Iso-growth lines

Rearranged, EV/IC is **linear** in ROIC:

$$\frac{EV}{IC} = \frac{ROIC}{w - g} - \frac{g}{w - g}$$

$$\text{slope} = \frac{1}{w - g} \qquad \text{intercept} = \frac{-g}{w - g}$$

So each value of $g$ is one straight line on a chart of EV/IC against ROIC.
These are theory-grounded bounds, unlike percentile bands drawn from the sample.

**The anchor point.** At $ROIC = w$:

$$\frac{EV}{IC} = \frac{w - g}{w - g} = 1$$

Every iso-growth line passes through $(w,\ 1.0)$ regardless of $g$. A unit
earning exactly its cost of capital is worth exactly the capital invested in it,
however fast it grows. Points above 1.0x represent returns above the cost of
capital; below 1.0x, value destruction.

This single fact makes the chart legible to a reader who has never seen it, and
is the caption worth putting under the exhibit.

**Guard:** where $EV/IC < 1$ the denominator in the solved form goes negative and
$g$ flips sign. Return `n/m`.

---

## 15. Fundamental growth

The growth rate an entity can fund from its own returns:

$$g_{fund} = ROIC \times RR$$

$$RR = \frac{C - D + \Delta NWC}{NOPAT}$$

### Use as a paired diagnostic

$g_{fund}$ and the implied $g$ from section 14 are not alternatives. Read the
gap:

| Relationship | Interpretation |
|---|---|
| Implied $g$ > $g_{fund}$ | Market expects returns or reinvestment to improve beyond current levels |
| Implied $g$ < $g_{fund}$ | Market is discounting the forecast, or the unit is out-earning its pricing |

**Constraint:** the identity assumes constant $ROIC$. A high current $ROIC$
cannot persist into perpetuity. For a terminal rate, fade $ROIC$ toward $w$ — at
$ROIC = w$, growth creates no value, which is the same statement as the anchor
point above.

---

## 16. Current operations value and future growth value

$$COV = \frac{NOPAT}{w} \qquad FGV = EV - COV$$

$COV$ is the value of current earnings held flat in perpetuity. $FGV$ is the
residual the market assigns to growth.

**Use current-period $NOPAT$, not terminal-period.** Terminal $NOPAT$ already
embeds the forecast growth, which is precisely what the split is designed to
isolate. Using it collapses $FGV$ and makes a growth business read as a value
business.

Either the last actual or the first forecast year is defensible. The first
forecast year is the better match to $EV$, which is forward-looking, and to the
`NTMA` basis used for peers. Apply the same choice to segments and peers.

**Normalize first:** strip non-operating items from the chosen period. A
one-time gain inside $EBIT$ inflates $COV$ and understates $FGV$.

$FGV$ as a share of $EV$ is the comparable output. Growth businesses typically
land in a 40–80% range; results far outside that warrant a check on the inputs.

---

## 17. Long-term growth from a market anchor

A variant that solves for the constant growth rate consistent with an observed
market value and two periods of near-term cash flow.

### Equity form

$$g = \frac{TV \times (1 + K_e)^{n} \times K_e - FCF_{n}}{TV \times (1 + K_e)^{n} + FCF_{n}}$$

Where:

- $TV = \text{Market cap} - PV(FCF_1) - PV(FCF_2)$
- $FCF_1$, $FCF_2$ from consensus, `NTMA` and `STMA` windows
- $n = 2$

**This is Gordon growth solved for $g$**, not a departure from it. $TV$ is
expressed as a present value, so it is grossed up by $(1 + K_e)^{n}$ before
being equated to $FCF_n(1+g)/(K_e - g)$. The algebra is identical to section 6.

The equity framing is internally consistent: equity value, cost of equity,
levered cash flow. Changing one requires changing all three.

### Unlevered form, for units with no observable market value

| Equity form | Unlevered form |
|---|---|
| Market cap | $EV_{peer}$ (section 13, Route B) |
| $K_e$ | $w$ |
| Levered FCF | $UFCF$ |

$$g = \frac{TV \times (1 + w)^{n} \times w - UFCF_{n}}{TV \times (1 + w)^{n} + UFCF_{n}}$$

$$TV = EV_{peer} - \sum_{k=1}^{n} PV(UFCF_k)$$

**Use $EV_{peer}$, not $EV_{model}$.** The equity form's anchor is an external
market price; the structural analogue is a peer-derived value, not a model
output. Substituting $EV_{model}$ reproduces the exit multiple's own implied
growth and answers nothing.

### Averaging peers

Average a **multiple**, never a dollar terminal value. Dollar values do not
scale across entities of different sizes. Take the peer median EV/EBITDA, apply
it to segment EBITDA, and derive $TV$ as the residual — the same structure as
the equity form, where market cap less near-term PV leaves the terminal
component.

### Horizon sensitivity

At $n = 2$ the terminal value is typically around 90% of total value, so $g$ is
solving for nearly the entire value at once and is highly sensitive to the
anchor multiple. Where a longer internal forecast exists, running $n = 4$ or
$n = 5$ moves more growth into the explicit period and produces a materially
lower, better-anchored rate.

Run both: $n = 2$ for the peer-comparable version, the longer horizon as the
internal estimate. Peers have no $FY_4$ consensus, so the longer run is
available only on the internal side.

### Relationship to section 14

The two implied-$g$ figures will not match. Section 14 is single-stage, so all
growth compresses into one perpetual rate; section 17 is two-stage. The
single-stage figure runs higher for a unit with strong near-term growth. Report
the method alongside the number.

---

## 18. Worked example

Illustrative placeholder values. Year-end convention ($\phi = 0$, $s = 1$),
$w = 10\%$, $\tau = 25\%$, $m = 10.0\times$, $IC_0 = 1{,}500$.

| Line | Y1 | Y2 |
|---|---|---|
| Revenue | 1,200 | 1,400 |
| EBITDA | 360 | 420 |
| D&A | 40 | 40 |
| EBIT | 320 | 380 |
| Tax at 25% | (80) | (95) |
| NOPAT | 240 | 285 |
| + D&A | 40 | 40 |
| Change in NWC | (30) | (35) |
| Capex | (60) | (70) |
| **UFCF** | **190** | **220** |
| Discount factor | 0.909 | 0.826 |
| **PV of UFCF** | **172.7** | **181.8** |

**Terminal and enterprise value**

| Item | Value |
|---|---|
| $g_{term}$ | 3.0% |
| Terminal EBITDA | 433 |
| Terminal value at $m = 10.0\times$ | 4,330 |
| PV of terminal value | 3,578.5 |
| PV of UFCF | 354.5 |
| **Enterprise value** | **3,933.0** |

**Derived metrics**

| Metric | Formula | Result |
|---|---|---|
| Implied forward EV/EBITDA | $3{,}933 / 360$ | 10.9x |
| Implied perpetuity growth | $(4{,}330 \times 0.10 - 226.6)/(4{,}330 + 226.6)$ | 4.5% |
| ROIC | $240 / 1{,}500$ | 16.0% |
| EV/IC | $3{,}933 / 1{,}500$ | 2.62x |
| Implied $g$, single-stage | $(2.62 \times 0.10 - 0.16)/(2.62 - 1)$ | 6.3% |
| Reinvestment rate | $(60 - 40 + 30)/240$ | 20.8% |
| Fundamental $g$ | $0.160 \times 0.208$ | 3.3% |
| COV | $240 / 0.10$ | 2,400 |
| FGV | $3{,}933 - 2{,}400$ | 1,533 |
| FGV as % of EV | | 39% |

**Identity check:** $(0.160 - 0.063)/(0.100 - 0.063) = 2.62$. Matches EV/IC.

Note the 4.5% versus 6.3% gap — the two-stage and single-stage methods differ as
described in section 17. Both are correct for their own framing.

---

## 19. Checks and failure modes

| Check | Why |
|---|---|
| Working capital sign convention | Inverted payables silently reverse $\Delta NWC$. Compare one year's derived UFCF against the source file's own FCF line. |
| Tax applied to EBIT, not pre-tax income | Double-counts the interest shield where the unit carries debt. |
| EBITDA allocation basis | Unallocated against peer multiples overstates value. |
| Invested capital definition match | The dominant driver of EV/IC comparability. |
| Stub fraction separate from mid-year offset | Two inputs that coincidentally share a value. |
| Discount exponent on $F$ | $\phi$, not 1. An exponent of 1 implies cash a full year early. |
| Forward multiple denominator | First full forecast year, never terminal. |
| $EV/IC < 1$ in the solved growth formula | Denominator flips sign. Return `n/m`. |
| Non-positive base in a CAGR | Returns a complex or nonsensical result. Guard it. |
| Implied $g$ against nominal GDP growth | A rate well above it indicts the multiple. |
| FGV share of EV in a plausible range | Catches normalization errors in $NOPAT$. |
| Estimate coverage count at $FY_3$ | Thin coverage silently biases a peer median. |
| Model-derived positions labelled as such | A model output on a peer chart is not an observed price. |

### Management-facing summary

> At $ROIC = w$, $EV/IC = 1.0\times$. A business earning exactly its cost of
> capital is worth exactly the capital invested in it, regardless of growth.
> Everything above that line is the market paying for returns in excess of the
> cost of capital; everything below is the market pricing in value destruction.

---

## 20. References

- Koller, Goeldhart and Wessels, *Valuation: Measuring and Managing the Value of
  Companies* — value driver formulas, ROIC and invested capital definitions,
  unlevered cash flow construction.
- Damodaran, *Investment Valuation* — fundamental growth, reinvestment rate,
  terminal value constraints, marginal versus effective tax rates.
- Stern Stewart EVA framework — COV and FGV decomposition.

---

*Reference only. Contains no source data.*
