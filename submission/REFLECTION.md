# Day 23 Lab Reflection

**Student:** TrinhQuangHung
**Submission date:** 2026-06-29
**Lab repo URL:** https://github.com/KinasTomes/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.3)
Compose v2:    OK  (5.1.3)
RAM available: 7.38 GB (OK)
Ports free:    OK
Report written: D:\Code\vinuni\Day23-Track2-Observability-Lab\00-setup\setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+75s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+10s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

I was surprised by how seamlessly Prometheus coordinates with Alertmanager to group multiple events. The grouping interval prevents alert storms (paging oncall multiple times for the same service down event), which is a key design aspect in production environments.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.667, "duration_seconds": 0.2436, "trace_id": "6a7cf2a6a54460605321fb7b1ffe1861", "event": "prediction served", "level": "info", "timestamp": "2026-06-29T04:21:55.229179Z"}
```

Linked to `trace_id`: `6a7cf2a6a54460605321fb7b1ffe1861`.

### Tail-sampling math

If your service produced $N$ traces/sec, what fraction did the policy keep? Show the calculation.

The tail-sampling policy is:
- Keep all errors (status code = `ERROR`)
- Keep all slow traces (latency > `2000ms`)
- Keep 1% of healthy/fast traces (sampling_percentage = `1%`)

Let:
- $p_{err}$ be the fraction of error traces.
- $p_{slow}$ be the fraction of slow traces.
- $p_{healthy}$ be the fraction of healthy/fast traces ($p_{healthy} = 1 - p_{err} - p_{slow}$).

The kept fraction $f$ of traces is calculated as:
$$f = p_{err} + p_{slow} + 0.01 \times p_{healthy}$$

For example, if the service generates $N = 100$ traces/sec, with $p_{err} = 0.01$ (1% errors), $p_{slow} = 0.02$ (2% slow), and $p_{healthy} = 0.97$ (97% healthy):
$$f = 0.01 + 0.02 + 0.01 \times 0.97 = 0.0397 = 3.97\%$$

This means we only keep $3.97$ traces/sec out of $100$ traces/sec, saving over $96\%$ of storage cost while retaining all critical data.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

- **`prompt_length`**: **PSI (Population Stability Index)** or **KS Test**. PSI is excellent for monitoring simple numeric features where we can bin ranges (e.g. short/medium/long prompts) and get a stable index score to detect sudden shifts in prompt complexity.
- **`embedding_norm`**: **KS Test (Kolmogorov-Smirnov)**. Embedding norms represent the continuous vector length. The KS test is ideal because it is non-parametric, compares continuous cumulative distributions directly, and does not require arbitrary binning.
- **`response_length`**: **PSI**. LLM generation lengths are typically grouped into buckets for cost modeling (e.g., token-count tiers). PSI is highly interpretable for business stakeholders tracking token consumption changes.
- **`response_quality`**: **KL-Divergence**. Since response quality scores are probabilities or bounded continuous evaluations [0,1], KL-Divergence measures exactly how much information is lost if we assume the baseline distribution instead of the current drifted distribution, which is sensitive to quality regressions.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Exposing the Day 20 llama.cpp serving metrics was the most challenging because llama.cpp does not natively output Prometheus-formatted metrics directly. We had to use sidecars to extract and translate the stdout or HTTP endpoint output into PromQL-compatible signals, which required extra service management and network wiring.

---

## 6. The single change that mattered most

Implementing **tail-sampling** in the OTel Collector made the single biggest difference between a stack that simply 'works' and one that is actually 'useful'. In a high-throughput production LLM environment, saving 100% of traces results in astronomical storage costs and performance overhead, especially for long LLM responses.

By filtering for 100% of errors and slow requests, while down-sampling healthy fast traces to 1%, we drastically reduced the telemetry footprint by over 90% without losing visibility into critical failure modes or latency outliers. This directly connects to the concept of **telemetry efficiency and volume control** from Section 7 of the lecture deck.
