Exploring GuideLLM: Benchmarking a Live LLM on OpenShift


A hands-on guide to running GuideLLM benchmarks against a vLLM model served via KServe on Red Hat OpenShift, interpreting the results, and understanding what the metrics mean for production capacity planning.


What We Are Building

In this blog, we will set up and run a GuideLLM benchmark against a live LLM deployment on OpenShift. The goal is to answer a straightforward question: how does this model perform under increasing load, and at what point does the user experience start to degrade?

We will run GuideLLM as a Kubernetes Job inside the cluster, targeting a Qwen3-0.6B model served through KServe with vLLM as the inference engine. GuideLLM will execute a sweep profile, automatically increasing the request rate from idle to maximum throughput across 10 rounds, and produce an interactive HTML report with detailed latency and throughput metrics.

By the end, we will have concrete numbers for time to first token, inter-token latency, throughput, and request latency at every load level, along with a clear understanding of the model's capacity limit on our hardware.

Press enter or click to view image in full size

Fig 1: GuideLLM HTML Report — Metrics Details


Why GuideLLM?

GuideLLM is an SLO-aware benchmarking and evaluation tool designed specifically for LLM inference endpoints. It is part of the vLLM project and was originally created by Neuralmagic (now part of Red Hat).

Unlike general-purpose load testing tools such as wrk or Locust, GuideLLM understands the nature of LLM workloads. It knows about tokens, streaming responses, time-to-first-token, and the specific latency patterns that matter when serving generative AI models. It supports six different load profiles (synchronous, concurrent, throughput, constant, Poisson, and sweep), can generate synthetic data calibrated to a specific tokenizer, and produces both machine-readable JSON and a visual HTML report.

For our purposes, the sweep profile is the most informative starting point. It automatically tests increasing request rates so we can observe exactly where the model transitions from healthy to saturated, without having to guess the right concurrency levels upfront.


Why Run Inside the Cluster?

GuideLLM can be installed locally on a laptop with pip install guidellm and pointed at an external Route. This works for quick tests, but the numbers will include network latency from the laptop to the cluster, which pollutes the results.

Running GuideLLM as a Kubernetes Job inside the cluster eliminates that noise. The Job pod communicates with the model service over the internal cluster network, giving us pure inference performance numbers. The container image ghcr.io/vllm-project/guidellm:v0.5.0 comes ready to use with no additional installation.


A Note on KServe and vLLM

For anyone unfamiliar with the relationship between these two components: KServe is the Kubernetes-native serving platform that handles infrastructure concerns such as autoscaling, traffic routing, canary deployments, and health checks. vLLM is the inference engine that actually loads the model weights, runs the forward pass, and generates tokens.

When a model is deployed through KServe on OpenShift AI, KServe creates the pods and services, and inside those pods vLLM does the actual work. The target for our benchmark is the vLLM engine, managed and exposed by KServe.


The Full Stack

Here is the complete environment for this benchmark:

- Qwen3-0.6B — A 0.6-billion parameter language model from Alibaba, small enough for experimentation on a single GPU while still producing meaningful inference behaviour. Served via vLLM through a KServe InferenceService.
- vLLM — High-performance inference engine with PagedAttention, continuous batching, and OpenAI-compatible API. Runs inside the KServe pod.
- KServe — Kubernetes-native model serving platform handling pod lifecycle, service routing, and TLS termination.
- GuideLLM v0.5.0 — LLM benchmarking tool, run as a Kubernetes Job with results persisted to a PVC.

Everything runs inside the OpenShift cluster. GuideLLM communicates with vLLM over the internal service network, so the results reflect pure inference performance without external network overhead.


Architecture

Press enter or click to view image in full size

Fig 2: Benchmark Architecture

The benchmark flow works as follows. The GuideLLM Job pod starts in the guidellm-lab namespace and connects to the vLLM model service in the my-first-model namespace using the internal cluster DNS. GuideLLM downloads the Qwen3 tokenizer from Hugging Face to calibrate its synthetic data generator, then begins sending requests at increasing rates. Results are written to a PersistentVolumeClaim so they survive after the Job pod terminates.


OpenShift Cluster Details

All components run across two namespaces:

- Qwen3-0.6B (vLLM via KServe) — namespace my-first-model, endpoint https://qwen3-0-6b-kserve-workload-svc.my-first-model.svc.cluster.local:8000
- GuideLLM Job — namespace guidellm-lab
- Results PVC — namespace guidellm-lab, 5Gi ReadWriteOnce

Cluster: OpenShift 4.17 on AWS (ROSA)
GPU: NVIDIA Tesla T4 (single GPU node)


Prerequisites

Before starting, make sure you have:

- A model already deployed and serving via KServe (we use Qwen3-0.6B, but any OpenAI-compatible endpoint works)
- The oc CLI installed and logged in with cluster-admin access
- GPU nodes available in the cluster with the NVIDIA GPU Operator and Node Feature Discovery installed


Step 1: Create the Namespace

Create a dedicated namespace for benchmarking work, separate from the model serving namespace.

oc new-project guidellm-lab


Step 2: Create the Results PVC

When a Kubernetes Job completes, the pod is terminated and everything in the container filesystem is lost. We need a PersistentVolumeClaim to persist the benchmark outputs (JSON and HTML files) beyond the pod lifecycle.

oc apply -f manifests/02-pvc.yaml

The PVC requests 5Gi with ReadWriteOnce access mode, which is more than sufficient for benchmark results.


Step 3: Deploy the GuideLLM Benchmark Job

This is where GuideLLM gets configured. The Job manifest defines what to benchmark, how to benchmark it, and where to store the results.

oc apply -f manifests/03-benchmark-job.yaml

Here is what each argument in the Job does:

- target — The internal HTTPS service URL of the model. KServe enforces TLS by default, so this must be https, not http.
- model — Tells GuideLLM which tokenizer to download. This is used for accurate token counting when generating synthetic prompts (see the section on tokenizers below).
- backend-kwargs — Set to {"verify": false} to disable SSL certificate verification. This is necessary because KServe uses self-signed certificates for internal TLS.
- data — Defines the synthetic workload: 256-token prompts with 128-token responses.
- profile — Set to sweep, which automatically tests increasing request rates across 10 rounds.
- max-seconds — Each rate level in the sweep runs for 30 seconds before moving to the next.
- outputs — Produce both benchmarks.json (for programmatic analysis) and benchmarks.html (for the visual report).


Why the Tokenizer Matters

When GuideLLM starts, one of the first things it does is download the tokenizer for the target model. The logs show: "Processor resolved — Using model Qwen/Qwen3-0.6B as processor."

This is necessary because GuideLLM needs to generate synthetic prompts of exactly 256 tokens, and "256 tokens" means something different for every model. Each model has its own vocabulary and its own way of splitting text into tokens. The word "benchmarking" might be one token for one model and two tokens for another.

GuideLLM downloads the small tokenizer file (not the model weights, just the vocabulary mapping), uses it to precisely calibrate token counts, and generates prompts that are exactly 256 tokens in the target model's language. The synthetic prompt corpus itself is built into GuideLLM, so that part works without internet access. Only the tokenizer requires a download.

This has implications for disconnected environments. In an air-gapped cluster, the tokenizer download will fail. You would need to pre-download the tokenizer and make it available via a PVC. Red Hat has a detailed article covering this workflow: Benchmarking GuideLLM in air-gapped OpenShift clusters (https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters).


Step 4: Monitor the Job

Watch the Job progress.

oc get pods -n guidellm-lab -w

The pod will go through Init, Running (for several minutes as it executes the 10 sweep rounds), and then Completed.


Step 5: Retrieve the Results

Since the Job pod is terminated after completion, we need a helper pod to access the PVC and copy the files out.

oc apply -f manifests/04-pvc-inspector.yaml

Wait for the inspector pod to be running, then copy the results.

oc cp guidellm-lab/pvc-inspector:/mnt/results/benchmarks.html ./benchmarks.html
oc cp guidellm-lab/pvc-inspector:/mnt/results/benchmarks.json ./benchmarks.json

Open benchmarks.html in a browser to view the interactive report.


Troubleshooting: Two Gotchas with KServe

During the initial setup, we ran into two issues that are worth documenting.

Issue 1: Server disconnected without sending a response. The first attempt used http in the target URL. KServe enforces TLS by default, so the model was only listening on HTTPS. Changing the URL scheme to https resolved this.

Issue 2: unexpected keyword argument verify_ssl. KServe uses self-signed certificates for internal TLS, and GuideLLM's HTTP client rejected the certificate. The correct way to disable SSL verification is through backend-kwargs with the value {"verify": false}. Note that the parameter is verify, not verify_ssl — the latter causes a TypeError.


Understanding the Results

GuideLLM produces an interactive HTML report with several sections. Here is how to read each one.


Workload Details

Press enter or click to view image in full size

Fig 3: Workload Details — Prompt, Server, and Generated cards

The report header confirms the model (Qwen/Qwen3-0.6B) and the timestamp. Below that, three cards summarise the workload:

Prompt — Shows a sample of the raw request sent to the model, the mean prompt length (264 tokens), and a token length distribution chart. The distribution shows a single spike at 264, confirming that all synthetic prompts were generated to the same size.

Server — Shows the target URL, that 10 benchmarks were run (the 10 rate levels in the sweep), and the rate type. The bar chart shows request volume over time, with visible spikes corresponding to each of the 10 sweep rounds.

Generated — Shows a sample response from the model. In our case, Qwen3 responses start with a think tag, which is the model's built-in chain-of-thought reasoning. The mean generated length is 128 tokens, exactly as configured.


Interactive Sweep Slider

Press enter or click to view image in full size

Fig 4: Interactive Sweep Slider with four metric charts

This is the most informative part of the report. A slider at the bottom moves from 0.37 requests per second (lightest load) to 8.53 requests per second (maximum throughput). Four line charts update as you move the slider:

Time to First Token (TTFT) — How long a user waits before the first word appears. At low load: 34.59 ms. Stays flat until about 5 rps, then shoots up to over 5000 ms at max load. That is a 150x degradation.

Inter-Token Latency (ITL) — The gap between each subsequent token during streaming. At low load: 20.06 ms. Holds steady until about 5 rps, then spikes to 147 ms. The stream that felt smooth and natural becomes choppy and slow.

Time Per Request — Total time for a complete 128-token response. At low load: 2.58 seconds. Climbs to 24 seconds at max load.

Throughput — Tokens per second. At low load: 50.87 tok/s. This is the one metric that goes up with load because the GPU processes more requests in parallel. At max load, total throughput reaches around 2000 tok/s. However, this is aggregate throughput — each individual user's experience is slower even though the GPU is busier overall.


Metrics Details: Percentile Breakdown

Press enter or click to view image in full size

Fig 5: TTFT and ITL charts with p50, p90, p95, p99 percentile lines

Below the slider, four detailed charts plot each metric against requests per second with multiple percentile lines (p50, p90, p95, p99) and the mean.

Understanding percentiles is critical for production capacity planning. p50 is the median — half the requests were faster than this value. p90 means 90% of requests were faster. p99 means 99 out of 100 requests were faster. The gap between p50 and p99 reveals how consistent the experience is.

Time to First Token at 0.37 rps:
p50: 31.98 ms
p90: 34.59 ms
p95: 612.22 ms
p99: 612.22 ms
Mean: 80.42 ms

The p50 and p90 are remarkably tight (32-35 ms), indicating a very consistent experience for the vast majority of requests. The p95/p99 jump to 612 ms suggests a handful of outlier requests, likely caused by warmup or garbage collection. The mean (80.42 ms) gets pulled up by these outliers, which is why percentiles are more useful than averages for capacity planning.

Inter-Token Latency at 0.37 rps:
p50: 19.74 ms
p90: 20.06 ms
p95: 20.29 ms
p99: 20.29 ms
Mean: 19.82 ms

The spread across all percentiles is less than 0.55 milliseconds. This indicates extremely consistent token generation at low load, with virtually no variance between requests.

Press enter or click to view image in full size

Fig 6: Time Per Request and Throughput charts

Time Per Request at 0.37 rps:
p50: 2.54 s
p90: 2.58 s
p95: 3.19 s
p99: 3.19 s

Throughput shows the percentile lines fanning apart at higher loads. The mean line (aggregate throughput) climbs steadily, but the p50 line (individual user experience) flattens much earlier. The GPU is doing more total work, but each individual request is not getting faster.

The key insight from all four charts is that there is a clear inflection point around 5-6 requests per second. Below that threshold, all metrics are stable and fast. Above it, metrics degrade exponentially. This is the "hockey stick" curve that defines the model's saturation point.


Capacity Summary

Based on the sweep results for Qwen3-0.6B on a single NVIDIA Tesla T4:

Safe operating range: 1 to 5 requests per second
TTFT at low load: approximately 32 ms (p50)
ITL at low load: approximately 20 ms (p50), consistent across all percentiles
Full response time at low load: approximately 2.5 seconds for 128 tokens
Saturation point: beyond 6 rps, TTFT jumps from milliseconds to seconds
Maximum aggregate throughput: approximately 2000 tok/s (but with degraded per-user latency)

For a production deployment, a reasonable SLO would be TTFT under 200 ms at p99 and ITL under 50 ms at p99. Based on these numbers, a capacity limit of 5 concurrent users with autoscaling to add replicas before hitting that threshold would keep the experience within acceptable bounds.


What is Next

This was the first step in a series exploring GuideLLM on OpenShift. We ran a single sweep profile with one synthetic data configuration against one model. In the next post, we will go deeper into each metric GuideLLM produces, explore the gap between mean and p99, and examine which metrics matter most for different use cases such as chat applications, RAG pipelines, and batch processing.

Future posts in this series will cover all six load profiles, synthetic versus real datasets, SLO validation, over-saturation detection, Red Hat AI Inference Server versus community vLLM, and distributed inference with llm-d.

The complete manifests and deployment files are available on GitHub (https://github.com/nirjhar17/guidellm-on-openshift).


References

GuideLLM Repository: https://github.com/vllm-project/guidellm
Red Hat: Deploy and benchmark vLLM with GuideLLM on Kubernetes (Dec 2025): https://developers.redhat.com/articles/2025/12/24/how-deploy-and-benchmark-vllm-guidellm-kubernetes
Red Hat: GuideLLM — Evaluate LLM deployments (Jun 2025): https://developers.redhat.com/articles/2025/06/20/guidellm-evaluate-llm-deployments-real-world-inference
Red Hat: Benchmarking GuideLLM in air-gapped OpenShift clusters (Sep 2025): https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters


About Me

I work on OpenShift, OpenShift AI, and observability solutions, focusing on simplifying complex setups into practical, repeatable steps for platform and development teams.

GitHub: github.com/nirjhar17
LinkedIn: linkedin.com/in/nirjhar-jajodia


Disclaimer

The views and opinions expressed in this article are my own and do not necessarily reflect the official policy or position of my employer. This guide is provided for educational purposes, and I make no warranties about the completeness, reliability, or accuracy of this information.
