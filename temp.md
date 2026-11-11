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