---
name: sql-data-analyst
description: >
  Use this skill whenever the user wants to analyze data from a SQL Server database,
  fetch data from a table or view, run statistical analysis, generate visualizations,
  or get insights/trends from database data. Triggers include: "analyze this table",
  "show me trends for", "fetch data from view", "give me insights on", "create a report
  from SQL", "run analysis on my database", "visualize this query result". Always use
  this skill when the user mentions a table name, view name, or SQL query alongside
  words like analyze, trend, insight, report, or visualize.
---

# SQL Data Analyst Skill

Fetch data from SQL Server → analyze it with Python/pandas → produce a self-contained
HTML report with charts and insights. Output opens directly in the browser.

---

## Environment assumptions

- Office laptop running **Windows**
- **Miniconda** installed; skill uses a dedicated venv at `~/sql_analyst_env/`
- **SQL Server** with **Windows Authentication** (Trusted Connection)
- Claude Code CLI is the only interface available
- No external APIs; everything runs locally

---

## Step 0 — One-time setup (first use only)

Check if the venv exists. If not, run setup:

```bash
python ~/sql_analyst_env_setup.py
```

If the setup script doesn't exist yet, write it first — see `references/setup.md`.

To verify environment is ready:
```bash
~/sql_analyst_env/Scripts/python.exe -c "import pandas, pyodbc, matplotlib; print('OK')"
```

---

## Step 1 — Understand the request

Parse the user's request for:

| Parameter | What to extract |
|-----------|----------------|
| `server` | SQL Server name/IP — ask if not given |
| `database` | Database name — ask if not given |
| `source` | Table name, view name, or raw SQL query |
| `focus` | What to analyze (optional — will auto-detect if missing) |

If `server` or `database` is missing, ask the user before proceeding.

---

## Step 2 — Generate and run the analysis script

Write a Python script to `/tmp/analyst_run.py` using the template in `references/analysis_template.md`.

Key rules:
- Use **Windows Auth**: `Trusted_Connection=yes` in connection string
- Load data into a pandas DataFrame
- Auto-detect column types: numeric, datetime, categorical
- Compute statistics: describe(), value_counts(), correlations, trends
- Generate an **HTML report** (single self-contained file, no external CDN needed)
- Save report to the user's Desktop: `C:/Users/<username>/Desktop/analysis_report.html`
- Get username via: `os.environ.get('USERNAME')`

Run the script:
```bash
~/sql_analyst_env/Scripts/python.exe /tmp/analyst_run.py
```

---

## Step 3 — Report structure

The HTML report must contain these sections in order:

1. **Header** — Title, source table/view name, timestamp, row/col count
2. **Dataset Overview** — Shape, dtypes, null counts, sample rows (top 5)
3. **Statistical Summary** — `df.describe()` as a styled table
4. **Distribution Charts** — Histogram for each numeric column (inline SVG via matplotlib base64)
5. **Trend Analysis** — If datetime column exists: line chart of numeric columns over time
6. **Categorical Breakdown** — Bar chart for top-10 values in each categorical column
7. **Correlation Heatmap** — If 2+ numeric columns exist
8. **Key Insights** — Auto-generated bullet list (see insight rules below)

All charts must be **base64-embedded** in the HTML — no external files needed.

---

## Step 4 — Auto-generate insights

Apply these rules to produce the "Key Insights" section:

```
For each numeric column:
  - Flag if max > 3× mean (possible outliers)
  - Flag if null% > 20%
  - Report min, max, mean

For datetime column (if exists):
  - Report date range
  - Find month/period with highest and lowest values

For categorical columns:
  - Report top category and its share %
  - Flag if one category > 80% (dominance warning)

Overall:
  - Total rows analyzed
  - Columns with most nulls
  - Strongest correlation pair (if any)
```

---

## Step 5 — Present results

After the script runs successfully:

1. Tell the user: **"Report saved to your Desktop as `analysis_report.html` — open it in Chrome/Edge."**
2. Print a brief text summary in the terminal (5–8 bullet points of key findings)
3. If the script errored, show the error and fix it — do not ask the user to fix it

---

## Error handling

| Error | Fix |
|-------|-----|
| `pyodbc.Error: Data source name not found` | Wrong server name — ask user to confirm |
| `Login failed` | Windows Auth may not work for remote server — ask for server name again |
| `Table not found` | Check schema prefix: try `dbo.tablename` |
| `No module named pandas` | Re-run setup script |
| `Memory error` on large table | Add `TOP 100000` to the query automatically |

If data > 500,000 rows, automatically add `TOP 500000` to prevent memory issues and warn the user.

---

## Reference files

- `references/setup.md` — One-time environment setup script to write and run
- `references/analysis_template.md` — Full Python script template for the analysis
