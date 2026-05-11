# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Thái Minh Kiên
**Submission date:** 11/5/2026
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.0)
Compose v2:    OK  (5.1.1)
RAM available: 7.65 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
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
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

I was surprised by how sensitive the Alertmanager/Slack pipeline is to timing. Initially, we didn't see the Slack notification because the container was restarted too quickly before the alert group-wait interval finished. Increasing the sleep times in the trigger script was essential to actually see the "firing" state in the UI.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 8, "output_tokens": 18, "quality": 0.873, "duration_seconds": 0.0684, "trace_id": "5833485f4ea1fb05d807c5ba5acb06a2", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T05:57:56.946898Z"}
```

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

The policy is designed to keep 100% of error traces, 100% of traces slower than 2s, and only 1% of healthy traces. Therefore, the fraction of traces kept per second is: `(Errors + Slow Traces) + 0.01 * (Healthy Traces)`. If 99% of requests are fast and healthy, the system drops ~98% of all generated traces to save storage space.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": { "psi": 3.4611, "drift": "yes" },
  "embedding_norm": { "psi": 0.0189, "drift": "no" },
  "response_length": { "psi": 0.0159, "drift": "no" },
  "response_quality": { "psi": 8.8488, "drift": "yes" }
}
```

### Which test fits which feature?

- **prompt_length**: **PSI** is perfect here because we just want to know if the "shape" of user input length has shifted significantly (e.g., users moving from short queries to long prompts).
- **embedding_norm**: **KS Test** is better for these continuous, normalized values because it detects even subtle shifts in the distribution that might signal a change in the underlying embedding model.
- **response_quality**: **KL Divergence** is the most meaningful for quality scores [0,1] as it measures the information loss between our "gold standard" reference and the current production model.

---

### Which prior-day metric was hardest to expose? Why?

The Day 20 Llama.cpp metrics were the hardest because they required a separate stub exporter running outside the main Docker Compose stack. Mapping the networking so that the containerized Prometheus could scrape the host-side exporter (on port 9102) was a tricky configuration step.

---

## 6. The single change that mattered most

The single most important change was fixing the **OpenTelemetry instrumentation logic** in `instrumentation.py`. Originally, the manual spans (`embed-text`, etc.) were not correctly linked to the FastAPI root span, resulting in "orphaned" traces with only 1 span. By fixing the `FastAPIInstrumentor` call to use the actual `app` instance, we enabled proper context propagation.

This is the difference between "works" and "useful" because disconnected spans are almost impossible to debug in a complex AI pipeline. Once the spans were correctly nested, we could finally see exactly which part of the LLM inference (Embedding vs. Generation) was causing latency bottlenecks. This directly implements the **distributed tracing** concepts from the deck, ensuring end-to-end visibility.
