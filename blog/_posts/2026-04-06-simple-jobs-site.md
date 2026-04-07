---
layout: post
title:  "Job Listings App with SerpApi"
date:   2026-04-06
categories:
    - python
    - SerpApi
    - flask
    - postgres
toc: true
---

# Building a Job Listings App with SerpApi, Flask, and Postgres

[Demo](https://hamcclellan.pythonanywhere.com/) | [Repo](https://github.com/harchermcclellan/job-finder)

A simple Flask app that wraps SerpApi's Google Jobs engine, caches results in Postgres, and supports multi-title search via a tag input UI. The motivation: I'm job searching across multiple roles at once (full-time dev rel, contract-based e-commerce management, part-time retail styling for fun) and wanted a single feed instead of running three separate searches.

Looking for a little more explanation? See the beginner-friendly guide [here](/blog/posts/serp-api-for-beginners)

## Stack

- **Flask** — HTTP layer
- **SerpApi** — scrapes Google Jobs results via a clean Python client
- **psycopg2** — Postgres adapter (I used Supabase for hosting the DB)
- **PythonAnywhere** — app hosting

## Setup

```bash
mkdir job-finder && cd job-finder
python3.12 -m venv .venv && source .venv/bin/activate
pip install flask python-dotenv serpapi psycopg2
```

Export your credentials in `~/.bashrc`:
```bash
export SERP_API_KEY="..."
export DATABASE_URL="..."
```

## App Structure

```
job-finder/
├── app.py        # Flask routes + business logic
├── db.py         # Cache layer
└── templates/
    └── index.html
```

## Backend

`app.py` has three concerns: serving the SPA, handling search POSTs, and delegating to the cache.

```python
app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/search", methods=["POST"])
def search():
    data = request.get_json()
    titles = data.get("titles", [])
    location = data.get("location", "").strip()
    work_type = data.get("work_type", "any")
    salary = data.get("salary", "").strip()

    if not titles:
        return jsonify({"error": "Please enter at least one job title."}), 400

    all_jobs = []
    for title in titles:
        cache_key = make_key(title, location, work_type, salary)
        cached = get_cached(cache_key)
        try:
            if cached:
                all_jobs.extend(cached)
            else:
                jobs = search_jobs(title, location, work_type, salary)
                set_cached(cache_key, title, location, jobs)
                all_jobs.extend(jobs)
        except Exception as e:
            print(f"Error searching for '{title}': {e}")
            traceback.print_exc()

    return jsonify({"jobs": all_jobs})
```

The SerpApi call itself is straightforward — the interesting part is pulling from `detected_extensions`, which is where SerpApi normalizes structured fields like salary, schedule type, and benefits out of the raw listing text:

```python
def search_jobs(title: str, location: str, work_type: str, salary: str) -> list[dict]:
    client = serpapi.Client(api_key=API_KEY)
    results = client.search({
        "engine": "google_jobs",
        "q": title,
        "location": location,
        "google_domain": "google.com",
        "hl": "en", "gl": "us",
    })

    response = []
    for result in results["jobs_results"]:
        extended = get_extended_details(result, ["salary", "schedule_type", "posted_at", "health_insurance"])
        response.append({
            "title": result.get("title", ""),
            "company": result.get("company_name", ""),
            "location": result.get("location", ""),
            "work_type": extended.get("schedule_type", ""),
            "url": result.get("source_link", ""),
            "salary": extended.get("salary", ""),
            "posted": extended.get("posted_at", ""),
        })
    return response

def get_extended_details(result, details_list):
    return {
        field: result.get("detected_extensions", {}).get(field, "See full listing")
        for field in details_list
    }
```

## Caching Layer (`db.py`)

To stay within SerpApi's monthly quota, results are cached in Postgres with a 24-hour TTL. The cache key is a SHA-256 hash of the search parameters:

```python
def make_key(description, location, work_type, salary) -> str:
    raw = f"{description}|{location}|{work_type}|{salary}".lower().strip()
    return hashlib.sha256(raw.encode()).hexdigest()

def get_cached(cache_key: str) -> list | None:
    cutoff = datetime.now(timezone.utc) - timedelta(hours=24)
    with get_conn() as conn, conn.cursor() as cur:
        cur.execute("""
            SELECT results FROM search_cache
            WHERE cache_key = %s AND created_at > %s
        """, (cache_key, cutoff))
        row = cur.fetchone()
        return row[0] if row else None

def set_cached(cache_key, role, location, results):
    with get_conn() as conn, conn.cursor() as cur:
        cur.execute("""
            INSERT INTO search_cache (cache_key, role, location, results, created_at)
            VALUES (%s, %s, %s, %s, NOW())
            ON CONFLICT (cache_key)
            DO UPDATE SET results = EXCLUDED.results, created_at = NOW()
        """, (cache_key, role, location, json.dumps(results)))
```

The upsert on conflict handles re-searching an existing query — it refreshes the cache rather than inserting a duplicate.

## Frontend Notes

I used Claude to generate the frontend (no shame — I just don't enjoy writing it). The main things worth calling out:

**Tag input for multi-title search** — comma or Enter adds a search term as a tag, matching the Google Flights multi-destination UX. On submit, `titles` is sent as an array in the POST body.

**Loading overlay** — since search + cache miss takes a noticeable moment, there's a full-screen overlay with a spinning ☀️ emoji:

```javascript
function setLoading(on) {
    document.getElementById('searchBtn').disabled = on;
    document.getElementById('loadingOverlay').classList.toggle('visible', on);
    if (on) {
        document.getElementById('jobList').innerHTML = '<div class="list-state">Searching...</div>';
        document.getElementById('resultCount').textContent = '';
    }
}
```

**Day/Night mode toggle** — default styling was too dark for me, so there's a toggle that switches CSS custom properties.

## Deployment

Hosted on PythonAnywhere. For the public demo, both `SERP_API_KEY` and `DATABASE_URL` env vars are absent, so the app falls back to returning static sample data — the `search()` and `search_jobs()` functions check for `None` before attempting live calls.

## What's Missing / Next Steps

- **Filter by work type and schedule** — SerpApi's `q` parameter doesn't natively support role type (remote/hybrid) or schedule (FT/PT). Would need to post-filter on the returned `detected_extensions`.
- **Multi-location search** — e.g., "internship in NYC or Boston or Chicago." Natural extension of the multi-title pattern.
- **Deduplication** — multi-title searches produce overlapping results. The `job_id` field in SerpApi responses works as a stable unique key for deduping.
- **Night mode spinner** — swap ☀️ for 🌙 when night mode is active. Small thing, but it's been bugging me.
- **Migrate cache to Redis** — Postgres works fine at low traffic, but it's not really what it's designed for. Redis handles TTL natively, stores everything in memory, and would meaningfully cut cache read latency. The `db.py` cache layer is already cleanly separated, so swapping the backing store would be a contained change — replace `get_cached` and `set_cached` with `redis-py` calls and let Redis handle expiry instead of the manual `created_at > cutoff` query.