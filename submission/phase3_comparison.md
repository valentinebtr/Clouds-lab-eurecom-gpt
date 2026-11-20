# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->38.5<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.015<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->8.7<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->9<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranked from fastest to slowest: local single-process word count (0.015 s), Spark TF-IDF collect (8.7 s), and cloud MapReduce word count (38.5 s). The single-process vs. cloud word count comparison is like-for-like: 0.015 s does pure in-memory Python work, whereas 38.5 s is dominated by Cloud Workflows orchestration, OIDC HTTP calls, Cloud Run function invocations/cold starts, and GCS shuffle/output I/O. Spark runs a heavier TF-IDF pipeline locally in 8.7 s, confirming that cloud MapReduce's overhead comes from distributed control plane and network/object-storage coordination rather than raw CPU work on 60 documents.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
With 4 stages (stages 0–3 in 3a), the job has 3 shuffle boundaries caused by term_freq (reduceByKey at line 32), doc_freq (reduceByKey at line 45), and build_tfidf_rdd (join at line 58). Nine retries out of 16 tasks (~56%) is above the ~30% (~4.8 retries) expected on average under --chaos 0.3 due to random variance across 16 attempts.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
flatMap, map, reduceByKey, and join are lazy transformations; collect (and toDebugString inspection) triggers execution as an action. Calling build_tfidf_rdd only constructed the logical RDD lineage graph without running tasks or shuffles. Calling collect materialized the DAG across 4 stages, executing the shuffles and returning the 5,975 TF-IDF rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
If a partition from the first reduceByKey (term_freq, stage 0) was lost, Spark would recompute only the affected partition(s) from stage 0's upstream lineage (docs_rdd -> flatMap -> reduceByKey), reusing already-materialized shuffle outputs from unaffected partitions and downstream stages. Recovery starts at the stage 0 / stage 1 shuffle boundary.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run functions<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->mapreduce.py<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is provided by the map_phase parallel step in workflow.yaml: execution cannot proceed to reduce_phase until every parallel map branch finishes and writes its shuffle partitions.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Retries are safe because re-running a map or reduce task for the same split or partition overwrites deterministic output objects rather than accumulating duplicate state. If run_map appended to shuffle files on retry after a partial failure or late response, retried tasks would duplicate (word, 1) pairs, inflating word counts and breaking equivalence with the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3, simulated worker crashes returned HTTP 503 errors that appeared in mr-worker logs and triggered exponential backoff retries in the workflow execution. Despite the injected failures, the workflow finished in SUCCEEDED state (38.5 s) and produced word counts identical to the local reference (matches_local: true).
<!--/answer-->
