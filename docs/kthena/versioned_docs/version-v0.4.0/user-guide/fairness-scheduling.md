# Fairness Scheduling

When many users hit the same model at once, "first come, first served" often rewards the noisiest client.  
Kthena Router fairness scheduling fixes that by giving quicker turns to users with lower recent usage.

Think of it as a fair checkout line:

- Across **different users**, lower recent usage goes first.
- For the **same user**, request order stays FIFO.
- If two requests have the same score, older request wins.

Fairness is applied **per model**. A user's heavy traffic on model A does not directly lower their priority on model B.

## Before You Enable It

You need:

1. A running Kthena deployment.
2. A `ModelRoute` and `ModelServer` for the model you want to protect.
3. A stable user identity per request.

Fairness scheduling reads identity from router context key `user_id`. In normal setups, this is populated by JWT auth (`sub` claim). If identity is missing, fairness scheduling rejects requests.

## How the Priority Score Works

Kthena tracks usage in a sliding window for each `(user, model)` pair.

First, token usage is weighted:

$$
weighted\_tokens = input\_tokens \times inputTokenWeight + output\_tokens \times outputTokenWeight
$$

Then request priority is computed:

$$
priority = tokenWeight \times weighted\_tokens + requestNumWeight \times requestCount
$$

Lower score means higher priority.

In plain terms: users who used fewer resources recently get served sooner during contention.

## Enable Fairness Scheduling (Quick Start)

Set fairness values in your Helm values file:

```yaml
networking:
  kthenaRouter:
    fairness:
      enabled: true
      windowSize: "1h"
      inputTokenWeight: 1.0
      outputTokenWeight: 2.0
```

Apply:

```bash
helm upgrade --install kthena charts/kthena \
  --namespace kthena-system \
  --create-namespace \
  -f your-values.yaml
```

## Queue Modes

Kthena supports two dequeue behaviors:

1. **QPS mode** (`FAIRNESS_MAX_CONCURRENT=0`): dequeues at fixed rate (`FAIRNESS_MAX_QPS`).
2. **Concurrency-gated mode** (`FAIRNESS_MAX_CONCURRENT>0`): only allows that many in-flight requests through the fairness gate per model.

Concurrency-gated mode is usually better when you want admission to match actual backend capacity.

## Configuration Reference

### Core settings

| Environment Variable | Default | What it controls |
| --- | --- | --- |
| `ENABLE_FAIRNESS_SCHEDULING` | `false` | Global on/off switch |
| `FAIRNESS_WINDOW_SIZE` | runtime default `5m` | Sliding window for recent usage |
| `FAIRNESS_INPUT_TOKEN_WEIGHT` | `1.0` | Input-token weight in tracked usage |
| `FAIRNESS_OUTPUT_TOKEN_WEIGHT` | `2.0` | Output-token weight in tracked usage |
| `FAIRNESS_QUEUE_TIMEOUT` | `60s` | Max queue wait before timeout |

`FAIRNESS_WINDOW_SIZE` must be between `1m` and `1h`; invalid values are ignored and the default is used.

### Queue behavior settings

| Environment Variable | Default | What it controls |
| --- | --- | --- |
| `FAIRNESS_MAX_CONCURRENT` | `0` | In-flight cap per model (`0` means QPS mode) |
| `FAIRNESS_MAX_QPS` | `100` | Dequeue rate in QPS mode |
| `FAIRNESS_PRIORITY_REFRESH_RETRIES` | `0` | Dequeue-time priority refresh attempts |
| `FAIRNESS_REBUILD_THRESHOLD` | `64` | Queue depth threshold for heap rebuild fallback |

### Priority score settings

| Environment Variable | Default | What it controls |
| --- | --- | --- |
| `FAIRNESS_PRIORITY_TOKEN_WEIGHT` | `1.0` | Weight of token usage in final score |
| `FAIRNESS_PRIORITY_REQUEST_NUM_WEIGHT` | `0.0` | Weight of request count in final score |

## Advanced Tuning

If Helm values are not enough, set extra env vars on the router Deployment:

```bash
kubectl -n kthena-system set env deployment/kthena-router \
  FAIRNESS_QUEUE_TIMEOUT=45s \
  FAIRNESS_MAX_CONCURRENT=32 \
  FAIRNESS_PRIORITY_REQUEST_NUM_WEIGHT=0.2 \
  FAIRNESS_PRIORITY_REFRESH_RETRIES=2 \
  FAIRNESS_REBUILD_THRESHOLD=64
```

Practical guidance:

- Keep `FAIRNESS_PRIORITY_REQUEST_NUM_WEIGHT=0` at first; add it only if tiny requests dominate.
- Prefer `FAIRNESS_MAX_CONCURRENT` over raw QPS when backend capacity is known.
- Increase `FAIRNESS_OUTPUT_TOKEN_WEIGHT` if generated tokens are costlier than prompt tokens.
- Reduce `FAIRNESS_WINDOW_SIZE` if you need fairness to react faster to traffic shifts.

## Verify It Is Working

Check the active fairness env vars:

```bash
kubectl -n kthena-system get deployment kthena-router -o yaml | grep FAIRNESS
```

Check fairness metrics:

```bash
kubectl -n kthena-system port-forward deploy/kthena-router 8080:8080
curl -s http://localhost:8080/metrics | grep kthena_router_fairness_queue
```

Useful metrics:

- `kthena_router_fairness_queue_size`
- `kthena_router_fairness_queue_duration_seconds`
- `kthena_router_fairness_queue_cancelled_total`
- `kthena_router_fairness_queue_dequeue_total`
- `kthena_router_fairness_queue_inflight`
- `kthena_router_fairness_queue_priority_refresh_total`
- `kthena_router_fairness_queue_heap_rebuild_total`

Then generate contention with at least two users on the same model. The lighter user should see lower queue wait under load.

## Operational Notes

- Fairness state is kept in memory on each router pod.
- In multi-replica router deployments, fairness history is **not shared** across replicas.
- Queue timeout returns `HTTP 504`; client disconnect while waiting returns `HTTP 503`.
- Fairness behavior is router-side and does not require `ModelRoute` schema changes.

## Troubleshooting

### `missing userId in request body`

Router could not resolve `user_id` for fairness scheduling. Verify your identity path (usually JWT auth with a valid `sub` claim).

### Throughput dropped after enabling fairness

If `FAIRNESS_MAX_CONCURRENT=0`, you are in fixed-QPS mode. Raise `FAIRNESS_MAX_QPS` or switch to concurrency-gated mode.

### Queue wait is still high

Fairness improves sharing, not total capacity. If all backends are saturated, wait time still grows.

## Related Guides

- [Router Routing](./router-routing)
- [Router Rate Limiting](./rate-limit)
- [Router Observability](./router-observability)
