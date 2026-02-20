# GuideLLM on OpenShift -- Learning Journey & Blog Series

A hands-on learning path for [GuideLLM](https://github.com/vllm-project/guidellm), the LLM benchmarking tool from the vLLM project. Everything runs on OpenShift with real GPU hardware.

## What is This Repo?

This repo documents my journey learning GuideLLM by running it against live LLM deployments on OpenShift. Each step involves hands-on experimentation, and I write about what I learn along the way.

## My Environment

| Component | Details |
|-----------|---------|
| **Cluster** | OpenShift 4.17 (ROSA on AWS) |
| **GPU** | NVIDIA Tesla T4 |
| **Model** | Qwen/Qwen3-0.6B |
| **Model Serving** | vLLM via KServe InferenceService |
| **Benchmarking** | GuideLLM v0.5.0 (as OpenShift Job) |
| **Distributed Inference** | llm-d |

## Blog Series

### Published

1. **[My First LLM Benchmark on OpenShift](blog-1-first-benchmark.md)** -- Setting up GuideLLM, running it as an OpenShift Job against Qwen3-0.6B via KServe, fixing TLS gotchas, and interpreting the HTML report. Includes real benchmark data from a Tesla T4.

2. **[Benchmarking P/D Disaggregation on OpenShift AI with GuideLLM](pd-disaggregation-lab/blog-benchmarking-llm-d-guidellm.md)** -- Deploying Prefill/Decode disaggregation with llm-d, fixing EPP intelligent routing (4 bugs found on RHOAI 3.0-3.2), proving prefix cache with inference-perf, and running all 7 GuideLLM load profiles. Full manifests and results included.

### Planned

3. Understand Every Metric GuideLLM Produces (TTFT, ITL, TPOT, percentiles)
4. Master Load Profiles -- All 6 Traffic Patterns
5. Datasets -- Synthetic vs Real Data
6. SLOs -- Setting and Validating Performance Targets
7. Over-Saturation Detection
8. Red Hat AI Inference Server vs Community vLLM
9. Multimodal Benchmarking on OpenShift
10. Air-Gapped Benchmarking
11. CI/CD Benchmarking with Tekton on OpenShift

## Key Findings (So Far)

From my first benchmark (Qwen3-0.6B on Tesla T4, sweep profile):

- **Sweet spot:** 1-5 requests per second
- **TTFT at low load:** ~32ms (p50) -- instant first token
- **ITL at low load:** ~20ms -- smooth streaming at 50 tokens/sec
- **Breaking point:** Beyond 6 rps, TTFT jumps from 32ms to 5000ms+ (150x degradation)
- **Capacity recommendation:** Cap at 5 concurrent users, autoscale beyond that

## Learning Plan

The full 11-step learning path is documented in [blog-series-plan.md](blog-series-plan.md), organized into four phases:

- **Phase 1 (Steps 1-2):** Get running and understand output
- **Phase 2 (Steps 3-5):** Core experimentation
- **Phase 3 (Steps 6-8):** Advanced features
- **Phase 4 (Steps 9-11):** Enterprise production workflows

## P/D Disaggregation Lab

The [`pd-disaggregation-lab/`](pd-disaggregation-lab/) directory is a self-contained deployment guide for Prefill/Decode disaggregation with llm-d on OpenShift AI. It includes:

- Full LLMInferenceService manifest with all EPP fixes applied
- EnvoyFilter for RHOAI 3.0-3.2 (required for EPP to work)
- All 7 GuideLLM benchmark profiles + inference-perf shared-prefix workload
- Benchmark results (JSON + HTML) with EPP active
- Step-by-step deploy instructions in its [README](pd-disaggregation-lab/README.md)

## Manifests (Blog 1)

The `manifests/` folder contains ready-to-use OpenShift YAML files to run GuideLLM (for the first blog, single-pod setup):

| File | What it does |
|------|-------------|
| `01-namespace.yaml` | Creates the `guidellm-lab` namespace |
| `02-pvc.yaml` | PersistentVolumeClaim to store benchmark results |
| `03-benchmark-job.yaml` | GuideLLM Job that runs a sweep benchmark against the model |
| `04-pvc-inspector.yaml` | Helper pod to retrieve results from the PVC after the Job completes |

**Quick start:**

```bash
oc apply -f manifests/01-namespace.yaml
oc apply -f manifests/02-pvc.yaml
# Edit 03-benchmark-job.yaml to set your model's service URL
oc apply -f manifests/03-benchmark-job.yaml
# Wait for job to complete, then retrieve results:
oc apply -f manifests/04-pvc-inspector.yaml
oc cp guidellm-lab/pvc-inspector:/mnt/results/benchmarks.html ./benchmarks.html
oc cp guidellm-lab/pvc-inspector:/mnt/results/benchmarks.json ./benchmarks.json
```

**Important:** Update the `--target` URL in `03-benchmark-job.yaml` to match your model's internal service URL. Also note the `--backend-kwargs '{"verify": false}'` flag -- this is needed when KServe uses self-signed TLS certificates.

## References

- [GuideLLM Repository](https://github.com/vllm-project/guidellm)
- [Red Hat: Deploy and benchmark vLLM with GuideLLM (Dec 2025)](https://developers.redhat.com/articles/2025/12/24/how-deploy-and-benchmark-vllm-guidellm-kubernetes)
- [Red Hat: GuideLLM -- Evaluate LLM deployments (Jun 2025)](https://developers.redhat.com/articles/2025/06/20/guidellm-evaluate-llm-deployments-real-world-inference)
- [Red Hat: Air-Gapped GuideLLM (Sep 2025)](https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters)
- [Red Hat: Over-Saturation Detection (Nov 2025)](https://developers.redhat.com/articles/2025/11/18/reduce-llm-benchmarking-costs-oversaturation-detection)
- [llm-d Documentation](https://llm-d.ai/docs)
