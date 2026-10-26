# RTP-LLM native histogram diagnostics

This optional tool collects native RTP-LLM latency histograms for inspection.
It does not emit model-serving-signals contract metrics and does not depend on
the `/worker_status` adapter.

## Collecting upstream latency histograms

[RTP-LLM PR #1545](https://github.com/alibaba/rtp-llm/pull/1545), tested at commit
`adc4feb851`, adds three Python KMonitor
Histogram companions. They preserve the existing Gauge samples, convert milliseconds
to seconds, and send cumulative `_bucket`, `_sum`, and `_count` components over
Flume. **The PR does not add a pull endpoint.**

The optional diagnostic
[`hack/rtp-llm/kmonitor_bridge.py`](https://github.com/kubernetes-sigs/model-serving-signals/blob/main/hack/rtp-llm/kmonitor_bridge.py)
receives these batches and exposes native Prometheus histograms:

| Native family | Preserved timing semantics | Contract mapping |
| --- | --- | --- |
| `py_rtp_framework_rt_seconds` | Existing framework/caller timing boundaries, which vary by operation/protocol. | Unmapped until request-duration coverage is verified. |
| `py_rtp_response_first_token_rt_seconds` | First response, including control chunks/non-streaming observations; not restricted to successful requests. | Unmapped: not verified as successful-request TTFT. |
| `py_rtp_response_iterate_rt_seconds` | Per-iteration observations, with caller-specific normalization. | Unmapped: not per-request mean TPOT. |

The bridge changes the transport representation only. It does not rename these
families to `gen_ai_*`, re-bucket observations, or reinterpret their timing. The
exporter's existing Prometheus scraper can read its endpoint, but the RTP-LLM
contract profile still leaves these signals unmapped. Scrape the native endpoint
separately when examining those distributions.

Run the bridge in the same Pod/network namespace as RTP-LLM, using a Python
environment with the RTP-LLM wheel and its `thrift` dependency installed:

```sh
python3 hack/rtp-llm/kmonitor_bridge.py &
# Before starting the engine:
export HIPPO_SLAVE_IP=127.0.0.1
export HIPPO_ROLE_SHORT_NAME=rtp-llm-validation
export RTP_LLM_LATENCY_HISTOGRAM_ENABLED=1
# Start RTP-LLM, send inference requests, then:
curl -s localhost:19465/metrics
```

`RTP_LLM_LATENCY_HISTOGRAM_ENABLED=0` disables the new observations; the default
is enabled. The existing Python KMonitor reporter additionally requires a nonempty
Hippo role and sends to `HIPPO_SLAVE_IP:4141`. The bridge binds loopback by default.
It is intended for a standalone test deployment: it consumes that Flume destination
and ignores unrelated metrics, so do not substitute it for an existing production
KMonitor collector. It reuses RTP-LLM's generated Flume protocol classes.

Only complete snapshots with the pinned upstream bucket sets are published. Repeated
reports replace the previous snapshot rather than adding counts. Labels, including
`process_instance_id`, remain intact; delayed/regressing snapshots are ignored and
series expire after 30 seconds without fresh source timestamps (`--max-age`). There
is a 4096-series bound. Invalid batches are logged and rejected atomically. A changed
upstream bucket schema requires updating the bridge before use. This utility is not
part of the Go exporter image and is not a general-purpose Flume gateway.

## GPU Kubernetes example

The [diagnostic deployment](https://github.com/kubernetes-sigs/model-serving-signals/blob/main/test/deploy/rtp-llm-histograms.yaml)
runs RTP-LLM and the bridge in the same Pod. It needs one NVIDIA GPU, a compatible
RTP-LLM runtime image with PR #1545's Python changes, and a PVC named
`rtp-llm-model` containing `Qwen3-0.6B/`. The image must provide `python3` on PATH.
The bridge disables NVIDIA device visibility and binds its endpoints to loopback.
There is no contract exporter in this diagnostic deployment.

```sh
export RTP_LLM_IMAGE=your-registry/rtp-llm:histogram-revision
kubectl create namespace rtp-llm-histogram-test
# Provision the model PVC in this namespace before applying the deployment.
kubectl -n rtp-llm-histogram-test create configmap rtp-llm-bridge \
  --from-file=hack/rtp-llm/kmonitor_bridge.py
envsubst '${RTP_LLM_IMAGE}' < test/deploy/rtp-llm-histograms.yaml | \
  kubectl -n rtp-llm-histogram-test apply -f -
kubectl -n rtp-llm-histogram-test rollout status deployment/rtp-llm-histograms --timeout=15m
kubectl -n rtp-llm-histogram-test port-forward deployment/rtp-llm-histograms \
  30000:30000 19465:19465
```

In another terminal:

```sh
curl -s localhost:30000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"default","messages":[{"role":"user","content":"Count to ten."}],"max_tokens":32,"stream":true}'
curl -s localhost:19465/metrics
```

## Validation

- `python3 -m unittest discover -s hack/rtp-llm -v` checks histogram reconstruction,
  repeated reports, process separation, malformed/incomplete batches, expiry, and limits.
- `go test ./pkg/translator` verifies the existing HTTP scraper can read a sanitized
  live capture of all three native histogram families.
- Prior live testing used Qwen3-0.6B on A10/CUDA 13.2, with the source-built wheel
  at `1812ac310f95` and upstream Python changes through `adc4feb851`. Six requests
  produced native counts of 6 framework RT, 6 first-response RT, and 72 iterate RT.
  Repeated reports retained identical totals. That batch reached one frontend;
  multi-process isolation is unit-tested.
