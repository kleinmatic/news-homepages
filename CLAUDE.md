# Claude Code Context: News Homepages Project

## Project Overview

**news-homepages** is an open-source archive that gathers, saves, shares, and analyzes news homepages. It's a Python package maintained by Ben Welsh.

- **Repository:** https://github.com/palewire/news-homepages
- **Archive:** https://archive.org/details/news-homepages
- **Documentation:** https://palewi.re/docs/news-homepages/
- **Site list:** `newshomepages/sources/sites.csv` (1000+ news sites)

## Core Architecture

The codebase follows a consistent pattern:

### 1. Collection Commands (Local)
Scripts that fetch/capture data and save locally. **Do NOT upload automatically.**

- `newshomepages/screenshot.py` - Takes screenshots with Playwright
- `newshomepages/robotstxt.py` - Fetches robots.txt files
- `newshomepages/adstxt.py` - Fetches ads.txt files
- `newshomepages/accessibility.py` - Captures accessibility data
- `newshomepages/hyperlinks.py` - Extracts hyperlinks

**Pattern:**
- CLI: `newshomepages-<name> <handle>`
- Options: `-o/--output-dir`, `--timeout`, `--verbose`
- Output: `<handle>.<type>.<extension>`
- Entry point: `cli()` function with `@click.command()`

**Output conventions (see .gitignore):**
- Default output dir: `./` (current directory) - specified with `-o` flag
- Common local organization: `_screenshots/`, `_hyperlinks/`, etc. at project root
- Site-related outputs: `_site/blocking/`, `_site/llmstxt/`, etc.
- These underscore directories are in `.gitignore` - outputs are not committed
- Generated site content goes in `_site/` (e.g., `_site/openai-gptbot-robotstxt.md`)

### 2. Archive Command (Upload)
Separate command that uploads files to Internet Archive.

- `newshomepages/archive.py`
- Requires env vars: `IA_ACCESS_KEY`, `IA_SECRET_KEY`, `IA_COLLECTION`
- Will fail if credentials not set (safety mechanism)
- Uploads timestamped files: `<handle>-<timestamp>.<extension>`

### 3. Extract Commands (Analysis)
Download archived data and analyze it.

- `newshomepages/extract/robotstxt.py`
- `newshomepages/extract/accessibility.py`
- `newshomepages/extract/hyperlinks.py`

**Pattern:**
- Support filtering: `--site`, `--country`, `--language`, `--bundle`
- Support time ranges: `--latest`, `--monthly`, `--days`
- Cache downloads to `~/.cache/news-homepages/`
- Output CSV files with analysis

### 4. Site Generation
Generate static website pages from extracts.

- `newshomepages/site/openai.py` - Tracks AI bot blocking
- `newshomepages/site/performance_ranking.py` - Lighthouse scores
- `newshomepages/site/accessibility_ranking.py` - Accessibility scores

## Key Files & Utilities

### Data Sources
- `newshomepages/sources/sites.csv` - Master list of 1000+ news sites
- `newshomepages/sources/bundles.csv` - Site groupings (e.g., "georgia", "texas", "tech")
- `newshomepages/sources/javascript/` - Custom JS for specific sites to remove overlays

### Utils (newshomepages/utils.py)
- `get_site(handle)` - Get site metadata by handle
- `get_site_list()` - Get all sites
- `get_user_agent()` - Returns random browser UA (lines 268-279)
- `get_url(url, user_agent=None)` - Fetch URL with optional custom UA (lines 118-148)
- `_load_persistent_context()` - Playwright browser with ad-blocking extensions
- `_get_common_blocking_javascript()` - JS to remove popups/modals/GDPR notices

### How to Run Commands

Using pipenv (recommended pattern from docs):
```bash
pipenv run python -m newshomepages.robotstxt latimes
pipenv run python -m newshomepages.screenshot nytimes
pipenv run python -m newshomepages.archive latimes -i ./output/
```

Or after installing package:
```bash
pipenv install -e .
pipenv run newshomepages-robotstxt latimes
```

## Current Work: AI Bot Blocking Detection

### Background
The project currently tracks which news sites block AI scrapers by:
1. Fetching robots.txt files
2. Parsing for AI bot user agents (GPTBot, CCBot, Google-Extended)
3. Checking if they have `Disallow: /` rules
4. Generating a report in `site/openai.py`

**Current limitation:** Only checks what robots.txt *declares*, not whether sites actually *enforce* blocking via HTTP status codes.

### AI Bot User Agents
Currently hardcoded in `newshomepages/site/openai.py` lines 32-38:
- GPTBOT (OpenAI)
- CCBOT (Common Crawl)
- GOOGLE-EXTENDED (Google AI)
- CHATGPT-USER (commented out)
- CLAUDE-WEB (commented out)

### Planned Enhancements (See TODO.md)

#### Task 1: Move AI scrapers to CSV ✅ Plan complete
- Create `newshomepages/sources/aiscrapers.csv`
- Columns: `user_agent`, `name`, `active`
- Update `site/openai.py` to read from CSV instead of hardcoded list
- Filter to `active == true` rows only

#### Task 2: Test actual blocking ✅ Plan complete
- **New file:** `newshomepages/blocking.py`
- Test sites with AI bot user agents (e.g., "GPTBot")
- Report actual HTTP response codes
- 200 = not blocking, 4xx = blocking, 3xx = follow redirect, 5xx = down
- Output: `<handle>.blocking-test.json`

#### Task 3: Check for llms.txt ✅ Plan complete
- **New file:** `newshomepages/llmstxt.py`
- Fetch `/llms.txt` from sites
- Mirror pattern from `robotstxt.py` exactly
- Output: `<handle>.llms.txt`

#### Task 4 & 5: RSL License + Content Signals ✅ Plan complete
- **Modify:** `newshomepages/extract/robotstxt.py`
- Parse `License:` lines (RSL - Really Simple Licensing)
- Parse `Content-Signal:` lines (Cloudflare content signals)
- Add as new columns to robotstxt extract CSV output
- Reference: https://rslcollective.org/robots.txt

## Important Patterns to Follow

1. **Separate collection from archiving** - Never auto-upload
2. **Mirror existing code structure** - Copy similar commands (robotstxt.py for llmstxt.py)
3. **Use consistent CLI options** - `-o/--output-dir`, `--timeout`, `--verbose`
4. **Handle 404s gracefully** - Write "404: No file found" to output file
5. **Use retry decorators** - `@retry(tries=3, delay=15, backoff=2)`
6. **Cache downloads** - Extract commands use `~/.cache/news-homepages/`
7. **Denormalized CSV output** - It's okay to duplicate metadata across rows

## Development Environment

### Standard Setup

```bash
# Install dependencies
pipenv install --dev

# Install pre-commit hooks
pipenv run pre-commit install

# Install Playwright browsers
pipenv run playwright install --with-deps chromium

# Run tests
pipenv run pytest

# Lint
pipenv run flake8

# Type check
pipenv run mypy .
```

### Setup Issues (Potentially Ephemeral)

**Note:** These issues were encountered during development on 2025-12-05 and may not affect all users or environments.

**Python version requirement:**
- Project requires Python 3.10 specifically (see `Pipfile` line 61)
- If you don't have it: `brew install python@3.10`
- Then run `pipenv install --dev` (pipenv will detect and use 3.10)

**libtorrent dependency issue:**
- On some systems (macOS), `libtorrent` may fail to install during `pipenv install`
- **Workaround:** Comment out the `libtorrent` line in `Pipfile.lock`
- Note: Listed in `setup.py` line 103 but commented out in `Pipfile` line 27
- The project appears to work without it for basic operations (screenshot, robotstxt, etc.)

## Archive Upload Safety

The archive command is safe to run without credentials:
- Requires: `IA_ACCESS_KEY`, `IA_SECRET_KEY`, `IA_COLLECTION`
- Asserts these exist (lines 46-48 in archive.py)
- Will fail immediately if not set
- No accidental uploads possible

## Status

Working on implementing Tasks 1-5 from TODO.md. All tasks have detailed implementation plans. Ready to begin coding.
