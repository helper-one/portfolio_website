# One-Time Environment Setup

When the user runs this skill for the first time, write this file to their home directory
and execute it. Path: `C:/Users/<USERNAME>/sql_analyst_env_setup.py`

Get the username first:
```bash
echo %USERNAME%
```

Then write the setup script:

## Setup script to write

```python
# sql_analyst_env_setup.py
# Run once to create the SQL Analyst virtual environment

import subprocess
import sys
import os

ENV_PATH = os.path.expanduser("~/sql_analyst_env")
PACKAGES = [
    "pandas",
    "pyodbc",
    "matplotlib",
    "seaborn",
    "openpyxl",   # for Excel export if needed later
    "tabulate",   # for pretty terminal tables
]

print("Creating virtual environment...")
subprocess.run([sys.executable, "-m", "venv", ENV_PATH], check=True)

pip_path = os.path.join(ENV_PATH, "Scripts", "pip.exe")

print("Installing packages...")
subprocess.run([pip_path, "install", "--quiet"] + PACKAGES, check=True)

print("\n✅ Setup complete!")
print(f"Environment at: {ENV_PATH}")
print("\nVerifying install...")
python_path = os.path.join(ENV_PATH, "Scripts", "python.exe")
subprocess.run([python_path, "-c",
    "import pandas, pyodbc, matplotlib, seaborn; print('All packages OK')"],
    check=True
)
```

## How to run setup (Claude executes this)

```bash
# Write the file (Claude writes it via create_file or echo)
# Then run:
python C:/Users/%USERNAME%/sql_analyst_env_setup.py
```

## After setup, verify with:

```bash
~/sql_analyst_env/Scripts/python.exe -c "import pandas, pyodbc, matplotlib, seaborn; print('Environment ready')"
```

If verification passes, setup is complete. Never run setup again unless the user
switches machines or recreates their Miniconda base.
