Build a production-ready GitLab Code Review Agent from scratch. 
This is a complete end-to-end project. Build every file completely 
with no placeholders, no TODOs, and no truncation.

---

PROJECT OVERVIEW:
An AI-powered code review agent that:
- Automatically triggers when a GitLab Merge Request is opened or updated
- Fetches the MR diff using GitLab REST API
- Sends the diff to Claude Code CLI (claude -p) for review
- Posts inline comments on the exact changed lines in the MR
- Posts a summary comment at the MR level with overall score
- Generates an HTML report of the review
- Has an MCP server with GitLab tools that Claude uses internally

---

TECH STACK:
- Python 3.11+
- FastAPI + Uvicorn (webhook server)
- httpx (async HTTP client for GitLab API)
- Claude Code CLI via subprocess (claude --bare -p)
- MCP server (mcp Python package) for GitLab tools
- Pydantic v2 + pydantic-settings (models and config)
- Jinja2 (HTML report generation)
- python-dotenv (.env loading)
- pytest + pytest-asyncio (tests)
- SQLite NOT needed - skip it entirely

---

FOLDER STRUCTURE TO CREATE:
code-review-agent/
├── main.py
├── requirements.txt
├── .env.example
├── .gitignore
├── CLAUDE.md
├── README.md
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── api/
│   │   ├── __init__.py
│   │   └── webhook.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── gitlab_service.py
│   │   ├── claude_runner.py
│   │   ├── review_orchestrator.py
│   │   └── report_generator.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── webhook_payload.py
│   │   └── review_result.py
│   └── utils/
│       ├── __init__.py
│       ├── diff_parser.py
│       └── logger.py
├── mcp/
│   ├── __init__.py
│   ├── server.py
│   ├── gitlab_tools.py
│   └── mcp_config.json
├── templates/
│   └── review_report.html
├── reports/
├── tests/
│   ├── __init__.py
│   ├── test_webhook.py
│   ├── test_gitlab_service.py
│   ├── test_claude_runner.py
│   └── test_diff_parser.py
└── scripts/
    ├── setup.sh
    └── test_webhook.sh

---

BUILD EACH FILE WITH THESE EXACT SPECS:

### main.py
- FastAPI app initialization
- Include webhook router at prefix /api/v1
- GET /health endpoint returning status ok
- Uvicorn runner with settings from config

### requirements.txt
fastapi==0.111.0
uvicorn==0.29.0
httpx==0.27.0
python-dotenv==1.0.1
pydantic==2.7.1
pydantic-settings==2.2.1
jinja2==3.1.4
pytest==8.2.0
pytest-asyncio==0.23.6
mcp==1.0.0

### .env.example
GITLAB_URL=https://gitlab.yourcompany.com
GITLAB_PRIVATE_TOKEN=your_gitlab_pat_token_here
GITLAB_WEBHOOK_SECRET=your_webhook_secret_here
APP_HOST=0.0.0.0
APP_PORT=8000
LOG_LEVEL=INFO
REPORTS_DIR=reports

### app/config.py
- Use pydantic-settings BaseSettings
- Load from .env file
- Fields: gitlab_url, gitlab_private_token, 
  gitlab_webhook_secret, app_host, app_port, 
  log_level, reports_dir
- Use @lru_cache for singleton pattern

### app/utils/logger.py
- StreamHandler to stdout
- Format: timestamp | level | name | message
- Use settings log_level

### app/models/webhook_payload.py
- Pydantic model for GitLab MR webhook payload
- Fields: object_kind, event_type, project 
  (id, name, web_url), object_attributes 
  (iid, title, state, action, url, 
  source_branch, target_branch, last_commit)
- Only process action = "open" or "update"

### app/models/review_result.py
- Pydantic models for Claude's review output
- ReviewIssue: file_path, line_number, severity 
  (critical/warning/info), message, suggestion
- FileReview: file_path, issues, file_score, summary
- FullReview: mr_id, mr_title, files, overall_score, 
  summary, total_issues, critical_count, 
  warning_count, info_count

### app/services/gitlab_service.py
- Async class GitLabService using httpx.AsyncClient
- Methods:
  get_mr_diff(project_id, mr_iid) -> list of diffs
  get_mr_details(project_id, mr_iid) -> MR info
  post_inline_comment(project_id, mr_iid, 
    file_path, line, message) -> None
  post_summary_comment(project_id, mr_iid, 
    markdown_body) -> None
- Use PRIVATE-TOKEN header for auth
- Handle errors with proper logging
- Base URL from settings.gitlab_url

### app/utils/diff_parser.py
- parse_diff(raw_diff_list) -> list of parsed file diffs
- Each parsed diff has: file_path, change_type 
  (added/modified/deleted), hunks, changed_lines
- Extract only added/changed lines for review
- Handle binary files gracefully (skip them)
- chunk_diff_for_review(parsed_diffs, max_chars=8000) 
  -> list of chunks so we dont exceed claude context

### app/services/claude_runner.py
- class ClaudeRunner
- Method: async run_review(diff_chunk, mr_context) 
  -> FullReview or FileReview
- Use subprocess to call:
  claude --bare -p "{prompt}" --output-format json 
  --allowedTools "Read" --max-turns 3
- Build a detailed prompt that instructs Claude to:
  * Review for security issues (SQL injection, 
    hardcoded secrets, XSS, auth bypass)
  * Review for code quality and best practices
  * Check error handling
  * Check performance issues
  * Suggest missing tests
  * Return ONLY valid JSON matching ReviewResult schema
- Parse JSON output from Claude
- Handle subprocess errors and timeouts (60s timeout)
- Retry once on failure

### app/services/review_orchestrator.py
- class ReviewOrchestrator
- Main method: async orchestrate(payload) -> FullReview
- Flow:
  1. Extract project_id and mr_iid from payload
  2. Fetch MR diff via GitLabService
  3. Parse diff via diff_parser
  4. Chunk diff for Claude context limits
  5. Run ClaudeRunner on each chunk
  6. Merge all chunk results into FullReview
  7. Post inline comments via GitLabService
  8. Post summary comment with score and issue counts
  9. Generate HTML report via ReportGenerator
  10. Return FullReview

### app/api/webhook.py
- FastAPI router
- POST /webhook endpoint
- Validate X-Gitlab-Token header against 
  settings.gitlab_webhook_secret
- Return 401 if token invalid
- Only process merge_request events
- Only process action open or update
- Call ReviewOrchestrator in background 
  (dont make GitLab wait)
- Return 200 immediately to GitLab
- Log all incoming events

### app/services/report_generator.py
- class ReportGenerator
- Method: generate(review: FullReview) -> str (file path)
- Use Jinja2 to render review_report.html template
- Save report to reports/ folder as 
  review_{mr_id}_{timestamp}.html
- Return the saved file path

### templates/review_report.html
- Clean professional HTML report
- Show MR title, overall score with color coding
  (red < 5, orange 5-7, green > 7)
- Summary section
- Per file breakdown with issues table
- Severity badges (red=critical, yellow=warning, 
  blue=info)
- Issue count summary at top
- Responsive design using inline CSS only 
  (no external dependencies)

### mcp/gitlab_tools.py
- MCP tools for GitLab operations
- Tool 1: get_mr_diff(project_id, mr_iid)
- Tool 2: post_comment(project_id, mr_iid, comment)
- Tool 3: get_mr_files(project_id, mr_iid)
- Use httpx for GitLab API calls
- Use settings for auth

### mcp/server.py
- MCP server setup using mcp Python package
- Register all tools from gitlab_tools.py
- Stdio transport

### mcp/mcp_config.json
- MCP config file for claude --bare -p to use
- Points to mcp/server.py

### CLAUDE.md
- Instructions for Claude when reviewing code
- Review checklist:
  * Security: secrets, injection, auth, encryption
  * Quality: naming, complexity, duplication
  * Error handling: try/catch, edge cases
  * Performance: N+1 queries, memory leaks
  * Testing: missing test suggestions
- Output format instructions (strict JSON only)
- Severity definitions:
  critical = security risk or breaking bug
  warning = bad practice or potential bug  
  info = suggestion or improvement
- Scoring guide: 1-10 where 10 is perfect code

### tests/test_webhook.py
- Test valid webhook with correct token
- Test invalid token returns 401
- Test non MR event is ignored
- Test MR action closed is ignored
- Use FastAPI TestClient

### tests/test_diff_parser.py
- Test parsing a sample git diff
- Test binary file is skipped
- Test chunking splits large diffs correctly

### tests/test_claude_runner.py
- Mock subprocess call
- Test valid JSON response is parsed correctly
- Test timeout is handled gracefully
- Test retry on failure works

### tests/test_gitlab_service.py
- Mock httpx calls
- Test get_mr_diff returns correct structure
- Test post_inline_comment calls correct endpoint
- Test post_summary_comment calls correct endpoint

### scripts/setup.sh
- Create venv
- Install requirements
- Create reports/ dir
- Copy .env.example to .env
- Print next steps

### scripts/test_webhook.sh
- curl command to send a fake MR webhook 
  to localhost:8000/api/v1/webhook
- Include sample GitLab MR payload
- Include correct X-Gitlab-Token header

### README.md
- Project overview
- Prerequisites
- Setup instructions
- How to configure GitLab webhook
- How to get GitLab PAT token
- How to run locally
- How to deploy on OpenShift
- Environment variables table

---

IMPORTANT RULES FOR BUILDING:
1. Every file must be 100% complete - no placeholders
2. No TODO comments anywhere
3. All imports must be correct and working
4. Error handling in every service method
5. Logging in every important step
6. Type hints on every function
7. Async/await used correctly throughout
8. The agent must work end to end when setup.sh is run

---

Start building now. Create every file one by one 
completely. Do not summarize or skip any file.
