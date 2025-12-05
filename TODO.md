# TODO

## 1. Move AI scrapers list to CSV

**Goal:** Change the hardcoded AI crawler list in `newshomepages/site/openai.py` (lines 32-38) to read from `newshomepages/sources/aiscrapers.csv`

**CSV Structure (to be created by user):**
```csv
user_agent,name,active
GPTBOT,OpenAI GPTBot,true
CHATGPT-USER,OpenAI ChatGPT Browse,false
CCBOT,Common Crawl,true
GOOGLE-EXTENDED,Google Extended (AI),true
CLAUDE-WEB,Anthropic Claude,false
```

**Implementation:**
- Replace hardcoded list with CSV read (pattern: `get_bundle_list()` in utils.py:349-357)
- Filter to only `active == true` rows
- Update `ai_user_agents` to pull from CSV instead of hardcoded list

**File to modify:** `newshomepages/site/openai.py`

---

## 2. Test user agent blocking

**Goal:** Create a new script that uses a known AI bot user agent and reports the HTTP response code

**Response code interpretation:**
- `200` = NOT blocking (negative result)
- `4xx` = Blocking via user agent (positive result)
- `3xx` = Redirect (follow it)
- `5xx` = Site is down

**Implementation:**
- **New file:** `newshomepages/blocking.py` (mirrors `robotstxt.py` and `adstxt.py`)
- **CLI command:** `newshomepages-blocking <handle>`
- **Options:** `-o/--output-dir` (default: `./`), `--timeout`, `--verbose`
- **Test UA:** Constant `TEST_USER_AGENT = "GPTBot"` at top of file
- **Output file:** `<handle>.blocking-test.json` (suggest using `-o _site/blocking/`)
- **Output structure:**
  ```json
  {
    "url": "https://example.com/",
    "user_agent": "GPTBot",
    "status_code": 403,
    "blocked": true,
    "timestamp": "2025-12-05T..."
  }
  ```
- **Logic:** `blocked = (400 <= status_code < 500)`
- Use `requests.get()` with `allow_redirects=True` to follow 3xx redirects
- Add console script to `setup.py`: `"newshomepages-blocking=newshomepages.blocking:cli"`
- Add `_blocking` to `.gitignore` for local output organization

---

## 3. Check for llms.txt

**Goal:** Look for the existence of a file called `llms.txt` at the root of each site

**Implementation:**
- **New file:** `newshomepages/llmstxt.py` (exactly mirrors `robotstxt.py`)
- **CLI command:** `newshomepages-llmstxt <handle>`
- **Options:** `-o/--output-dir` (default: `./`), `--timeout`, `--verbose`
- **Fetch from:** `https://example.com/llms.txt`
- **Output file:** `<handle>.llms.txt` (suggest using `-o _site/llmstxt/`)
- **Handle 404s:** Same as `robotstxt.py` - write "404: No file found"
- Copy `robotstxt.py` and change:
  - Function name: `_get_llmstxt()`
  - URL path: `/llms.txt`
  - Error message: "No llms.txt for {handle}"
- Add console script to `setup.py`: `"newshomepages-llmstxt=newshomepages.llmstxt:cli"`

---

## 4. RSL (Really Simple Licensing)

**Goal:** Look for a line in robots.txt that starts with "License:" and extract the URL

**Example:** https://rslcollective.org/robots.txt

**Implementation:**
- **File to modify:** `newshomepages/extract/robotstxt.py`
- Add parsing function after line 135:
  ```python
  def _parse_metadata(robotstxt: str) -> dict:
      """Extract RSL License and Content Signal from robots.txt."""
      rsl_license = None
      content_signal = None

      for line in robotstxt.split("\n"):
          line = line.strip()
          if line.startswith("License:"):
              rsl_license = line.split(":", 1)[1].strip()
          elif line.startswith("Content-Signal:"):
              content_signal = line.split(":", 1)[1].strip()

      return {"rsl_license": rsl_license, "content_signal": content_signal}
  ```
- Apply to dataframe after line 137:
  ```python
  metadata_df = qualified_df["robotstxt"].apply(_parse_metadata).apply(pd.Series)
  qualified_df = pd.concat([qualified_df, metadata_df], axis=1)
  ```
- Update SQL export (line 157) to include new columns: `rsl_license`, `content_signal`
- Update final SELECT query (line 183) to include them in output
- **Result:** CSV output will have new columns: `rsl_license`, `content_signal`

---

## 5. Content Signals Policy

**Goal:** Look for a line in robots.txt that starts with "Content-Signal:" and extract the text

**Reference:** https://blog.cloudflare.com/robots.txt

**Implementation:**
- Same as Task 4 above - implemented in the same `_parse_metadata()` function
- Extracts both RSL License and Content Signal together
- Both will appear as new columns in the robotstxt extract CSV output
