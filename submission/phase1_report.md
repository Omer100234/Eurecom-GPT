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
| IaaS VM | <!--answer:vm_cold_ms-->251.1<!--/answer--> | <!--answer:vm_warm_median_ms-->212.1<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1361.8<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->130.4<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1748.5<!--/answer--> | <!--answer:function_warm_median_ms-->132.8<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  251.1ms  warm_median=  212.1ms
          ########################################
[OK  ] cloudrun  cold= 1361.8ms  warm_median=  130.4ms
          #########################
[OK  ] function  cold= 1748.5ms  warm_median=  132.8ms
          #########################
------------------------------------------------------------
```
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
**IaaS VM:** nothing is created. One e2-micro (2 shared vCPUs, 1 GB) runs gunicorn with 1 worker × 8 threads, so ~8 requests are served concurrently and the rest queue; at a hundred requests latency explodes. The hard limit is the machine size is dependant on the machine picked.
**Cloud Run:** new container instances of my image are started automatically, from 0 up to 3, each taking up to 80 concurrent requests (default). The limit is the flag passed.
**Cloud Function:** also creates instances (Cloud Run instances underneath), but with concurrency 1, so ten simultaneous requests need ~ten instances, each paying a cold start. Its limit is the platform's default max-instances, since none is set.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
**IaaS VM:** the most work. I built the network myself (custom VPC , subnet, two firewall rules), granted the VM's service account Artifact Registry read access, created the VM, SSH'd in, installed and authenticated Docker, then pulled and ran the container with a restart policy. I would also have to maintain the OS, Docker, and the ephemeral IP, and there is no HTTPS.
**Cloud Run:** I wrote the Dockerfile and built/pushed the image, then one deploy. Google handles networking, HTTPS, instances, scaling and restarts.
**Cloud Function:** least effort: no Dockerfile or image — I pointed the source at the folder and Google built the container with buildpacks. The cost is less control over the image and a fixed entry-point format.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-456785733525.us-central1.run.app
LAST DEPLOYED BY: service-456785733525@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-11-08T18:12:15.739205Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-456785733525.us-central1.run.app
LAST DEPLOYED BY: omer.gabai45@gmail.com
LAST DEPLOYED AT: 2026-11-08T18:09:38.589657Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
`gcloud run services list` shows two services: `echo-bot` (deployed by me) and `echo`, which I never deployed to Cloud Run — it was "LAST DEPLOYED BY" `gcf-admin-robot`, the Cloud Functions service agent, at the same moment as my function deploy. So a gen-2 Cloud Function *is* a Cloud Run service: Cloud Functions builds a container from my source and deploys it to Cloud Run for me (my deploy output also shows `service: .../services/echo`, revision `echo-00001-qok`). Their cold starts look alike because they are the same mechanism — Cloud Run scaling from zero (1361.8 ms vs 1748.5 ms cold, ~130 ms warm for both). What genuinely differs is the build and defaults: my slim multi-stage Dockerfile vs a buildpack image with the Functions Framework (likely larger, a plausible cause of the extra ~0.4 s), and the function's concurrency of 1 and 0.1666 vCPU vs Cloud Run's concurrency of 80.
<!--/answer-->
