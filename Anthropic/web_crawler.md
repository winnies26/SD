# 🕸️ Web Crawler (Full Breakdown)

---

## 📌 Problem Prompt

> **"Design and implement a web crawler that starts from a given seed URL and crawls all reachable URLs within the same domain."**
>
> You’ll be provided a helper function like:

```python
def get_urls(url: str) -> List[str]:
    """Returns a list of links (strings) found on the given URL page."""
```

> You’ll be asked to:

* Write a **single-threaded** crawler first.
* Then make it **multi-threaded or async**.
* Then answer **production system design** follow-ups.

---

## 🧠 Constraints & Clarifications

* **Only crawl URLs in the same domain** as the seed URL.
* **Ignore fragments** (`#section`).
* You don’t need to handle normalization like `http:// vs https://`.
* Replit or an online environment will be used — be ready to print logs.
* URLs may appear multiple times — don’t revisit.
* They may provide a **wrapper around jsoup** (in Java) or equivalent.

---

## ✅ Step-by-Step Implementation

---

### 🔹 Step 1 – Basic Single-threaded Crawler (DFS)

```python
from urllib.parse import urlparse
from typing import Set, List

def web_crawler(start_url: str, get_urls) -> Set[str]:
    seen = set()
    domain = urlparse(start_url).netloc

    def dfs(url):
        if url in seen:
            return
        if urlparse(url).netloc != domain:
            return
        seen.add(url)
        for next_url in get_urls(url):
            if '#' in next_url:  # filter fragment
                next_url = next_url.split('#')[0]
            dfs(next_url)

    dfs(start_url)
    return seen
```

---

### 🔹 Step 2 – Convert to Multi-threaded Crawler (ThreadPool)

```python
import threading
from concurrent.futures import ThreadPoolExecutor
from urllib.parse import urlparse
from typing import Set

def web_crawler_multithreaded(start_url: str, get_urls) -> Set[str]:
    seen = set()
    seen_lock = threading.Lock()
    domain = urlparse(start_url).netloc
    executor = ThreadPoolExecutor(max_workers=10)

    def crawl(url):
        with seen_lock:
            if url in seen:
                return
            seen.add(url)
        if urlparse(url).netloc != domain:
            return

        for next_url in get_urls(url):
            if '#' in next_url:
                next_url = next_url.split('#')[0]
            executor.submit(crawl, next_url)

    crawl(start_url)
    executor.shutdown(wait=True)
    return seen
```

---

## 🧪 Real Interview Follow-Up Questions + Answers

---

### 🔍 Q1: **How would you implement a politeness policy (rate-limiting)?**

> “Our current crawler is too aggressive.”

#### ✅ Answer:

* Use a **per-domain rate limiter**:

  * Sleep `X` milliseconds between requests.
  * Use a `domain → last_request_time` dictionary.
* Better: use a **token bucket** or `asyncio.Semaphore`.

Example (simplified):

```python
import time

last_fetch = {}
def polite_fetch(url):
    domain = urlparse(url).netloc
    now = time.time()
    if domain in last_fetch and now - last_fetch[domain] < 1:
        time.sleep(1 - (now - last_fetch[domain]))
    last_fetch[domain] = time.time()
    return get_urls(url)
```

---

### 🔍 Q2: **How would you scale this to millions of URLs across many machines?**

> “单机扛不住了怎么办？”

#### ✅ Answer:

* Use **distributed crawling architecture**:

  * Frontend or CLI submits seed URLs.
  * Store unvisited URLs in a shared queue (e.g. Redis, Kafka).
  * Each worker:

    * Pulls URL from queue.
    * Crawls and extracts new URLs.
    * Pushes new URLs back into the queue if unseen.
  * Use **Bloom filters** or sharded databases for deduplication.

---

### 🔍 Q3: **How would you detect and avoid duplicate/similar content?**

> “很多不同 URL 对应相同内容，怎么优化？”

#### ✅ Answer:

* Detect duplicate content via:

  * **Hashing the HTML** (e.g. SHA-256 of content).
  * **Shingling + MinHash** to detect near-duplicates.
  * **SimHash** for structural similarity.
* Store and compare hashes in a cache/DB.

---

### 🔍 Q4: **What's the difference between thread and process? When to use each?**

> “thread vs process 区别和使用场景”

| Aspect        | Thread                     | Process                           |
| ------------- | -------------------------- | --------------------------------- |
| Memory        | Shared                     | Isolated                          |
| Overhead      | Low                        | High                              |
| Best for      | I/O-bound tasks            | CPU-bound tasks                   |
| Python limits | Blocked by GIL (use async) | Not blocked (via multiprocessing) |

#### ✅ Use:

* **ThreadPoolExecutor** for I/O (like web scraping).
* **ProcessPoolExecutor** for CPU (like parsing, ML).

---

### 🔍 Q5: **How would you benchmark and optimize this?**

> “还需要你 benchmark 下 performance”

#### ✅ Answer:

* Log **crawling rate**: URLs per second.
* Measure time taken per URL.
* Profile using:

  * `time.perf_counter()`
  * `concurrent.futures.as_completed()`
* Optimize:

  * Thread pool size
  * Async I/O (e.g. `aiohttp` + `asyncio`)
  * URL deduplication speed
  * Use a real profiler (`cProfile`, `pyinstrument`)

---

### 🔍 Q6: **What if the helper blocks the event loop (async fails)?**

> “coroutine 是 lightweight 但要看 helper 是不是 blocking”

#### ✅ Answer:

* `async def` works only if all I/O is **non-blocking**.
* If `get_urls()` uses `requests` or blocking HTTP:

  * Async won’t help — it **blocks the whole loop**.
* Use `aiohttp` or offload blocking calls:

```python
loop.run_in_executor(None, blocking_get_urls, url)
```

---

### 🔍 Q7: **What if many URLs have `#` fragments?**

> “最后让我 filter 掉 # 的 postfix”

#### ✅ Answer:

* Remove fragments using `split('#')[0]`
* Or use `urllib.parse.urldefrag()`:

```python
from urllib.parse import urldefrag
clean_url = urldefrag(url)[0]
```

---

### 🔍 Q8: **Why use BFS over DFS?**

> “大概解决方案是 BFS”

#### ✅ Answer:

* **BFS** ensures broader crawl coverage early (like a Googlebot).
* Avoids deep single-path exploration.
* Easier to parallelize with a queue structure.

---

## ✅ Interview Plan Summary

| Time   | Task                                            |
| ------ | ----------------------------------------------- |
| 0–5m   | Implement DFS crawler                           |
| 5–10m  | Add BFS + dedup                                 |
| 10–20m | Make it multi-threaded                          |
| 20–30m | Handle `#`, domain check, discuss system design |
