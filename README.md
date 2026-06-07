Create a file at C:\Users\<username>\.claude\skills\sql-data-analyst\references\setup.md with this exact content:

# One-Time Environment Setup

When this skill is used for the first time, check if conda environment exists and if not then create it.

## Step 1 - Check if environment exists
Run this command: conda env list
If sql_analyst_env is in the list then skip to verification step.

## Step 2 - Create conda environment
Run: conda create -n sql_analyst_env python=3.11 -y

## Step 3 - Install required packages
Run: conda run -n sql_analyst_env pip install pandas pyodbc matplotlib seaborn

## Step 4 - Verify installation
Run: conda run -n sql_analyst_env python -c "import pandas, pyodbc, matplotlib, seaborn; print('All packages OK')"

If output says All packages OK then environment is ready.

## Notes
- Always use conda run -n sql_analyst_env python to run scripts, never use plain python
- ODBC Driver 17 for SQL Server must be installed on the machine
- To check available ODBC drivers run: conda run -n sql_analyst_env python -c "import pyodbc; print(pyodbc.drivers())"
- If ODBC Driver 17 not found then use whatever SQL Server driver is available in the list




Create a file at C:\Users\<username>\.claude\skills\sql-data-analyst\references\analysis_template.md with this exact content:

# Analysis Script Template

When user asks to analyze a table or view, generate a Python script and save it to C:\Users\%USERNAME%\analyst_run.py then run it using conda run -n sql_analyst_env python C:\Users\%USERNAME%\analyst_run.py

## Connection string to use
DRIVER={ODBC Driver 17 for SQL Server};SERVER=<server>;DATABASE=<database>;Trusted_Connection=yes;

## Script must do these things in order

### 1. Connect and fetch data
import pyodbc
import pandas as pd
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import seaborn as sns
import base64
import io
import os
from datetime import datetime

Connect using Windows Auth trusted connection and load data into a pandas DataFrame called df.

### 2. Auto detect column types
numeric_cols = df.select_dtypes(include='number').columns.tolist()
datetime_cols = df.select_dtypes(include='datetime').columns.tolist()
cat_cols = categorical columns where nunique is less than 50

For object columns try to convert to datetime using pd.to_datetime with errors coerce, if more than 70 percent values convert successfully then keep as datetime.

### 3. Generate charts as base64 images
For each chart use this pattern:
- Create figure with plt.subplots
- Plot the data
- Save to BytesIO buffer with fig.savefig(buf, format='png', dpi=120, bbox_inches='tight')
- Convert to base64 string
- Embed in HTML as: <img src="data:image/png;base64,<base64string>">
- Always call plt.close(fig) after

### 4. Charts to generate
- Histogram for every numeric column
- Line chart over time if datetime column exists showing monthly average
- Horizontal bar chart for top 10 values of each categorical column
- Correlation heatmap if 2 or more numeric columns exist using seaborn heatmap

### 5. Auto insights to generate as bullet points
- Total rows and columns count
- For each numeric column report min, max, mean and flag if max is more than 3 times mean
- Flag any column where null percentage is more than 20 percent
- Report date range if datetime column exists
- Report top category and its percentage for each categorical column
- Report strongest correlation pair if correlation is above 0.7

### 6. HTML report structure
Single self-contained HTML file with dark theme background color #1e1e2e and text color #cdd6f4.
Sections in order: Header, Key Insights, Statistical Summary table, Column Overview table, Sample Data top 5 rows, all charts.
No external CSS, no CDN links, everything inline.
Save to C:/Users/<USERNAME>/Desktop/analysis_report.html using os.environ.get('USERNAME')

### 7. After saving report
Print to terminal: Report saved to Desktop as analysis_report.html
Print each insight as a bullet point in terminal
