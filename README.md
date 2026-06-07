## Step 0 — One-time setup check
Check if conda environment exists by running: conda env list
If sql_analyst_env is not in the list then run: conda create -n sql_analyst_env python=3.11 -y
Then run: conda run -n sql_analyst_env pip install pandas pyodbc matplotlib seaborn
To verify run: conda run -n sql_analyst_env python -c "import pandas, pyodbc, matplotlib; print('OK')"

## Step 1 — Understand the request
Ask user for these if not provided:
- server: SQL Server hostname or IP
- database: Database name
- source: Table name, view name, or raw SQL query
- focus: What to analyze, auto-detect if missing

## Step 2 — Generate and run the analysis script
Write a Python script to C:\Users\%USERNAME%\analyst_run.py with these rules:
- Connection string must use: DRIVER={ODBC Driver 17 for SQL Server};SERVER=<server>;DATABASE=<database>;Trusted_Connection=yes;
- Load data into pandas DataFrame
- Auto detect column types: numeric, datetime, categorical
- Compute statistics using describe(), value_counts(), correlations, trends
- Generate single self-contained HTML report with all charts as base64 embedded images
- Save report to desktop: C:/Users/<username>/Desktop/analysis_report.html
- Get username via os.environ.get('USERNAME')

Run the script using: conda run -n sql_analyst_env python C:\Users\%USERNAME%\analyst_run.py

## Step 3 — HTML Report must contain these sections
1. Header with title, source name, timestamp, row and column count
2. Dataset Overview with shape, dtypes, null counts, sample top 5 rows
3. Statistical Summary using df.describe() as styled table
4. Distribution Charts: histogram for each numeric column as base64 png
5. Trend Analysis: if datetime column exists show line chart of numeric columns over time
6. Categorical Breakdown: bar chart for top 10 values in each categorical column
7. Correlation Heatmap: if 2 or more numeric columns exist
8. Key Insights: auto generated bullet list

All charts must be base64 embedded in HTML, no external files, no CDN links.

## Step 4 — Auto generate insights rules
For each numeric column: flag if max is greater than 3 times mean as possible outlier, flag if null percentage is greater than 20 percent, report min max mean
For datetime column if exists: report date range, find month with highest and lowest values
For categorical columns: report top category and its share percentage, flag if one category is more than 80 percent as dominance warning
Overall: report total rows analyzed, columns with most nulls, strongest correlation pair if any

## Step 5 — After script runs successfully
Tell user: Report saved to your Desktop as analysis_report.html, open it in Chrome or Edge
Print brief text summary in terminal with 5 to 8 bullet points of key findings
If script errored then fix it automatically, do not ask user to fix it

## Error handling
If pyodbc error data source not found then wrong server name, ask user to confirm
If login failed then Windows Auth may not work, ask for server name again
If table not found then check schema prefix, try dbo.tablename
If no module named pandas then re run setup
If memory error on large table then add TOP 100000 to query automatically
If data is more than 500000 rows then automatically add TOP 500000 and warn user
If ODBC Driver 17 not found then try ODBC Driver 13 for SQL Server or SQL Server as driver name

5. After creating the SKILL.md file, confirm the full path where it was saved.
