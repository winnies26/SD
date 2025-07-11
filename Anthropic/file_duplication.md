# 💼 File Deduplication (Full Breakdown)

---

## 📌 Question Prompt

> **"Write a program to find duplicate files in a file system."**
>
> Two files are duplicates if they have **identical content**, regardless of filename or path.
> You will work in **Replit**, and are expected to **manually create test files** and **print output**.

---

## ✅ Requirements

* Support nested directories and many files.
* Handle large files efficiently (don't read whole file into memory).
* Optimize for both speed and correctness.
* Expect open-ended follow-up questions about:

  * Hash algorithm tradeoffs
  * Chunking strategy
  * I/O vs CPU bottlenecks
  * Production system design

---

## 🛠️ Step-by-Step Strategy

### 1. Traverse all files using `os.walk()`

```python
import os

for dirpath, _, filenames in os.walk(root_dir):
    for fname in filenames:
        full_path = os.path.join(dirpath, fname)
        print(full_path)
```

---

### 2. Group files by size to skip unnecessary work

```python
from collections import defaultdict

def group_by_size(root_dir):
    size_map = defaultdict(list)
    for dirpath, _, filenames in os.walk(root_dir):
        for fname in filenames:
            path = os.path.join(dirpath, fname)
            try:
                size = os.path.getsize(path)
                size_map[size].append(path)
            except:
                continue
    return size_map
```

---

### 3. Hash file content in chunks (e.g., 4KB)

```python
import hashlib

def compute_sha256(path, chunk_size=4096):
    sha = hashlib.sha256()
    with open(path, 'rb') as f:
        while chunk := f.read(chunk_size):
            sha.update(chunk)
    return sha.hexdigest()
```

---

### 4. Main function to detect duplicates

```python
def find_duplicates(root_dir):
    size_map = group_by_size(root_dir)
    hash_map = defaultdict(list)

    for paths in size_map.values():
        if len(paths) < 2:
            continue
        for path in paths:
            try:
                h = compute_sha256(path)
                hash_map[h].append(path)
            except:
                continue

    return [group for group in hash_map.values() if len(group) > 1]
```

---

## 🧪 Testing Setup (Replit)

```python
import tempfile

def create_test_file(content):
    f = tempfile.NamedTemporaryFile(delete=False, mode='w')
    f.write(content)
    f.close()
    return f.name

f1 = create_test_file("hello world")
f2 = create_test_file("hello world")
f3 = create_test_file("something else")

print(find_duplicates("/tmp"))
```

---

## 🔍 Real Interview Follow-Up Questions + Answers

---

### Q1: **Why chunk read instead of reading whole file?**

> "先写全部读完，然后写 chunk 读"

* Prevents out-of-memory errors on large files.
* Chunking allows streaming file hash.
* More memory-efficient and scalable.

---

### Q2: **Why 4KB chunk size?**

> “chunk size 的选择原因”

* Matches common filesystem block size.
* Good trade-off between I/O calls and memory usage.
* Can tune: 1KB, 4KB, 64KB — benchmark if needed.

---

### Q3: **Which hash should you use and why?**

> “hash 用哪种算法为什么”

| Algorithm   | Pros           | Cons                | Use              |
| ----------- | -------------- | ------------------- | ---------------- |
| MD5         | Fast           | Broken (collisions) | ✅ Quick check    |
| SHA-1       | Legacy only    | Broken              | ❌                |
| **SHA-256** | Strong, secure | Slower              | ✅ Best for dedup |
| BLAKE3      | Super fast     | Not built-in        | ✅ Modern alt     |

**Use SHA-256** + byte-by-byte compare if in doubt.

---

### Q4: **How do you know if it's CPU-bound or I/O-bound?**

> “如何判断是 i/o block 还是 cpu block”

* Use `time.perf_counter()` to measure time.
* If hashing takes long: CPU-bound.
* If `f.read()` is slow: I/O-bound.
* Use tools: `iotop`, `top`, `cProfile`.

---

### Q5: **How to calculate I/O throughput?**

> “如何得到 i/o throughput”

```python
import time
def measure_throughput(path):
    total = 0
    start = time.perf_counter()
    with open(path, 'rb') as f:
        while chunk := f.read(4096):
            total += len(chunk)
    end = time.perf_counter()
    print(f"Throughput: {total / (end - start) / 1e6:.2f} MB/s")
```

---

### Q6: **What if hash collides?**

* Use byte-by-byte comparison.
* Or second hash (e.g., MD5 + SHA-256).
* Rare but possible.

---

### Q7: **Production Design — how to scale this?**

> “dedup 在 production environment 运行的知识”

* Client computes hash → sends to backend.
* Backend checks hash DB:

  * If exists: reference existing file
  * If new: store and index it
* Store content in **blob storage**
* Use **reference counting**
* Run periodic **MapReduce dedup jobs**

---

### Q8: **Do I have to implement SHA-256?**

> "要自己写 sha256 吗"

No — use:

```python
hashlib.sha256()
```

Or in Java:

```java
MessageDigest md = MessageDigest.getInstance("SHA-256");
```

---

### Q9: **How to manually create test files?**

> “要自己创建 file 写 test”

* In Replit/local, use Python:

```python
with open("test1.txt", "w") as f:
    f.write("hello")
```

* Or `tempfile` for cleaner tests.

---

### Q10: **What if they ask for a framework you don’t know?**

> “说随便用 framework 给我整不会了”

Say:

> “I'll use standard Python modules like `os` and `hashlib` for clarity.”

Then write from scratch. That’s acceptable.

---

### Q11: **What's the expected flow in the interview?**

> “先写全部读完 → chunk 读 → group by size → follow-up问题”

| Stage        | Task                         |
| ------------ | ---------------------------- |
| Step 1       | Read full file + hash        |
| Step 2       | Optimize with chunking       |
| Step 3       | Add grouping by file size    |
| Final 15 min | Discuss perf, hashes, system |

