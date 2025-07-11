Great — let’s now walk through the **“Distributed Mode and Median”** system design + algorithm interview question in full detail, incorporating every note from your Anthropic dataset and adding scalable system-level thinking.

This is a **classic distributed data problem** requiring knowledge of:

* MapReduce
* Data partitioning
* Network bottlenecks
* Probabilistic dedup
* Distributed selection (for median)

---

# 📊 Anthropic System Design Interview: Distributed Mode & Median

---

## 📌 Problem Prompt

> **"Given a massive dataset split across N machines, find the mode (most frequent element)."**
> Then as a **follow-up**, find the **median**.
>
> ✅ You are given:

* A cluster of **10 machines**
* Each machine has a local dataset (e.g., millions of integers)
* A fixed **data transmission API** already implemented
* **Network is the bottleneck** — avoid multiple machines sending to the same receiver at the same time

---

## ✅ Problem 1: Distributed Mode

### 🧠 Definitions

* **Mode**: The number that appears most frequently in a dataset.
* In this case: **all values are unique except for two repeated ones** — only two duplicates in the entire distributed dataset.

---

## 💡 Key Observations

* There are **only two duplicates** in the entire dataset.
* Those two numbers **may not be on the same machine**.
* So we need a way to **move them together** for detection.
* We need to **minimize bandwidth** and avoid **congestion**.

---

## 🧩 Strategy: Hash-Based Shuffling (modulo trick)

> “每台机器读完自己的数据之后，把数模十，然后发到对应的机器上”

### 🧠 Why this works:

* Each number `x` is assigned to machine `x % 10`
* This guarantees that if two machines contain the **same number**, both will send it to the **same destination**
* Machines **only send to one machine**, and **each machine receives uniformly** — no congestion

---

## 🛠️ Implementation Plan

### Step 1 – Local Preprocessing

Each machine:

* Scans its local dataset
* Computes `x % 10` for each number
* Sends each number to the corresponding machine

This is similar to a **MapReduce Shuffle step**.

### Step 2 – Local Counting

Each machine:

* Receives a stream of integers
* Maintains a hash map `num → frequency`
* If any number has frequency > 1, it's a duplicate ⇒ possible mode

### Step 3 – Result Aggregation

Each machine returns either:

* A number that appears twice (likely candidate), or
* "No duplicates found"

One coordinator machine aggregates the answers.

---

### 🔍 Example:

Let’s say we have 3 machines:

```
M1: [1, 3, 5, 1001]
M2: [2, 4, 5, 7]
M3: [6, 8, 1001, 10]
```

We hash each number to `num % 10`:

* `5` → M5
* `1001` → M1

Both `5`s will be sent to M5
Both `1001`s will be sent to M1

Each machine can now locally detect duplicates.

---

## 🚫 Bandwidth Bottleneck Awareness

> “数据传输速度会是瓶颈，尽量不要所有机器都同时往一台机器发数据”

Using `mod 10` ensures:

* Even distribution
* No more than one send per machine
* No machine becomes a bottleneck receiver

---

## ✅ Result

Each machine independently detects local duplicates, then the coordinator picks the highest-frequency one (mode). In this case, we expect the result to be **either of the two duplicated values**.

---

## 🔁 Follow-Up: **Distributed Median**

---

## 📌 Problem 2: Distributed Median

> Find the **median** (middle element) of a massive dataset split across 10 machines

---

### 🧠 Constraints

* Each machine has local data (sorted or not)
* You **cannot send all data to one machine**
* Must reduce **network usage**
* Interface to **send/receive is fixed**

---

### 💡 Strategy: Distributed Quickselect / Binary Search

We can treat this like **finding the Kth smallest element** in a distributed setting.

---

## 🧮 Step-by-Step Plan

---

### Step 1 – Coordinator chooses a pivot

* Randomly pick a pivot value (from a sample, or uniformly)
* Broadcast to all machines

---

### Step 2 – Each machine returns:

* `count_lt` = count of values < pivot
* `count_eq` = count of values == pivot
* `count_gt` = count of values > pivot

Each machine computes these counts **locally**.

---

### Step 3 – Aggregation

Coordinator aggregates:

* Total `count_lt`
* Total `count_eq`
* Total `count_gt`

Now compare `count_lt` to desired K (median index):

* If `count_lt` > K → repeat on left half
* If `count_lt + count_eq` < K → repeat on right half
* Else → pivot is the median

---

### 🔁 Repeat Until Converge

Keep narrowing down the range:

* Broadcast new pivot
* Machines send back local counts
* Coordinator updates bounds

This is essentially a **distributed quickselect**.

---

## 🚀 Optimization: Sampling for Pivot

* Each machine sends a **random sample** (e.g., 100 numbers)
* Coordinator selects a pivot from combined sample
* Better convergence than random guesses

---

## 📦 Result

Median is found after **O(log N)** iterations with **minimal data transfer**, only sending **counts and pivots**, not full arrays.

---

## 🧵 Additional Follow-Up Questions

---

### 🔍 Q1: How do you minimize network usage?

* Avoid sending raw data → send counts only
* Compress numbers if transmitting samples
* Batch messages if possible

---

### 🔍 Q2: What if the data isn’t evenly distributed?

* Use **stratified sampling**
* Balance load by breaking largest partitions across multiple machines

---

### 🔍 Q3: What if data is already sorted?

* Use **merge-style median finding**
* Use **two pointers** (like external k-way merge)

---

### 🔍 Q4: Can this be done with MapReduce?

✅ Yes.

For mode:

* Map: `num → 1`
* Reduce: sum counts → max

For median:

* Needs sorting / partial sampling
* Use MapReduce to get distribution
* Do post-processing to find midpoint

---

## ✅ Interview Plan Summary

| Time   | Task                            |
| ------ | ------------------------------- |
| 0–5m   | Understand constraints, scale   |
| 5–10m  | Propose hash + shuffle for mode |
| 10–15m | Handle bandwidth tradeoffs      |
| 15–25m | Binary search for median        |
| 25–30m | Optimize + answer follow-ups    |

---

