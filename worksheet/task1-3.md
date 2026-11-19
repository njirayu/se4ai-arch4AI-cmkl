# Week 6 worksheet: serving patterns

**Name:Natthakit Jirayuthanyawat**

**Student ID: 01326174**

This is **individual work**: each student completes and submits their own copy, in their own fork.

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in English.

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 491 | 583 | 93.5% | 100% |
| t0-small | 23 | 38 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

>The large model provides much better accuracy 93.5%, while the small model's 72.5% falls below the 90% target (QR3). However, the large model is slower (491 ms vs. 23 ms at p50) and every review goes to it, so every review costs money.

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 30.5 | 244 | 403 | 21 | 0 | 92.7% |
| t1-after | 9.6 | 401 | 1007 | 0 | 27 | 93.1% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

>The p50 only measures successful requests, so the 21 failed requests are excluded from the latency calculation. This makes the response time appear faster because roughly 1 in 10 users receives no answer at all. The errors column tells the real story, while the run time also shows the impact: t1-before took 30.5 seconds compared with only 9.6 seconds in t1-after after fallback was added.

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

>A waiting request holds an open connection, memory, and a worker slot in both the review service and gateway. Since the load test only allows 10 requests in flight, a hung request can occupy a slot for 10–20 seconds instead of around 200 ms, causing healthy requests to queue behind it. This explains why t1-before took 30.5 seconds compared with 9.6 seconds in t1-after.

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

>Pending reviews should be placed in a queue and processed later instead of keeping the original request waiting. This uses the asynchronous/batch topology (T4).

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

>A 0.1-second timeout is shorter than the normal large-model response time of 140–260 ms, so almost every request would time out and go pending, breaking FR2 in practice. A 10-second timeout would make requests wait too long, violating QR2 (wait at most 1 second) and QR1 (95% answered within 1 second). This would create problems similar to t1-before.

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 8 | 544 | 91.5% | 38% |
| t2-0.7 | 0.7 | 360 | 584 | 93.0% | 53% |
| t2-0.8 | 0.8 | 496 | 708 | 93.5% | 64% |
| t2-0.9 | 0.9 | 482 | 644 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

>I would choose a threshold of 0.6 because it sends only 38% of reviews to the large model, the lowest among all four thresholds, reducing inference costs while still passing QR3. It also achieves a fast p50 of 8 ms and p95 of 544 ms. However, I give up some accuracy margin, achieving 91.5%, only 1.5 percentage points above the 90% requirement and up to 2 points below the higher thresholds.

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

>The cascade handles the median review using the fast small model, which explains why p50 drops from 491 ms to just 8 ms. However, the slowest 5% still include reviews escalated to the large model, which must run after the small model, so p95 only improves slightly from 583 ms to 544 ms.

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

>A review like "Not bad at all, I'd come back" may be confidently classified as negative because the small model misinterprets the negation. Since its confidence is high, it won't be escalated to the large model, resulting in a false alert to the restaurant owner. Repeated false alerts could cause owners to lose trust in the system and eventually ignore notifications.

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache-0.6 | 7 | 508 | 91.5% | 31% | 36% |

**Q3.1** Which reviews benefit from the cache, and which never will?

>Short and generic reviews like "Great food", "Delicious!" or "Very good" benefit from the cache because many people may write exactly the same text. Long, detailed or personal reviews rarely benefit because even a small difference in wording or punctuation creates a cache miss. The cache matches exact text, not reviews with similar meanings.

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

>The old cached predictions must not be reused because they were generated by the previous model and may no longer be accurate. I would include the model version in the cache key so the new model automatically uses separate entries. Old entries will eventually be removed through LRU eviction, so a manual cache flush is unnecessary.

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

>I learned that reducing inference costs does not necessarily improve response time for every request. The cascade reduced large-model calls by 62%, but p95 only improved slightly from 583 ms to 544 ms because uncertain reviews still had to wait for the large model.
