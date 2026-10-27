# Phase 1 — Comparison Report

Answers follow the template's `<!--answer:...-->` markers.

## 1. Setup facts

Image size in decimal MB (compressed registry layers):

<!--answer:image_size_mb-->
50.249893
<!--/answer-->

VPC and subnet:

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

The image is echo-bot:v1 in us-central1/eurecomgpt; the VM and Cloud Run use the same digest. Its unpacked VM size is 136.659632 MB. Debian 12 VM: e2-micro, 10 GB pd-standard disk, us-central1-a. TCP 8080 is public; TCP 22 is restricted to IAP (35.235.240.0/20). Both ingress rules target echo-http, so untagged instances do not inherit access.

## 2. Measurements

All values below are milliseconds.

| Platform | First request | Warm median |
|---|---:|---:|
| IaaS VM | <!--answer:vm_cold_ms-->253.1<!--/answer--> | <!--answer:vm_warm_median_ms-->212.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->4326.1<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->136.1<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->915.0<!--/answer--> | <!--answer:function_warm_median_ms-->131.2<!--/answer--> |

<!--answer:latency_plot-->
Warm HTTP latencies (ms), 20 sequential requests per platform. Each # = one sample.

| Platform | Min | Median | p95 | Max |
|---|---:|---:|---:|---:|
| vm | 210.6 | 212.4 | 215.5 | 216.0 |
| cloudrun | 121.9 | 136.1 | 686.5 | 689.9 |
| function | 123.1 | 131.2 | 681.0 | 2164.5 |

```text
vm (separate bin ranges in ms):
  [210, 212) ###### (6)
  [212, 214) ########### (11)
  [214, 216) ## (2)
  [216, 218) # (1)
cloudrun (separate bin ranges in ms):
  [120, 263) ############## (14)
  [263, 406)  (0)
  [406, 549)  (0)
  [549, 692) ###### (6)
function (separate bin ranges in ms):
  [120, 632) ################# (17)
  [632, 1144) ## (2)
  [1144, 1656)  (0)
  [1656, 2168) # (1)
```
<!--/answer-->

Protocol: correctness GET/POST checks finished at 2026-10-27T14:13:19.594044+00:00. Endpoints then received no deliberate traffic for 22.23 minutes; control-plane instance-count evidence was checked before running the unchanged measure.py once with all three URLs and its default 20 warm requests. All first responses were HTTP 200 and echoed the nonce correctly. The VM first request is an already-running baseline, not a scale-to-zero cold start. Latencies include client/network/protocol overhead; the harness opens a new connection each time. The p95 uses the provided harness's empirical rank, not interpolation. Warm medians were lower for both serverless endpoints than for the VM in this run; this compares complete network paths (HTTP on the VM, HTTPS on serverless), not raw compute. The warm label describes request order; instance reuse is not guaranteed, and the recorded tail spikes are retained. URLs, verification results, timestamps and deployment evidence are in phase1_evidence.json.

## 3. Per-platform analysis

**Scaling behaviour.**

<!--answer:scaling_behaviour-->
The VM creates nothing automatically: one Gunicorn worker and eight threads share the e2-micro's 1 GiB RAM and shared CPU. One, ten or a hundred clients still use one machine; work queues as capacity saturates. Cloud Run and the gen-2 function create container instances automatically. Both have 1 vCPU, 256 MiB, concurrency 8, minimum 0 and maximum 3 instances: normally up to 24 admitted concurrent requests; excess work queues and may receive HTTP 429. Brief platform overshoot is possible; an instance cap is not a spending limit. [Autoscaling reference](https://docs.cloud.google.com/run/docs/about-instance-autoscaling).
<!--/answer-->

**Deployment effort.**

<!--answer:deployment_effort-->
I built and tested the multi-stage image locally, then published it with Cloud Build after the local Docker daemon could not reach Artifact Registry. The VM required a VPC, subnet, tagged firewall rules, IAP SSH, Debian, Docker, registry authentication and repository-reader IAM. I maintain its OS, container restart policy and capacity. Cloud Run consumed the same image digest with port, scaling, resource and public-invoker settings. The function instead uploaded main.py/app.py and requirements; Google built its image from source. Its builder has repository-write, source-object-read and log-write permissions; serverless runtime identities need no application roles. Google maintains serverless hosts, TLS, scheduling and autoscaling.
<!--/answer-->

## 4. Cloud Run and gen-2 function convergence

Complete, unmodified output of `gcloud run services list`:

<!--answer:run_services_list-->
```text
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-321318688734.us-central1.run.app
LAST DEPLOYED BY: service-321318688734@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-27T14:11:33.804392Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-321318688734.us-central1.run.app
LAST DEPLOYED BY: gabin.bruniaux@eurecom.fr
LAST DEPLOYED AT: 2026-10-27T14:12:15.894205Z
```
<!--/answer-->

<!--answer:convergence_explanation-->
Warm medians are close: Cloud Run 136.1 ms and Function 131.2 ms. Cold requests differ: 4326.1 ms versus 915.0 ms. The listing shows echo alongside echo-bot: the gen-2 function is itself a Cloud Run service, deployed by the Cloud Functions service agent. Both use managed containers, HTTPS routing and scale-to-zero autoscaling. Different images, layer caches, framework initialization and network/TLS overhead can affect first-request latency; this single sample does not establish the cause. For Cloud Run I supplied the Dockerfile and Gunicorn command. For the function I supplied Python source, requirements and the echo entry point; buildpacks, Cloud Build and the Functions Framework assembled its serving image. Google manages scheduling, hosts and function runtime integration. [Function architecture](https://docs.cloud.google.com/run/docs/functions/comparison); [source build process](https://docs.cloud.google.com/functions/docs/building).
<!--/answer-->
