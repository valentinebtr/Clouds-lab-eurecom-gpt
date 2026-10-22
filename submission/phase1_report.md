# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->234.4<!--/answer--> | <!--answer:vm_warm_median_ms-->209.5<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1716.2<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->383.1<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1796.3<!--/answer--> | <!--answer:function_warm_median_ms-->135.0<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
[OK  ] vm        cold=  234.4ms  warm_median=  209.5ms

            ######################

[OK  ] cloudrun  cold= 1716.2ms  warm_median=  383.1ms

            ########################################

[OK  ] function  cold= 1796.3ms  warm_median=  135.0ms

            ##############
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The IaaS VM creates nothing automatically; load hits a single static instance, capped entirely by the hardware limits of its e2-micro machine size. Conversely, Cloud Run and Cloud Functions scale out dynamically, automatically spinning up new container instances to absorb concurrent spikes like ten or a hundred requests. Their upper bound is strictly restricted by the --max-instances=3 flag passed during deployment, preventing resource exhaustion while handling traffic bursts.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The IaaS VM required heavy manual configuration: building a custom VPC, subnets, firewall rules for ports 22 and 8080, setting up Artifact Registry IAM permissions, and manually pulling and running the Docker container via SSH. Cloud Run required moderate effort, needing a packaged Docker image pushed to the registry and a single deployment command. Finally, Cloud Functions demanded the least operational overhead, deploying raw Python source code directly via gcloud without managing container registries, runtimes, or servers.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-1060977908829.us-central1.run.app
LAST DEPLOYED BY: service-1060977908829@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-22T14:30:53.179763Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-1060977908829.us-central1.run.app
LAST DEPLOYED BY: valentine.bouteyre@eurecom.fr
LAST DEPLOYED AT: 2026-10-22T14:29:04.148147Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The gcloud run services list output reveals that Cloud Functions gen 2 runs entirely on top of Cloud Run under the hood, managed by the gcf-admin-robot service account. This architectural convergence explains why both platforms exhibit nearly identical cold start latencies (~1.7s), as both must spin up a container instance from zero. The primary difference lies in the abstraction layer and packaging mechanism: Cloud Run expects a pre-built container image stored in Artifact Registry, whereas Cloud Functions accepts raw source code and relies on Google buildpacks to build and manage the container automatically, offering a streamlined developer experience at the cost of less direct control over the underlying container configuration.
<!--/answer-->
