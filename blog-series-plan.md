# GuideLLM on OpenShift -- Learning Path + Blog Series

> Primary goal: **Learn GuideLLM hands-on on OpenShift**
> Secondary goal: Blog about the journey (experience, experiments, findings -- not rewriting docs)
>
> Repository: https://github.com/vllm-project/guidellm

---

## How This Plan Works

Each step has 3 parts:

- **LEARN**: Read these resources and follow the hands-on steps
- **EXPERIMENT**: Try things the docs don't cover -- this is where real learning happens
- **BLOG**: Write about your experience, findings, and what surprised you

---

## OpenShift Environment Setup (Do This First)

**What you need:**

- OpenShift 4.14+ cluster with GPU nodes (NVIDIA A100/L4/H100)
- NVIDIA GPU Operator + Node Feature Discovery installed
- `oc` CLI configured
- HuggingFace account + token

```bash
oc new-project guidellm-lab
oc create secret generic hf-secret --from-literal=hf_token=<your-token>
```

---

## Step 1: Understand GuideLLM, Install It, and Run Your First Benchmark

### LEARN Part A: Concepts (read these first)

Understand what GuideLLM is and why it exists:

1. [GuideLLM: Evaluate LLM deployments (Red Hat, Jun 2025)](https://developers.redhat.com/articles/2025/06/20/guidellm-evaluate-llm-deployments-real-world-inference)
   - What GuideLLM does, use cases, why it matters
2. [GuideLLM README](https://github.com/vllm-project/guidellm/blob/main/README.md)
   - Features, comparison with other tools, CLI usage examples
3. [Architecture Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/architecture.md)
   - Internal components: DatasetCreator -> RequestLoader -> Scheduler -> RequestsWorker -> Backend -> BenchmarkAggregator

After reading, you should understand:
- [ ] GuideLLM simulates real users hitting an LLM server and measures performance
- [ ] It works against any OpenAI-compatible endpoint (vLLM, TGI, Red Hat AI Inference Server)
- [ ] Key metrics at a high level: TTFT, ITL, throughput, request latency
- [ ] It supports multiple load profiles (synchronous, concurrent, sweep, etc.)

### LEARN Part B: Installation

**Two places to install -- you need both:**

**1. Local install (on your laptop) -- for inspecting results and preprocessing:**

```bash
pip install guidellm[recommended]
guidellm --help
```

**2. On OpenShift (as a Job) -- for running benchmarks against your cluster:**

No install needed -- you use the container image `ghcr.io/vllm-project/guidellm:v0.5.0` directly in a Job manifest. GuideLLM ships as a ready-to-use container.

### LEARN Part C: Deploy a model on OpenShift (if not already done)

If you don't have a model serving yet, follow the Red Hat article:
- [Deploy and benchmark vLLM with GuideLLM on Kubernetes (Red Hat, Dec 2025)](https://developers.redhat.com/articles/2025/12/24/how-deploy-and-benchmark-vllm-guidellm-kubernetes)

This covers:
- Creating project, service account, HF secret
- PVC for model weights
- vLLM Deployment with GPU
- Exposing via Service + Route

**If you already have a model running** (like Qwen3-0.6B on your cluster), skip this part and go straight to running the benchmark.

### LEARN Part D: Run your first GuideLLM benchmark

**Option A -- As an OpenShift Job (recommended for learning the production workflow):**

- Create a PVC for benchmark results
- Create a GuideLLM Job manifest pointing at your model's internal service URL
- Apply the Job, wait for completion
- Retrieve results via helper pod + `oc cp`

**Option B -- Locally from your laptop (quick and simple):**

```bash
# If your model service is exposed via a Route:
guidellm benchmark \
  --target "http://<your-route-url>" \
  --model "Qwen/Qwen3-0.6B" \
  --profile sweep \
  --max-seconds 30 \
  --data "prompt_tokens=256,output_tokens=128"
```

Note: Running locally goes through the external Route, which adds network latency. OpenShift Job gives more accurate numbers because it uses the internal service URL.

### EXPERIMENT (go beyond the article)

- [ ] Run the same benchmark 3 times -- how consistent are the numbers?
- [ ] Try `--profile synchronous` vs `--profile sweep` -- what's the difference in output?
- [ ] Open the HTML report -- explore every chart and table, understand what each shows
- [ ] Use `guidellm benchmark from-file ./benchmarks.json` locally to re-display results
- [ ] Compare running from your laptop (via Route) vs as a Job (via internal Service) -- how much does network latency affect results?

### BLOG: "My First LLM Benchmark on OpenShift -- What I Learned"

Write about:
- Your cluster setup (what GPU, what OpenShift version, what model)
- Two ways to run GuideLLM (local vs OpenShift Job) and which is better
- What went smoothly, what broke (PVC storage class? GPU scheduling? Image pull?)
- The actual numbers you got -- screenshot the HTML report
- What surprised you about the results
- Link to the Red Hat article as reference (don't rewrite it)

---

## Step 2: Understand Every Metric GuideLLM Produces

### LEARN (study the output)

Read the docs:
- [Metrics Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/metrics.md)
- [Outputs Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/outputs.md)

Study the 5 output tables from your Step 1 results:
1. Run Summary Info
2. Text Metrics Statistics
3. Request Token Statistics
4. Request Latency Statistics (Request Latency, TTFT, ITL, TPOT)
5. Server Throughput Statistics (Input Tok/s, Output Tok/s, Total Tok/s)

Understand the statistical summaries: mean, median, p50, p90, p95, p99, variance, min, max.

### EXPERIMENT

- [ ] Look at your JSON output -- find the `metrics` section for a single benchmark
- [ ] Compare mean vs p99 for TTFT -- how big is the gap? What does that tell you?
- [ ] Compare mean vs p99 for ITL -- same question
- [ ] If you ran multiple concurrency levels (1, 2, 4), how do metrics change as concurrency increases?
- [ ] Load the JSON into Python and extract specific values:

```python
import json

with open("benchmarks.json") as f:
    data = json.load(f)

for b in data["benchmarks"]:
    print(f"Profile: {b['args']}")
    metrics = b["metrics"]
    print(f"  TTFT mean: {metrics['time_to_first_token_ms']['mean']:.1f}ms")
    print(f"  TTFT p99:  {metrics['time_to_first_token_ms']['p99']:.1f}ms")
    print(f"  ITL mean:  {metrics['inter_token_latency_ms']['mean']:.1f}ms")
    print(f"  Output tok/s: {metrics['output_tokens_per_second']['mean']:.1f}")
```

(Note: verify the exact JSON key names from your own output -- they may differ slightly)

### BLOG: "Decoding LLM Benchmark Metrics -- What TTFT, ITL, and TPOT Actually Mean"

Write about:
- Explain each metric in plain language with your own numbers as examples
- Show the gap between mean and p99 -- why p99 matters more for production
- How metrics change with concurrency -- include a simple table/chart from your data
- Which metrics matter for which use case (chat = TTFT+ITL, batch = throughput)

---

## Step 3: Master Load Profiles -- All 6 Traffic Patterns

### LEARN

Read:
- [README - Load Patterns](https://github.com/vllm-project/guidellm/blob/main/README.md#load-patterns)
- [README - Benchmark Controls](https://github.com/vllm-project/guidellm/blob/main/README.md#benchmark-controls)

The 6 profiles:
1. **synchronous** -- one request at a time (baseline)
2. **concurrent** -- fixed number of parallel requests
3. **throughput** -- max capacity (firehose)
4. **constant** -- fixed requests/sec
5. **poisson** -- randomized requests/sec (realistic traffic)
6. **sweep** -- automatically explores increasing rates

### EXPERIMENT (the real learning)

Run all 6 profiles as separate OpenShift Jobs against the same model. Keep prompt/output tokens constant (e.g., prompt_tokens=256, output_tokens=128).

- [ ] Synchronous: `--profile synchronous --max-seconds 60`
- [ ] Concurrent (4): `--profile concurrent --rate 4 --max-seconds 60`
- [ ] Concurrent (16): `--profile concurrent --rate 16 --max-seconds 60`
- [ ] Throughput: `--profile throughput --max-seconds 60`
- [ ] Constant (5 req/s): `--profile constant --rate 5 --max-seconds 60`
- [ ] Poisson (5 req/s): `--profile poisson --rate 5 --max-seconds 60`
- [ ] Sweep: `--profile sweep --max-seconds 30`

Collect all results and build a comparison table:

| Profile | Rate | Req/s | TTFT p50 | TTFT p99 | ITL p50 | Output Tok/s |
|---------|------|-------|----------|----------|---------|-------------|
| synchronous | - | ? | ? | ? | ? | ? |
| concurrent | 4 | ? | ? | ? | ? | ? |
| concurrent | 16 | ? | ? | ? | ? | ? |
| throughput | - | ? | ? | ? | ? | ? |
| constant | 5 | ? | ? | ? | ? | ? |
| poisson | 5 | ? | ? | ? | ? | ? |
| sweep | - | ? | ? | ? | ? | ? |

- [ ] At what concurrency does latency start degrading?
- [ ] How does constant vs poisson compare at the same rate?
- [ ] What does the sweep automatically discover that you wouldn't know from a single run?
- [ ] Try adding `--warmup 0.1 --cooldown 0.1` -- do results change?

### BLOG: "6 Load Profiles, One Model: What Each Traffic Pattern Reveals About Your LLM"

Write about:
- Your comparison table with real numbers (this doesn't exist in any Red Hat article)
- Which profile to use for which purpose -- your practical recommendation
- The "aha moment" when you see latency degrade at higher concurrency
- Constant vs Poisson -- does randomness matter?
- How sweep saves time vs running individual profiles

---

## Step 4: Datasets -- Synthetic vs Real Data

### LEARN

Read:
- [Datasets Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/datasets.md)

Dataset types GuideLLM supports:
- Synthetic (built-in, works everywhere including air-gapped)
- HuggingFace datasets
- File-based (.txt, .csv, .jsonl, .json, .parquet)
- In-memory (Python API only)

### EXPERIMENT

- [ ] Run with **different synthetic configs** and compare:
  - `--data "prompt_tokens=128,output_tokens=64"` (short)
  - `--data "prompt_tokens=1000,output_tokens=500"` (medium)
  - `--data "prompt_tokens=4000,output_tokens=1000"` (long)
  - How does prompt length affect TTFT? How does output length affect ITL?

- [ ] Run with a **HuggingFace dataset** (create a Job with HF_TOKEN):
  - `--data "garage-bAInd/Open-Platypus"`
  - How do results differ from synthetic data with similar token counts?

- [ ] Create a **custom .jsonl file**, upload to a PVC, mount in the Job:
  ```json
  {"prompt": "Summarize the key benefits of Kubernetes for enterprise workloads."}
  {"prompt": "Write a Python function that implements binary search."}
  {"prompt": "Explain the differences between TCP and UDP protocols."}
  ```
  - How do results differ from synthetic?

- [ ] Try **preprocessing** locally:
  ```bash
  guidellm preprocess dataset "input.jsonl" "processed.jsonl" \
    --processor "meta-llama/Llama-3.1-8B-Instruct" \
    --config "prompt_tokens=512,output_tokens=256"
  ```

### BLOG: "Synthetic vs Real Data: How Dataset Choice Affects LLM Benchmark Results"

Write about:
- Side-by-side results: synthetic vs HF dataset vs custom prompts
- How prompt length impacts TTFT (with actual numbers)
- How output length impacts ITL and throughput
- When to use synthetic (quick tests) vs real data (production validation)
- Gotchas: tokenizer requirements, PVC mounting for datasets

---

## Step 5: SLOs -- Setting and Validating Performance Targets

### LEARN

Read:
- [Service Level Objectives Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/service_level_objectives.md)

Key SLO examples from the docs:

| Use Case | Metric | Target |
|----------|--------|--------|
| Chat app | TTFT | <= 200ms at p99 |
| Chat app | ITL | <= 50ms at p99 |
| RAG system | Request Latency | <= 3s at p99 |
| Code completion | Request Latency | <= 2s at p99 |
| Batch summarization | Throughput | >= 100 req/s |

### EXPERIMENT

- [ ] Pick a use case (e.g., "chat app") and define your SLOs
- [ ] Run a benchmark at different concurrency levels
- [ ] Write a Python script to automatically check pass/fail:

```python
import json

SLOS = {
    "ttft_p99_ms": 200,
    "itl_p99_ms": 50,
}

with open("benchmarks.json") as f:
    data = json.load(f)

for b in data["benchmarks"]:
    ttft = b["metrics"]["time_to_first_token_ms"]["p99"]
    itl = b["metrics"]["inter_token_latency_ms"]["p99"]

    print(f"TTFT p99: {ttft:.0f}ms -- {'PASS' if ttft <= SLOS['ttft_p99_ms'] else 'FAIL'}")
    print(f"ITL p99:  {itl:.0f}ms -- {'PASS' if itl <= SLOS['itl_p99_ms'] else 'FAIL'}")
```

- [ ] At what concurrency do your SLOs start failing? That's your capacity limit.
- [ ] Try with different models or quantization levels -- how does that change the SLO boundary?

### BLOG: "Finding Your LLM's Capacity Limit: SLO Validation with GuideLLM on OpenShift"

Write about:
- Your chosen SLOs and why
- The concurrency level where SLOs break (with real numbers)
- Your pass/fail script (useful for others to copy)
- How model size/quantization affects where the SLO boundary sits
- Practical recommendation: "for a chat app on A100 with Llama-8B, you can handle X concurrent users"

---

## Step 6: Over-Saturation Detection

### LEARN

Read the Red Hat 3-part series (these are excellent, study them):
- [Part 1: Reduce benchmarking costs with OSD](https://developers.redhat.com/articles/2025/11/18/reduce-llm-benchmarking-costs-oversaturation-detection)
- [Part 2: Evaluation metrics for OSD](https://developers.redhat.com/articles/2025/11/20/oversaturation-detection-evaluation-metrics)
- [Part 3: Building an OSD with iterative error analysis](https://developers.redhat.com/articles/2025/11/24/building-oversaturation-detector-iterative-error-analysis)

Also read:
- [Over-Saturation Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/over_saturation_stopping.md)

### EXPERIMENT

- [ ] Run a sweep WITHOUT OSD: `--profile sweep --max-seconds 120`
  - How long does it run? Does it waste time on saturated rates?

- [ ] Run a sweep WITH OSD: `--profile sweep --max-seconds 120 --detect-saturation`
  - How much earlier does it stop? How much GPU time did you save?

- [ ] Try advanced tuning:
  ```
  --over-saturation '{"enabled": true, "min_seconds": 30, "moe_threshold": 1.5}'
  ```
  vs
  ```
  --over-saturation '{"enabled": true, "min_seconds": 30, "moe_threshold": 3.0}'
  ```
  - Does a lower threshold catch saturation earlier? Is it too aggressive (false positives)?

- [ ] Check the OSD metadata in your JSON output:
  - `is_over_saturated`, `concurrent_slope`, `ttft_slope`, `ttft_violations`

### BLOG: "How Over-Saturation Detection Saved Me X GPU-Hours on OpenShift"

Write about:
- Concrete comparison: with vs without OSD (time saved, cost saved)
- What the saturation point looked like in your metrics
- Tuning the sensitivity threshold -- your findings
- Link to the Red Hat 3-part series as background reading (don't rewrite it)

---

## Step 7: Red Hat AI Inference Server vs Community vLLM

### LEARN

Read:
- [Red Hat AI Inference Server](https://developers.redhat.com/products/red-hat-ai/inference-server)
- [Introducing Red Hat AI Inference Server](https://www.redhat.com/en/blog/red-hat-ai-inference-server-technical-deep-dive)
- [Deploy LLM inference on OpenShift AI](https://developers.redhat.com/articles/2025/11/03/deploy-llm-inference-service-openshift-ai)
- [Optimize and deploy LLMs with OpenShift AI](https://developers.redhat.com/articles/2025/10/06/optimize-and-deploy-llms-production-openshift-ai)

### EXPERIMENT (this comparison doesn't exist in any Red Hat article!)

- [ ] Deploy the same model on both:
  - Community vLLM: `vllm/vllm-openai:v0.11.2`
  - Red Hat AI Inference Server: `registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.1-xxxx`

- [ ] Run identical GuideLLM benchmarks against both (same profile, same data, same duration)

- [ ] Compare side-by-side:

| Metric | Community vLLM | Red Hat AIIS | Difference |
|--------|---------------|--------------|------------|
| TTFT p50 | ? | ? | ? |
| TTFT p99 | ? | ? | ? |
| ITL p50 | ? | ? | ? |
| Output Tok/s | ? | ? | ? |
| Error rate | ? | ? | ? |

- [ ] Try the MLOps workflow: quantize a model with LLM Compressor, deploy on RHAIIS, benchmark
- [ ] Deploy via OpenShift AI (KServe InferenceService) instead of raw Deployment -- benchmark difference?

### BLOG: "Community vLLM vs Red Hat AI Inference Server: A Benchmark Comparison on OpenShift"

Write about:
- Your comparison table with real numbers (NOBODY has published this)
- Are there actual performance differences or is it the same vLLM under the hood?
- When to use community (dev/test) vs RHAIIS (production support)
- The OpenShift AI deployment experience vs raw Deployment
- This is genuinely unique content that doesn't exist anywhere

---

## Step 8: Multimodal Benchmarking on OpenShift

### LEARN

Read:
- [Multimodal Image Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/multimodal/image.md)
- [Multimodal Audio Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/multimodal/audio.md)
- [Multimodal Video Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/multimodal/video.md)
- [Speech-to-text with Whisper on RHAIIS](https://developers.redhat.com/articles/2025/06/10/speech-text-whisper-and-red-hat-ai-inference-server)

### EXPERIMENT

- [ ] Deploy a vision-language model on OpenShift (e.g., LLaVA via vLLM)
- [ ] Benchmark with image+text inputs
- [ ] Check multimodal-specific metrics: image sizes, audio lengths
- [ ] Compare text-only vs multimodal performance on the same model
- [ ] Deploy Whisper on RHAIIS and benchmark audio transcription

### BLOG: "Benchmarking Vision and Audio Models on OpenShift with GuideLLM"

Write about:
- How to deploy multimodal models on OpenShift
- Performance differences: text-only vs image+text
- Audio transcription benchmarking (no one has covered this with GuideLLM on OpenShift)
- Practical tips for multimodal dataset preparation

---

## Step 9: Air-Gapped Benchmarking

### LEARN

Follow the Red Hat article hands-on:
- [Benchmarking with GuideLLM in air-gapped OpenShift clusters (Red Hat, Sep 2025)](https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters)

This covers: oc-mirror, ICSP, PVCs for models/tokenizers, GuideLLM Job in disconnected environment.

### EXPERIMENT

- [ ] Simulate a disconnected environment (block external registry access)
- [ ] Mirror all required images (RHAIIS, GuideLLM, UBI)
- [ ] Copy model weights + tokenizer to PVCs manually
- [ ] Run benchmarks -- verify synthetic data works without internet (corpus is built-in)
- [ ] Try different models -- what's the workflow for swapping models in air-gapped?

### BLOG: "My Experience Running LLM Benchmarks in a Disconnected OpenShift Cluster"

Write about:
- What went wrong during the disconnected setup (there will be gotchas!)
- The full image list you needed to mirror
- Time/effort estimate for the air-gapped workflow
- Tips you wish you knew before starting
- Link to the Red Hat article as the base guide

---

## Step 10: Distributed Inference with llm-d

### LEARN

Read:
- [llm-d QuickStart](https://llm-d.ai/docs/guide/Installation/quickstart)
- [llm-d Benchmark Tools](https://llm-d.ai/docs/architecture/Components/benchmark)
- [Getting started with llm-d (Red Hat)](https://developers.redhat.com/articles/2025/08/19/getting-started-llm-d-distributed-ai-inference)
- [Accelerate multi-turn workloads with llm-d (Red Hat)](https://developers.redhat.com/articles/2026/01/13/accelerate-multi-turn-workloads-llm-d)
- [llm-d Well-Lit Paths](https://llm-d.ai/docs/guide)

### EXPERIMENT

- [ ] Deploy llm-d on OpenShift following the QuickStart
- [ ] Run GuideLLM through llm-d-benchmark automation (standup/run/teardown)
- [ ] Compare naive vLLM multi-replica vs llm-d intelligent routing:
  - KV cache hit rates
  - P95/P99 TTFT
  - GPU utilization
- [ ] Try different well-lit paths: inference scheduling, P/D disaggregation
- [ ] Monitor with Grafana

### BLOG: "From Single vLLM to Distributed llm-d: Scaling LLM Inference on OpenShift"

Write about:
- The deployment experience (llm-d is complex -- document the learning curve)
- Performance comparison: standalone vs distributed (with real numbers)
- Which well-lit path made the biggest difference for your workload
- When llm-d is worth the complexity vs when standalone vLLM is enough

---

## Step 11: CI/CD Benchmarking with Tekton on OpenShift

### LEARN

- [Outputs Guide](https://github.com/vllm-project/guidellm/blob/main/docs/guides/outputs.md)
- [Autoscaling vLLM with OpenShift AI](https://developers.redhat.com/articles/2025/11/26/autoscaling-vllm-openshift-ai-model-serving)
- OpenShift Pipelines (Tekton) documentation

### EXPERIMENT

- [ ] Create a Tekton Task that runs GuideLLM as a Job
- [ ] Create a Tekton Pipeline: deploy model -> run benchmark -> check SLOs -> gate promotion
- [ ] Store baseline results in a ConfigMap
- [ ] Add `--output-extras '{"tag": "v1.2", "gpu": "A100", "cluster": "prod"}'` for tracking
- [ ] Test autoscaling: run GuideLLM at increasing load, verify HPA kicks in

### BLOG: "Automated LLM Performance Gates with GuideLLM and Tekton on OpenShift"

Write about:
- Your Tekton Pipeline definition (shareable YAML)
- How to auto-fail deployments if benchmarks degrade
- Tracking benchmark regression across releases with output-extras
- This is production-grade automation nobody has published

---

## Summary

| Step | Primary Learning | Blog Angle | Unique? |
|------|-----------------|------------|---------|
| 1 | First benchmark on OpenShift | Setup experience + gotchas | References Red Hat article |
| 2 | Metrics deep dive | What the numbers actually mean | Your own data interpretation |
| 3 | All 6 load profiles | Side-by-side comparison table | **YES** -- doesn't exist anywhere |
| 4 | Dataset types | Synthetic vs real data impact | **YES** -- not covered |
| 5 | SLOs + validation | Finding capacity limits | **YES** -- pass/fail scripts |
| 6 | Over-saturation detection | GPU time/cost savings | References Red Hat OSD series |
| 7 | RHAIIS vs community vLLM | Performance comparison | **YES** -- nobody has published this |
| 8 | Multimodal benchmarking | Vision + audio on OpenShift | **YES** -- not covered |
| 9 | Air-gapped workflow | Disconnected gotchas | References Red Hat article |
| 10 | llm-d distributed inference | Standalone vs distributed perf | Extends Red Hat llm-d articles |
| 11 | CI/CD automation | Tekton Pipeline for benchmarks | **YES** -- not covered |

**Learning order:** Sequential (1 through 11). Each step builds on the previous.

- **Steps 1-2:** Get running and understand output
- **Steps 3-5:** Core experimentation (most unique blog content)
- **Steps 6-8:** Advanced features (mix of learning + unique content)
- **Steps 9-11:** Enterprise production (complex, big payoff)

---

## Source Verification

| Source | URL |
|--------|-----|
| GuideLLM Repository | https://github.com/vllm-project/guidellm |
| GuideLLM Docs | https://github.com/vllm-project/guidellm/tree/main/docs |
| Red Hat: GuideLLM Intro (Jun 2025) | https://developers.redhat.com/articles/2025/06/20/guidellm-evaluate-llm-deployments-real-world-inference |
| Red Hat: Air-Gapped (Sep 2025) | https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters |
| Red Hat: OSD 3-part (Nov 2025) | https://developers.redhat.com/articles/2025/11/18/reduce-llm-benchmarking-costs-oversaturation-detection |
| Red Hat: Deploy+Benchmark (Dec 2025) | https://developers.redhat.com/articles/2025/12/24/how-deploy-and-benchmark-vllm-guidellm-kubernetes |
| Red Hat: llm-d multi-turn (Jan 2026) | https://developers.redhat.com/articles/2026/01/13/accelerate-multi-turn-workloads-llm-d |
| Red Hat AI Inference Server | https://developers.redhat.com/products/red-hat-ai/inference-server |
| llm-d Docs | https://llm-d.ai/docs |
