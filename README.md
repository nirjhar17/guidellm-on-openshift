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

### Planned

2. Understand Every Metric GuideLLM Produces (TTFT, ITL, TPOT, percentiles)
3. Master Load Profiles -- All 6 Traffic Patterns
4. Datasets -- Synthetic vs Real Data
5. SLOs -- Setting and Validating Performance Targets
6. Over-Saturation Detection
7. Red Hat AI Inference Server vs Community vLLM
8. Multimodal Benchmarking on OpenShift
9. Air-Gapped Benchmarking
10. Distributed Inference with llm-d
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

## References

- [GuideLLM Repository](https://github.com/vllm-project/guidellm)
- [Red Hat: Deploy and benchmark vLLM with GuideLLM (Dec 2025)](https://developers.redhat.com/articles/2025/12/24/how-deploy-and-benchmark-vllm-guidellm-kubernetes)
- [Red Hat: GuideLLM -- Evaluate LLM deployments (Jun 2025)](https://developers.redhat.com/articles/2025/06/20/guidellm-evaluate-llm-deployments-real-world-inference)
- [Red Hat: Air-Gapped GuideLLM (Sep 2025)](https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters)
- [Red Hat: Over-Saturation Detection (Nov 2025)](https://developers.redhat.com/articles/2025/11/18/reduce-llm-benchmarking-costs-oversaturation-detection)
- [llm-d Documentation](https://llm-d.ai/docs)
