# Exploring GuideLLM: Benchmarking a Live LLM on OpenShift

A hands-on guide to running GuideLLM benchmarks against a vLLM model served via KServe on Red Hat OpenShift, interpreting the results, and understanding what the metrics mean for production capacity planning.

---

## What We Are Building

In this blog, we will set up and run a GuideLLM benchmark against a live LLM deployment on OpenShift. The goal is to answer a straightforward question: how does this model perform under increasing load, and at what point does the user experience start to degrade?

We will run GuideLLM as a Kubernetes Job inside the cluster, targeting a Qwen3-0.6B model served through KServe with vLLM as the inference engine. GuideLLM will execute a **sweep** profile, automatically increasing the request rate from idle to maximum throughput across 10 rounds, and produce an interactive HTML report with detailed latency and throughput metrics.

By the end, we will have concrete numbers for time to first token, inter-token latency, throughput, and request latency at every load level, along with a clear understanding of the model's capacity limit on our hardware.

> **Fig 1:** GuideLLM HTML Report — Metrics Details

---

## Why GuideLLM?

GuideLLM is an SLO-aware benchmarking and evaluation tool designed specifically for LLM inference endpoints. It is part of the vLLM project and was originally created by Neuralmagic (now part of Red Hat).

Unlike general-purpose load testing tools such as `wrk` or `Locust`, GuideLLM understands the nature of LLM workloads. It knows about tokens, streaming responses, time-to-first-token, and the specific latency patterns that matter when serving generative AI models. It supports six different load profiles (`synchronous`, `concurrent`, `throughput`, `constant`, `poisson`, and `sweep`), can generate synthetic data calibrated to a specific tokenizer, and produces both machine-readable JSON and a visual HTML report.

For our purposes, the **sweep** profile is the most informative starting point. It automatically tests increasing request rates so we can observe exactly where the model transitions from healthy to saturated, without having to guess the right concurrency levels upfront.

---

## Why Run Inside the Cluster?

GuideLLM can be installed locally on a laptop and pointed at an external Route:

```bash
pip install guidellm[recommended]
guidellm benchmark run --target "https://<your-route-url>" --model "Qwen/Qwen3-0.6B" --profile sweep
```

This works for quick tests, but the numbers will include network latency from the laptop to the cluster, which pollutes the results.

Running GuideLLM as a **Kubernetes Job** inside the cluster eliminates that noise. The Job pod communicates with the model service over the internal cluster network, giving us pure inference performance numbers. The container image `ghcr.io/vllm-project/guidellm:v0.5.0` comes ready to use with no additional installation.

---

## A Note on KServe and vLLM

For anyone unfamiliar with the relationship between these two components:

- **KServe** is the Kubernetes-native serving platform that handles infrastructure concerns — autoscaling, traffic routing, canary deployments, and health checks.
- **vLLM** is the inference engine that actually loads the model weights, runs the forward pass, and generates tokens.

When a model is deployed through KServe on OpenShift AI, KServe creates the pods and services, and inside those pods vLLM does the actual work. The target for our benchmark is the vLLM engine, managed and exposed by KServe.

---

## The Full Stack

Here is the complete environment for this benchmark:

| Component | Details |
|-----------|---------|
| **Model** | Qwen/Qwen3-0.6B (0.6B parameters) |
| **Inference Engine** | vLLM via KServe InferenceService |
| **Benchmarking Tool** | GuideLLM v0.5.0 (as Kubernetes Job) |
| **Cluster** | OpenShift 4.17 on AWS (ROSA) |
| **GPU** | NVIDIA Tesla T4 (single GPU node) |
| **Model Namespace** | `my-first-model` |
| **Benchmark Namespace** | `guidellm-lab` |

Everything runs inside the OpenShift cluster. GuideLLM communicates with vLLM over the internal service network, so the results reflect pure inference performance without external network overhead.

---

## Architecture

> **Fig 2:** Benchmark Architecture

The benchmark flow works as follows:

1. The GuideLLM Job pod starts in the `guidellm-lab` namespace
2. It connects to the vLLM model service in `my-first-model` namespace using internal cluster DNS
3. GuideLLM downloads the Qwen3 tokenizer from Hugging Face to calibrate its synthetic data generator
4. It begins sending requests at increasing rates (sweep profile — 10 rounds)
5. Results are written to a PersistentVolumeClaim so they survive after the Job pod terminates

---

## Prerequisites

Before starting, make sure you have:

- A model already deployed and serving via KServe (we use Qwen3-0.6B, but any OpenAI-compatible endpoint works)
- The `oc` CLI installed and logged in with cluster-admin access
- GPU nodes available in the cluster with the NVIDIA GPU Operator and Node Feature Discovery installed

---

## Step 1: Create the Namespace

Create a dedicated namespace for benchmarking work, separate from the model serving namespace.

```bash
oc new-project guidellm-lab
```

---

## Step 2: Create the Results PVC

When a Kubernetes Job completes, the pod is terminated and everything in the container filesystem is lost. We need a PersistentVolumeClaim to persist the benchmark outputs beyond the pod lifecycle.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: guidellm-results-pvc
  namespace: guidellm-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Apply it:

```bash
oc apply -f manifests/02-pvc.yaml
```

Verify:

```bash
oc get pvc -n guidellm-lab
# Expected: guidellm-results-pvc   Bound   ...   5Gi   RWO
```

---

## Step 3: Deploy the GuideLLM Benchmark Job

This is where GuideLLM gets configured. The Job manifest defines what to benchmark, how to benchmark it, and where to store the results.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: guidellm-first-benchmark
  namespace: guidellm-lab
spec:
  template:
    spec:
      containers:
      - name: guidellm
        image: ghcr.io/vllm-project/guidellm:v0.5.0
        env:
        - name: HOME
          value: /results
        command: ["guidellm"]
        args:
        - "benchmark"
        - "run"
        - "--target"
        - "https://qwen3-0-6b-kserve-workload-svc.my-first-model.svc.cluster.local:8000"
        - "--model"
        - "Qwen/Qwen3-0.6B"
        - "--backend-kwargs"
        - '{"verify": false}'
        - "--data"
        - '{"prompt_tokens":256,"output_tokens":128}'
        - "--profile"
        - "sweep"
        - "--max-seconds"
        - "30"
        - "--output-dir"
        - "/results"
        - "--outputs"
        - "benchmarks.json,benchmarks.html"
        volumeMounts:
        - name: results
          mountPath: /results
      volumes:
      - name: results
        persistentVolumeClaim:
          claimName: guidellm-results-pvc
      restartPolicy: Never
  backoffLimit: 1
```

### What Each Argument Does

| Argument | Value | Purpose |
|----------|-------|---------|
| `--target` | `https://...svc.cluster.local:8000` | Internal HTTPS service URL of the model. Must be `https` — KServe enforces TLS. |
| `--model` | `Qwen/Qwen3-0.6B` | Tells GuideLLM which tokenizer to download for accurate token counting. |
| `--backend-kwargs` | `{"verify": false}` | Disables SSL certificate verification — needed for KServe's self-signed certs. |
| `--data` | `{"prompt_tokens":256,"output_tokens":128}` | Synthetic workload: 256-token prompts, 128-token responses. |
| `--profile` | `sweep` | Automatically tests increasing request rates across 10 rounds. |
| `--max-seconds` | `30` | Each rate level runs for 30 seconds. |
| `--outputs` | `benchmarks.json,benchmarks.html` | Produce both JSON (programmatic) and HTML (visual report). |

Apply it:

```bash
oc apply -f manifests/03-benchmark-job.yaml
```

---

## Why the Tokenizer Matters

When GuideLLM starts, one of the first things it does is download the tokenizer for the target model. The logs show:

```
✔ Processor resolved — Using model 'Qwen/Qwen3-0.6B' as processor
```

This is necessary because GuideLLM needs to generate synthetic prompts of exactly 256 tokens, and "256 tokens" means something different for every model. Each model has its own vocabulary and its own way of splitting text into tokens. The word "benchmarking" might be one token for one model and two tokens for another.

GuideLLM downloads the small tokenizer file (not the model weights, just the vocabulary mapping), uses it to precisely calibrate token counts, and generates prompts that are exactly 256 tokens in the target model's language.

**Important for disconnected environments:** The synthetic prompt corpus is built into GuideLLM, but the tokenizer requires an internet download. In an air-gapped cluster, you would need to pre-download the tokenizer and make it available via a PVC. Red Hat has a detailed article covering this workflow: [Benchmarking GuideLLM in air-gapped OpenShift clusters](https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters).

---

## Step 4: Monitor the Job

Watch the Job progress:

```bash
oc get pods -n guidellm-lab -w
```

The pod will go through `Init` → `Running` (several minutes as it executes the 10 sweep rounds) → `Completed`.

You can also tail the logs in real time:

```bash
oc logs -f job/guidellm-first-benchmark -n guidellm-lab
```

---

## Step 5: Retrieve the Results

Since the Job pod is terminated after completion, we need a helper pod to access the PVC and copy the files out.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-inspector
  namespace: guidellm-lab
spec:
  containers:
  - name: inspector
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: results
      mountPath: /mnt/results
  volumes:
  - name: results
    persistentVolumeClaim:
      claimName: guidellm-results-pvc
```

Apply it and copy the results:

```bash
# Deploy the inspector pod
oc apply -f manifests/04-pvc-inspector.yaml

# Wait for it to be running
oc get pods -n guidellm-lab -w

# Copy results to your local machine
oc cp guidellm-lab/pvc-inspector:/mnt/results/benchmarks.html ./benchmarks.html
oc cp guidellm-lab/pvc-inspector:/mnt/results/benchmarks.json ./benchmarks.json

# Open the report
open benchmarks.html
```

---

## Troubleshooting: Two Gotchas with KServe

During the initial setup, we ran into two issues worth documenting.

### Issue 1: Server disconnected without sending a response

```
httpcore.RemoteProtocolError: Server disconnected without sending a response.
```

**Cause:** The target URL used `http://`. KServe enforces TLS by default, so the model only listens on HTTPS.

**Fix:** Change the URL scheme to `https://`.

### Issue 2: Unexpected keyword argument verify_ssl

```
TypeError: OpenAIHTTPBackend.__init__() got an unexpected keyword argument 'verify_ssl'
```

**Cause:** KServe uses self-signed certificates for internal TLS. We tried to disable verification with `verify_ssl`, but the correct parameter name is `verify`.

**Fix:** Use `--backend-kwargs '{"verify": false}'` (not `verify_ssl`).

---

## Understanding the Results

GuideLLM produces an interactive HTML report with several sections. Here is how to read each one.

### Workload Details

> **Fig 3:** Workload Details — Prompt, Server, and Generated cards

The report header confirms the model (`Qwen/Qwen3-0.6B`) and the timestamp. Below that, three cards summarise the workload:

- **Prompt** — Sample request, mean prompt length (**264 tokens**), token distribution chart. The single spike at 264 confirms all synthetic prompts were the same size.
- **Server** — Target URL, **10 benchmarks** (the 10 sweep rounds), rate type. The bar chart shows request volume over time.
- **Generated** — Sample response from the model. Qwen3 responses start with `<think>` tags (built-in chain-of-thought reasoning). Mean generated length: **128 tokens**.

---

### Interactive Sweep Slider

> **Fig 4:** Interactive Sweep Slider with four metric charts

A slider at the bottom moves from **0.37 rps** (lightest load) to **8.53 rps** (maximum throughput). Four line charts update as you move the slider:

| Metric | At Low Load (0.37 rps) | At Max Load (8.53 rps) | What It Means |
|--------|----------------------|----------------------|---------------|
| **Time to First Token** | 34.59 ms | 5000+ ms | How long before the first word appears |
| **Inter-Token Latency** | 20.06 ms | 147 ms | Gap between each streamed token |
| **Time Per Request** | 2.58 s | 24 s | Total time for a complete response |
| **Throughput** | 50.87 tok/s | 2000+ tok/s | Aggregate tokens per second (GPU utilisation) |

The first three metrics degrade with load. Throughput is the one metric that goes *up* — but this is aggregate throughput. Each individual user's experience is slower even though the GPU is busier overall.

---

### Metrics Details: Percentile Breakdown

> **Fig 5:** TTFT and ITL charts with p50, p90, p95, p99 percentile lines

Below the slider, four detailed charts plot each metric against requests per second with percentile lines (**p50**, **p90**, **p95**, **p99**) and the **mean**.

**What are percentiles?** p50 is the median — half the requests were faster. p90 means 90% were faster. p99 means 99 out of 100 were faster. The gap between p50 and p99 reveals how consistent the experience is. For production SLOs, p99 matters more than the mean — it tells you about the worst-case experience for real users.

**Time to First Token at 0.37 rps:**

| Percentile | Value |
|-----------|-------|
| p50 | 31.98 ms |
| p90 | 34.59 ms |
| p95 | 612.22 ms |
| p99 | 612.22 ms |
| Mean | 80.42 ms |

The p50 and p90 are tight (32–35 ms), indicating a consistent experience for most requests. The p95/p99 jump to 612 ms suggests a handful of outliers from warmup or garbage collection. The mean (80 ms) gets pulled up by these outliers — this is why percentiles are more useful than averages for capacity planning.

**Inter-Token Latency at 0.37 rps:**

| Percentile | Value |
|-----------|-------|
| p50 | 19.74 ms |
| p90 | 20.06 ms |
| p95 | 20.29 ms |
| p99 | 20.29 ms |
| Mean | 19.82 ms |

The spread across all percentiles is less than **0.55 milliseconds**. Extremely consistent token generation at low load.

> **Fig 6:** Time Per Request and Throughput charts

**Time Per Request at 0.37 rps:**

| Percentile | Value |
|-----------|-------|
| p50 | 2.54 s |
| p90 | 2.58 s |
| p95 | 3.19 s |
| p99 | 3.19 s |

**Throughput** shows the percentile lines fanning apart at higher loads. The mean line (aggregate throughput) climbs steadily, but the p50 line (individual user experience) flattens much earlier.

The key insight: there is a **clear inflection point around 5–6 requests per second**. Below that, all metrics are stable. Above it, metrics degrade exponentially — the "hockey stick" curve that defines the model's saturation point.

---

## Capacity Summary

Based on the sweep results for Qwen3-0.6B on a single NVIDIA Tesla T4:

| Metric | Value |
|--------|-------|
| **Safe operating range** | 1–5 requests per second |
| **TTFT at low load** | ~32 ms (p50) |
| **ITL at low load** | ~20 ms (p50), <0.55 ms variance across percentiles |
| **Full response time** | ~2.5 s for 128 tokens |
| **Saturation point** | Beyond 6 rps — TTFT jumps from ms to seconds |
| **Max aggregate throughput** | ~2000 tok/s (with degraded per-user latency) |
| **Recommended SLO** | TTFT < 200 ms at p99, ITL < 50 ms at p99 |
| **Capacity limit** | 5 concurrent users, autoscale beyond that |

---

## What is Next

This was the first step in a series exploring GuideLLM on OpenShift. We ran a single sweep profile with one synthetic data configuration against one model. In the next post, we will go deeper into each metric GuideLLM produces, explore the gap between mean and p99, and examine which metrics matter most for different use cases such as chat applications, RAG pipelines, and batch processing.

Future posts in this series will cover all six load profiles, synthetic versus real datasets, SLO validation, over-saturation detection, Red Hat AI Inference Server versus community vLLM, and distributed inference with llm-d.

The complete manifests and deployment files are available on [GitHub](https://github.com/nirjhar17/guidellm-on-openshift).

---

## References

- [GuideLLM Repository](https://github.com/vllm-project/guidellm)
- [Red Hat: Deploy and benchmark vLLM with GuideLLM on Kubernetes (Dec 2025)](https://developers.redhat.com/articles/2025/12/24/how-deploy-and-benchmark-vllm-guidellm-kubernetes)
- [Red Hat: GuideLLM — Evaluate LLM deployments (Jun 2025)](https://developers.redhat.com/articles/2025/06/20/guidellm-evaluate-llm-deployments-real-world-inference)
- [Red Hat: Benchmarking GuideLLM in air-gapped OpenShift clusters (Sep 2025)](https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters)

---

## About Me

I work on OpenShift, OpenShift AI, and observability solutions, focusing on simplifying complex setups into practical, repeatable steps for platform and development teams.

🐙 GitHub: [github.com/nirjhar17](https://github.com/nirjhar17)

💼 LinkedIn: [linkedin.com/in/nirjhar-jajodia](https://linkedin.com/in/nirjhar-jajodia)

---

## Disclaimer

The views and opinions expressed in this article are my own and do not necessarily reflect the official policy or position of my employer. This guide is provided for educational purposes, and I make no warranties about the completeness, reliability, or accuracy of this information.
