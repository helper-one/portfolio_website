# Analysis Script Template

Claude generates this script dynamically, filling in placeholders, then saves it to
`/tmp/analyst_run.py` and executes it with the venv Python.

## Placeholders to fill

| Placeholder | Replace with |
|-------------|-------------|
| `{{SERVER}}` | SQL Server hostname/IP from user |
| `{{DATABASE}}` | Database name from user |
| `{{SOURCE_QUERY}}` | `SELECT * FROM dbo.tablename` or user's query |
| `{{REPORT_TITLE}}` | Friendly title e.g. "Analysis: vw_IAM_Metrics" |

## Full script template

```python
import pyodbc
import pandas as pd
import matplotlib
matplotlib.use('Agg')  # non-interactive backend — MUST be before pyplot import
import matplotlib.pyplot as plt
import matplotlib.colors as mcolors
import seaborn as sns
import base64
import io
import os
import json
from datetime import datetime

# ── Config ────────────────────────────────────────────────────────────────────
SERVER   = "{{SERVER}}"
DATABASE = "{{DATABASE}}"
QUERY    = "{{SOURCE_QUERY}}"
TITLE    = "{{REPORT_TITLE}}"
USERNAME = os.environ.get('USERNAME', 'user')
OUTPUT   = f"C:/Users/{USERNAME}/Desktop/analysis_report.html"

# ── Connect & fetch ───────────────────────────────────────────────────────────
print("Connecting to SQL Server...")
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={SERVER};"
    f"DATABASE={DATABASE};"
    f"Trusted_Connection=yes;"
)
conn = pyodbc.connect(conn_str)
print(f"Running query...")
df = pd.read_sql(QUERY, conn)
conn.close()
print(f"Fetched {len(df):,} rows × {len(df.columns)} columns")

# ── Auto-detect column types ──────────────────────────────────────────────────
# Convert obvious date columns
for col in df.columns:
    if df[col].dtype == object:
        try:
            converted = pd.to_datetime(df[col], infer_datetime_format=True, errors='coerce')
            if converted.notna().sum() > len(df) * 0.7:
                df[col] = converted
        except:
            pass

numeric_cols  = df.select_dtypes(include='number').columns.tolist()
datetime_cols = df.select_dtypes(include='datetime').columns.tolist()
cat_cols      = [c for c in df.select_dtypes(include='object').columns
                 if df[c].nunique() < 50]  # only low-cardinality as categorical

# ── Helper: fig → base64 HTML img tag ─────────────────────────────────────────
def fig_to_b64(fig):
    buf = io.BytesIO()
    fig.savefig(buf, format='png', dpi=120, bbox_inches='tight',
                facecolor='#1e1e2e', edgecolor='none')
    buf.seek(0)
    b64 = base64.b64encode(buf.read()).decode()
    plt.close(fig)
    return f'<img src="data:image/png;base64,{b64}" style="max-width:100%;border-radius:8px;margin:10px 0;">'

# ── Styling constants ──────────────────────────────────────────────────────────
BG      = '#1e1e2e'
SURFACE = '#2a2a3e'
ACCENT  = '#7c6af7'
TEXT    = '#cdd6f4'
GREEN   = '#a6e3a1'
RED     = '#f38ba8'
YELLOW  = '#f9e2af'
plt.rcParams.update({
    'figure.facecolor': BG, 'axes.facecolor': SURFACE,
    'axes.edgecolor': '#444', 'axes.labelcolor': TEXT,
    'xtick.color': TEXT, 'ytick.color': TEXT,
    'text.color': TEXT, 'grid.color': '#333', 'grid.linestyle': '--',
    'grid.alpha': 0.5,
})

charts_html = ""

# ── Distribution histograms ────────────────────────────────────────────────────
if numeric_cols:
    n = len(numeric_cols)
    cols_per_row = 2
    rows = (n + cols_per_row - 1) // cols_per_row
    fig, axes = plt.subplots(rows, cols_per_row, figsize=(14, 4 * rows))
    axes = [axes] if rows == 1 and cols_per_row == 1 else axes
    axes_flat = [ax for row in (axes if rows > 1 else [axes]) for ax in (row if hasattr(row, '__iter__') else [row])]
    for i, col in enumerate(numeric_cols):
        ax = axes_flat[i]
        data = df[col].dropna()
        ax.hist(data, bins=30, color=ACCENT, alpha=0.85, edgecolor='none')
        ax.set_title(col, color=TEXT, fontsize=11, pad=8)
        ax.set_xlabel('Value', fontsize=9)
        ax.set_ylabel('Count', fontsize=9)
        ax.grid(True)
    for j in range(i + 1, len(axes_flat)):
        axes_flat[j].set_visible(False)
    fig.suptitle('Distributions', color=TEXT, fontsize=14, y=1.01)
    charts_html += "<h2>📊 Distributions</h2>" + fig_to_b64(fig)

# ── Trend analysis (datetime × numeric) ───────────────────────────────────────
if datetime_cols and numeric_cols:
    date_col = datetime_cols[0]
    df_sorted = df.sort_values(date_col)
    fig, ax = plt.subplots(figsize=(14, 5))
    colors = [ACCENT, GREEN, YELLOW, RED, '#89dceb']
    for i, col in enumerate(numeric_cols[:5]):
        monthly = df_sorted.groupby(pd.Grouper(key=date_col, freq='ME'))[col].mean()
        ax.plot(monthly.index, monthly.values, color=colors[i % len(colors)],
                linewidth=2, marker='o', markersize=4, label=col)
    ax.set_title('Trends Over Time (Monthly Average)', color=TEXT, fontsize=13)
    ax.set_xlabel('Date', fontsize=10)
    ax.set_ylabel('Average Value', fontsize=10)
    ax.legend(loc='upper left', facecolor=SURFACE, edgecolor='#444', labelcolor=TEXT)
    ax.grid(True)
    charts_html += "<h2>📈 Trends Over Time</h2>" + fig_to_b64(fig)

# ── Categorical breakdowns ─────────────────────────────────────────────────────
if cat_cols:
    for col in cat_cols[:4]:  # max 4 categorical charts
        vc = df[col].value_counts().head(10)
        fig, ax = plt.subplots(figsize=(10, 4))
        bars = ax.barh(vc.index.astype(str)[::-1], vc.values[::-1],
                       color=ACCENT, alpha=0.85)
        for bar, val in zip(bars, vc.values[::-1]):
            ax.text(bar.get_width() + 0.3, bar.get_y() + bar.get_height()/2,
                    f'{val:,}', va='center', fontsize=9, color=TEXT)
        ax.set_title(f'Top Values — {col}', color=TEXT, fontsize=12)
        ax.set_xlabel('Count', fontsize=9)
        ax.grid(True, axis='x')
        charts_html += f"<h2>🏷️ {col} Breakdown</h2>" + fig_to_b64(fig)

# ── Correlation heatmap ────────────────────────────────────────────────────────
if len(numeric_cols) >= 2:
    corr = df[numeric_cols].corr()
    fig, ax = plt.subplots(figsize=(max(6, len(numeric_cols)), max(5, len(numeric_cols) - 1)))
    cmap = sns.diverging_palette(240, 10, as_cmap=True)
    sns.heatmap(corr, annot=True, fmt='.2f', cmap=cmap, ax=ax,
                linewidths=0.5, linecolor='#333',
                annot_kws={'size': 9, 'color': TEXT},
                cbar_kws={'shrink': 0.8})
    ax.set_title('Correlation Matrix', color=TEXT, fontsize=13, pad=12)
    ax.tick_params(colors=TEXT)
    charts_html += "<h2>🔗 Correlation Matrix</h2>" + fig_to_b64(fig)

# ── Auto-insights ──────────────────────────────────────────────────────────────
insights = []
insights.append(f"✅ Total rows analyzed: <strong>{len(df):,}</strong>")
insights.append(f"📋 Columns: <strong>{len(df.columns)}</strong> "
                f"({len(numeric_cols)} numeric, {len(datetime_cols)} datetime, {len(cat_cols)} categorical)")

null_pct = (df.isnull().sum() / len(df) * 100).sort_values(ascending=False)
high_null = null_pct[null_pct > 20]
if not high_null.empty:
    for col, pct in high_null.items():
        insights.append(f"⚠️ Column <strong>{col}</strong> has {pct:.1f}% missing values")

for col in numeric_cols:
    s = df[col].dropna()
    if len(s) == 0: continue
    mean_val = s.mean()
    max_val  = s.max()
    if mean_val != 0 and max_val > 3 * abs(mean_val):
        insights.append(f"⚠️ <strong>{col}</strong>: max ({max_val:,.0f}) is >3× the mean ({mean_val:,.0f}) — possible outliers")

if datetime_cols and numeric_cols:
    date_col = datetime_cols[0]
    date_range = f"{df[date_col].min().date()} → {df[date_col].max().date()}"
    insights.append(f"📅 Date range: <strong>{date_range}</strong>")

if cat_cols:
    for col in cat_cols[:2]:
        top_val = df[col].value_counts().idxmax()
        top_pct = df[col].value_counts(normalize=True).max() * 100
        insights.append(f"🏆 Top <strong>{col}</strong>: <em>{top_val}</em> ({top_pct:.1f}% of records)")
        if top_pct > 80:
            insights.append(f"⚠️ <strong>{col}</strong> is dominated by one value — low diversity")

if len(numeric_cols) >= 2:
    corr = df[numeric_cols].corr().abs()
    corr_unstacked = corr.where(
        ~pd.np.eye(len(corr), dtype=bool) if hasattr(pd, 'np')
        else corr.values != 1
    ).stack()
    # safe version
    mask = corr != 1.0
    upper = corr.where(pd.DataFrame(
        [[i < j for j in range(len(corr.columns))] for i in range(len(corr.columns))],
        index=corr.index, columns=corr.columns
    ))
    stacked = upper.stack()
    if not stacked.empty:
        best_pair = stacked.idxmax()
        best_val  = stacked.max()
        if best_val > 0.7:
            insights.append(f"🔗 Strong correlation: <strong>{best_pair[0]}</strong> ↔ <strong>{best_pair[1]}</strong> (r = {best_val:.2f})")

insights_html = "".join(f"<li>{i}</li>" for i in insights)

# ── Statistical summary table ──────────────────────────────────────────────────
if numeric_cols:
    desc_html = df[numeric_cols].describe().round(2).to_html(
        classes='stats-table', border=0
    )
else:
    desc_html = "<p>No numeric columns found.</p>"

# ── Null summary table ─────────────────────────────────────────────────────────
null_df = pd.DataFrame({
    'Column': df.columns,
    'Dtype': df.dtypes.astype(str).values,
    'Nulls': df.isnull().sum().values,
    'Null %': (df.isnull().sum() / len(df) * 100).round(1).values
})
null_html = null_df.to_html(index=False, classes='stats-table', border=0)

# ── Sample rows ────────────────────────────────────────────────────────────────
sample_html = df.head(5).to_html(index=False, classes='stats-table', border=0)

# ── Assemble HTML ──────────────────────────────────────────────────────────────
timestamp = datetime.now().strftime("%d %b %Y, %H:%M")
html = f"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{TITLE}</title>
<style>
  * {{ box-sizing: border-box; margin: 0; padding: 0; }}
  body {{ background: {BG}; color: {TEXT}; font-family: 'Segoe UI', sans-serif;
          font-size: 14px; line-height: 1.6; padding: 0; }}
  .hero {{ background: linear-gradient(135deg, #2a1f6f 0%, #1e1e2e 100%);
           padding: 40px 40px 30px; border-bottom: 1px solid #333; }}
  .hero h1 {{ font-size: 26px; color: #fff; margin-bottom: 6px; }}
  .hero .meta {{ color: #888; font-size: 13px; }}
  .hero .badges {{ margin-top: 12px; display: flex; gap: 8px; flex-wrap: wrap; }}
  .badge {{ background: {SURFACE}; border: 1px solid #444; border-radius: 20px;
            padding: 3px 12px; font-size: 12px; color: {TEXT}; }}
  .container {{ max-width: 1100px; margin: 0 auto; padding: 30px 24px; }}
  h2 {{ font-size: 18px; color: #fff; margin: 32px 0 12px;
        border-left: 3px solid {ACCENT}; padding-left: 10px; }}
  .insights {{ background: {SURFACE}; border-radius: 10px; padding: 20px 24px;
               border: 1px solid #333; margin-bottom: 24px; }}
  .insights ul {{ list-style: none; display: flex; flex-direction: column; gap: 8px; }}
  .insights li {{ padding: 6px 0; border-bottom: 1px solid #333; font-size: 14px; }}
  .insights li:last-child {{ border-bottom: none; }}
  table.stats-table {{ width: 100%; border-collapse: collapse; margin: 12px 0;
                       background: {SURFACE}; border-radius: 8px; overflow: hidden; }}
  table.stats-table th {{ background: #333; color: #fff; padding: 10px 14px;
                          text-align: left; font-size: 12px; text-transform: uppercase;
                          letter-spacing: 0.5px; }}
  table.stats-table td {{ padding: 9px 14px; border-bottom: 1px solid #333;
                          font-size: 13px; color: {TEXT}; }}
  table.stats-table tr:hover td {{ background: #333; }}
  table.stats-table tr:last-child td {{ border-bottom: none; }}
  .section {{ margin-bottom: 32px; }}
  footer {{ text-align: center; color: #555; font-size: 12px;
            padding: 24px; border-top: 1px solid #333; margin-top: 40px; }}
</style>
</head>
<body>
<div class="hero">
  <h1>📊 {TITLE}</h1>
  <div class="meta">Generated on {timestamp}</div>
  <div class="badges">
    <span class="badge">🗄️ {DATABASE}</span>
    <span class="badge">📋 {len(df):,} rows</span>
    <span class="badge">🔢 {len(df.columns)} columns</span>
    <span class="badge">📐 {len(numeric_cols)} numeric</span>
    {"<span class='badge'>📅 " + str(len(datetime_cols)) + " datetime</span>" if datetime_cols else ""}
  </div>
</div>

<div class="container">

  <h2>💡 Key Insights</h2>
  <div class="insights">
    <ul>{insights_html}</ul>
  </div>

  <h2>📋 Statistical Summary</h2>
  <div class="section">{desc_html}</div>

  <h2>🔍 Column Overview</h2>
  <div class="section">{null_html}</div>

  <h2>👀 Sample Data (Top 5 rows)</h2>
  <div class="section">{sample_html}</div>

  {charts_html}

</div>
<footer>SQL Data Analyst · {DATABASE} · {timestamp}</footer>
</body>
</html>"""

with open(OUTPUT, 'w', encoding='utf-8') as f:
    f.write(html)

print(f"\n✅ Report saved to: {OUTPUT}")
print("\n── Quick summary ─────────────────────────────────────")
for insight in insights:
    import re
    clean = re.sub('<[^>]+>', '', insight)
    print(f"  {clean}")
print("──────────────────────────────────────────────────────")
print("Open the HTML file in Chrome or Edge to view the report.")
```

## Notes for Claude when filling the template

- If user gives a **view name**: `QUERY = "SELECT * FROM dbo.{view_name}"`
- If user gives a **table name**: `QUERY = "SELECT * FROM dbo.{table_name}"`
- If user gives a **custom query**: use it directly as `QUERY`
- If the table has >500k rows, automatically add `TOP 500000`
- TITLE should be user-friendly, e.g. `"IAM Metrics Analysis"` not `"vw_iam_metrics"`
- Always use `matplotlib.use('Agg')` before pyplot import — required for headless runs
- The ODBC Driver version: try `17` first, if it fails try `13` or `SQL Server`
