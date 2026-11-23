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


- Give the user the file (or its path) with a short summary: what the repo does, how many Databricks sections were flagged, and anything that could not be determined.
- Then stop and wait for direction. Do not start rewriting or converting code.
- Delete the temp folder when the user says they are done, or at the end of the session. If deletion cannot be confirmed, tell the user the path so they can remove it.


