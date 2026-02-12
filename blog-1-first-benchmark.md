Exploring GuideLLM: Benchmarking a Live LLM Deployment on OpenShift


As part of evaluating inference performance for LLM deployments on OpenShift, I recently started exploring GuideLLM, an open-source benchmarking tool that lives under the vLLM project. I had a model already running on my cluster and wanted to understand its actual performance characteristics under different load conditions.

In this post, I will walk through the setup process, the issues I ran into along the way, and how to interpret the benchmark results.


What is GuideLLM

GuideLLM is an SLO-aware benchmarking and evaluation tool designed specifically for LLM inference endpoints. It simulates realistic user traffic against any OpenAI-compatible API, whether that is vLLM, Red Hat AI Inference Server, or TGI, and measures the key metrics that define the end-user experience: time to first token, inter-token latency, throughput, and request latency.

What sets it apart from general-purpose load testing tools is that it understands the nature of LLM workloads. It knows about tokens, streaming responses, and the specific performance patterns that matter when serving generative AI models.

The project was originally created by Neuralmagic (now part of Red Hat) and is hosted at github.com/vllm-project/guidellm.


My Setup

Here is what I was working with:

Cluster: OpenShift 4.17 on AWS (ROSA)
GPU: NVIDIA Tesla T4 (single GPU node)
Model: Qwen/Qwen3-0.6B (a small but capable model, perfect for experimentation)
Model Serving: vLLM deployed via KServe InferenceService
Namespace: my-first-model (for the model), guidellm-lab (for benchmarking)

One quick note on KServe and vLLM for anyone confused about the relationship. KServe is the Kubernetes-native serving platform that handles the infrastructure (autoscaling, traffic routing, canary deployments, health checks). vLLM is the inference engine that actually runs the model and generates tokens. When you deploy a model through KServe on OpenShift AI, KServe creates the pods and services, and inside those pods vLLM is doing the actual work. So my target was a vLLM engine, managed by KServe.

I also had llm-d installed on my cluster, which is the distributed inference stack that adds intelligent routing on top of vLLM. But for this first benchmark, I went directly to the model service to get a clean baseline.


Two Ways to Run GuideLLM

GuideLLM gives you two options.

Option one is installing it locally on your laptop with pip install guidellm and running it against an external Route. This is fine for quick tests, but the numbers include network latency from your machine to the cluster, which pollutes the results.

Option two is running it as a Kubernetes Job directly inside the cluster. The Job pod talks to the model service over the internal cluster network, so you get pure inference performance numbers without any external network noise. This is the approach I took.

The container image is ghcr.io/vllm-project/guidellm:v0.5.0 and it comes ready to use. No installation inside the cluster required.


Setting Up the Benchmark

I created a dedicated namespace for my benchmarking work.

oc new-project guidellm-lab

Then I created a PersistentVolumeClaim to store the benchmark results. This is important because when a Job completes, the pod gets terminated, and you lose everything in the container filesystem. The PVC survives after the pod is gone.

The PVC was straightforward, 5Gi with ReadWriteOnce access mode.

Next came the Job manifest. This is where GuideLLM gets configured. Let me walk through what each argument does.

The target is the internal service URL of my model. Since I deployed through KServe, the service name follows KServe's naming convention: qwen3-0-6b-kserve-workload-svc.my-first-model.svc.cluster.local on port 8000.

The model argument tells GuideLLM which model to expect. This matters because GuideLLM downloads the tokenizer for that specific model to accurately count tokens. More on that in a moment.

The data argument is where you define what kind of requests to send. I used synthetic data with prompt_tokens set to 256 and output_tokens set to 128. This means GuideLLM generates synthetic prompts that are exactly 256 tokens long and asks the model to produce 128 tokens in response.

The profile was set to sweep. This is the most informative profile for a first run because it automatically tests increasing request rates, from a gentle trickle up to maximum throughput, so you can see exactly where the model starts struggling.

The max-seconds was 30, which means each rate level in the sweep runs for 30 seconds before moving to the next one.

Finally, the outputs argument specifies that I want both a JSON file (for programmatic analysis) and an HTML file (for the visual report).


The Gotchas -- What Broke and How I Fixed It

This is the part no tutorial tells you about.

My first attempt failed immediately. The pod crashed with a "server disconnected without sending a response" error. The problem was that I used http in my target URL. KServe enforces TLS by default, so the model was only listening on HTTPS. Changing the URL to https fixed the connection.

But then the second attempt failed with a different error: "unexpected keyword argument verify_ssl." The issue here was that KServe uses self-signed certificates for its internal TLS, and GuideLLM's HTTP client was rejecting the certificate. I needed to disable SSL verification, but I had the wrong parameter name. The correct argument is backend-kwargs with the value {"verify": false}, not verify_ssl. This tells the underlying HTTP client to skip certificate validation, which is fine for internal cluster traffic.

Third attempt worked perfectly.

If you are deploying with KServe, remember these two things: use HTTPS for the target URL, and pass {"verify": false} in backend-kwargs to handle self-signed certificates.


The Tokenizer Question

When GuideLLM starts, one of the first things it does is download the tokenizer for your model. You will see a log line that says "Processor resolved -- Using model Qwen/Qwen3-0.6B as processor."

This confused me at first. Why does a benchmarking tool need a tokenizer?

The answer is that GuideLLM needs to generate synthetic prompts of exactly 256 tokens. But "256 tokens" means something different for every model. Each model has its own vocabulary and its own way of splitting text into tokens. The word "benchmarking" might be one token for one model and two tokens for another.

So GuideLLM downloads the small tokenizer file (not the model weights, just the vocabulary mapping), uses it to precisely measure token counts, and generates prompts that are exactly 256 tokens in the target model's language.

This means GuideLLM needs internet access to download the tokenizer. In a disconnected or air-gapped environment, this step would fail. You would need to pre-download the tokenizer and make it available locally. Red Hat has a whole article on running GuideLLM in air-gapped clusters that covers this workflow.

The synthetic prompt corpus itself is built into GuideLLM, so that part works without internet. It is only the tokenizer that requires a download.


Reading the Results

After the Job completed, I needed to get the files off the PVC. Since the Job pod was already terminated, I spun up a small helper pod (a UBI9 image running sleep infinity) with the same PVC mounted, then used oc cp to copy the files to my laptop.

GuideLLM produces two output files. The JSON file contains every single data point, all the raw metrics for programmatic analysis. The HTML file is a beautiful interactive report that you can open in a browser.

Let me walk through what the HTML report shows.


The Report: Header and Workload Details

At the top, the report confirms the model (Qwen/Qwen3-0.6B), the timestamp (February 12, 2026), and shows three cards.

The Prompt card shows a sample of the raw request that was sent, along with the mean prompt length of 264 tokens and a distribution chart. The distribution shows a single spike at 264, which makes sense because all synthetic prompts were generated to be the same size.

The Server card shows the target URL, that 10 benchmarks were run (the 10 rate levels in the sweep), and the rate type as synchronous for the baseline measurement.

The Generated card shows a sample response from the model. Mine started with a think tag, which is Qwen3's chain-of-thought reasoning. The mean generated length was 128 tokens, exactly as configured. The distribution chart again shows a single spike.


The Report: Interactive Sweep Slider

This is the most powerful part of the report.

At the bottom of the metrics section, there is a slider that goes from 0.37 requests per second (the lightest load) to 8.53 requests per second (maximum throughput). As you drag the slider, four line charts update in real time to show how each metric behaves at that load level.

Time to First Token starts at 34.59 milliseconds at the lightest load. This is how long a user waits before they see the first word appear. It stays beautifully flat around 35ms all the way up to about 5 requests per second. Then, past 6 requests per second, it suddenly shoots up to over 5000 milliseconds. That is a 150x degradation. A user who was getting instant responses would now be staring at a blank screen for 5 seconds.

Inter-Token Latency starts at 20.06 milliseconds. This is the gap between each subsequent token, which determines how fast the text "streams" to the user. It holds steady around 20ms up to about 5 requests per second, then spikes to around 147 milliseconds at max load. The stream that was feeling fast and natural would start feeling choppy and sluggish.

Time Per Request starts at 2.58 seconds for a complete 128-token response. This climbs gradually and then rockets up to 24 seconds at max load. So a response that took 2.5 seconds would now take nearly half a minute.

Throughput starts at 50.87 tokens per second for a single request. This is the one metric that actually goes up with load, because the GPU is processing more requests in parallel. At max load, total throughput reaches around 2000 tokens per second. But this is misleading. Total throughput is up, but each individual user is getting a slower experience. The GPU is busier, but from each user's perspective, things are worse.


The Report: Detailed Metrics Charts

Below the slider, the report has four detailed charts that show the same metrics plotted against requests per second, with multiple percentile lines (p50, p90, p95, p99) and the mean.

For Time to First Token at the lowest load (0.37 rps), the percentiles are remarkably tight. The p50 is 31.98 milliseconds, the p90 is 34.59 milliseconds. The p95 and p99 jump to 612 milliseconds, which tells me there were a couple of outlier requests that were slow, probably due to some initial warmup or garbage collection. The mean gets pulled up to 80.42 milliseconds by those outliers.

For Inter-Token Latency at low load, the percentiles are extremely tight: p50 at 19.74ms, p90 at 20.06ms, p95 and p99 both at 20.29ms. The mean is 19.82ms. This tells me that token generation is incredibly consistent at low load. Almost no variance at all.

For Time Per Request, the p50 is 2.54 seconds and p90 is 2.58 seconds at low load. Very consistent.

For Throughput, the chart is interesting because it shows the percentile lines fanning apart at higher loads. The mean line (solid blue) climbs steadily as the GPU handles more parallel work. But the p50 line (what a typical user experiences) flattens out much earlier. So while the server is doing more total work, each individual request is not getting faster.

The key insight from all four charts is that there is a clear inflection point around 5-6 requests per second. Below that, everything is stable and fast. Above that, metrics degrade exponentially. This is the "hockey stick" curve that every LLM operator needs to find for their deployment.


The 30-Second Summary

If someone asks me "how does Qwen 0.6B perform on a Tesla T4?", here is what I now know.

The sweet spot is 1 to 5 requests per second. In that range, users get their first token in about 35 milliseconds, each subsequent token streams at 20-millisecond intervals, and a complete 128-token response takes about 2.5 seconds. The experience feels instant and fluid.

Beyond 6 requests per second, things fall off a cliff. Time to first token jumps from milliseconds to seconds. Token streaming gets choppy. Total response time balloons to 30 seconds.

For a production deployment, I would set a capacity limit at 5 concurrent users and set up autoscaling to add replicas before hitting that threshold. The SLOs I would target are TTFT under 200 milliseconds at p99 and ITL under 50 milliseconds at p99. Based on my numbers, that gives comfortable headroom.


What I Learned

Running a benchmark tool is easy. Understanding what the numbers mean is where the real learning happens.

The biggest surprise was how sharply things degrade. There is no gradual slowdown. The model goes from perfectly fine to completely overwhelmed in the span of one or two additional requests per second. This is why benchmarking matters. You cannot guess your capacity limit. You have to measure it.

The second surprise was how consistent the model is at low load. The gap between p50 and p99 for inter-token latency was less than 1 millisecond. That kind of consistency is great for user experience, and it only falls apart when you push past the saturation point.

The third lesson was operational. Running GuideLLM as an OpenShift Job is the right approach for accurate numbers, but you need to plan for result retrieval. The PVC plus helper pod pattern works, but it is a few extra steps. In a future post, I will look at automating this workflow.


What is Next

This was just the first step. I ran one profile (sweep) with one data configuration (synthetic 256/128 tokens) against one model. There is a lot more to explore.

Next, I want to dig deeper into every metric GuideLLM produces, understand the gap between mean and p99, and figure out which metrics matter most for different use cases. That will be the second post in this series.

I also want to try all six load profiles that GuideLLM supports, compare synthetic data versus real datasets, set up formal SLO validation, and eventually benchmark llm-d's distributed inference against standalone vLLM. But one step at a time.

If you want to try this yourself, GuideLLM is at github.com/vllm-project/guidellm and the Red Hat article that got me started is at developers.redhat.com/articles/2025/12/24/how-deploy-and-benchmark-vllm-guidellm-kubernetes.


References

GuideLLM Repository: https://github.com/vllm-project/guidellm
Red Hat: Deploy and benchmark vLLM with GuideLLM on Kubernetes (Dec 2025): https://developers.redhat.com/articles/2025/12/24/how-deploy-and-benchmark-vllm-guidellm-kubernetes
Red Hat: GuideLLM -- Evaluate LLM deployments (Jun 2025): https://developers.redhat.com/articles/2025/06/20/guidellm-evaluate-llm-deployments-real-world-inference
Red Hat: Benchmarking GuideLLM in air-gapped OpenShift clusters (Sep 2025): https://developers.redhat.com/articles/2025/09/15/benchmarking-guidellm-air-gapped-openshift-clusters
