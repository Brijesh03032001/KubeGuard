# 🛡️ KubeGuard — Autonomous Kubernetes AI Monitoring Agent

![KubeGuard Banner](images/banner.png)

> **Predict pod memory anomalies before they become OOMKills — then let an LLM write the fix.**

[![Node.js](https://img.shields.io/badge/Node.js-2E7D32?style=for-the-badge&logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org/)
[![Go 1.22](https://img.shields.io/badge/Go_1.22-007D9C?style=for-the-badge&logo=go&logoColor=white)](https://golang.org/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Prometheus](https://img.shields.io/badge/Prometheus-C1431F?style=for-the-badge&logo=prometheus&logoColor=white)](https://prometheus.io/)
[![TensorFlow](https://img.shields.io/badge/TensorFlow-C25600?style=for-the-badge&logo=tensorflow&logoColor=white)](https://tensorflow.org/)
[![Gemini 1.5 Pro](https://img.shields.io/badge/Gemini_1.5_Pro-6B4FA0?style=for-the-badge&logo=googlegemini&logoColor=white)](https://ai.google.dev/)

---

## Highlights

- Developed a Kubernetes monitor that polls **Prometheus every 1 second** and uses an **LSTM** forecast plus a **LightGBM** classifier to flag pod memory anomalies *before* limits are breached, benchmarked against **2 traditional approaches**: threshold alerts and Prometheus alerting rules.
- Normalized every sample against the pod's configured memory limit pulled from the Kubernetes API, turning raw usage into a portable **0–1 memory ratio** that generalizes across pods.
- Maintained a **10-point sliding window** of memory ratios feeding the LSTM, delaying predictions until a **10-second warm-up** fills the window so inference never runs on an incomplete series.
- Separated forecasting from classification with a **2-model hybrid**: TensorFlow Keras LSTM for temporal prediction, LightGBM for binary classification (**0** healthy, **1** anomaly).
- Chose persistent Node.js `child_process` **IPC over REST**, streaming the **1 Hz** rolling window to a Python subprocess through stdin/stdout so high-frequency inference does not pay HTTP overhead.
- Built a **Go 1.22** exporter with `client_golang` exposing **3 endpoints** and **3 custom metrics**, routing `kubeguard_predicted_memory_ratio` to the **Horizontal Pod Autoscaler** through Prometheus Adapter so workloads scale out *before* memory failure.
- Invoked **Gemini 1.5 Pro** only after a confirmed anomaly, generating **1 targeted `kubectl` remediation command** from live pod context — for example setting memory to **512Mi** for a predicted ratio of **1.24** against a healthy baseline of **0.91**.
- Split remediation into **2 triage paths**: recoverable anomalies get LLM-suggested fixes, while `CrashLoopBackOff` cases escalate to a cluster administrator instead of auto-remediating an already-failing pod.
- Added a complementary **Datadog** integration for API and background-worker failure visibility, structured logs and latency alerting — Prometheus stays the source for the memory-ratio and autoscaling pipeline.

---

## Architecture

```mermaid
flowchart TB
    K8S["Kubernetes API<br/>pod memory limits"] --> NORM
    PROM["Prometheus<br/>polled every 1s"] --> NORM["Normalizer<br/>usage / limit → 0-1 ratio"]

    NORM --> WIN["10-point sliding window<br/>10s warm-up"]
    WIN --> IPC["Node.js child_process IPC<br/>stdin / stdout @ 1 Hz"]

    IPC --> LSTM["TensorFlow Keras LSTM<br/>temporal forecast"]
    LSTM --> LGBM["LightGBM<br/>0 = healthy / 1 = anomaly"]

    LGBM --> EXP["Go 1.22 exporter<br/>client_golang<br/>3 endpoints · 3 metrics"]
    EXP --> ADAPT["Prometheus Adapter"]
    ADAPT --> HPA["Horizontal Pod Autoscaler<br/>scales before failure"]

    LGBM --> TRI{"Anomaly confirmed?"}
    TRI -->|"recoverable"| GEM["Gemini 1.5 Pro<br/>1 kubectl remediation"]
    TRI -->|"CrashLoopBackOff"| ESC["Escalate to cluster admin"]

    EXP --> DD["Datadog<br/>app + worker failures, logs, latency"]
```

## Detection to remediation

```mermaid
sequenceDiagram
    participant P as Prometheus
    participant N as Node.js agent
    participant M as Python ML subprocess
    participant G as Gemini 1.5 Pro
    participant H as HPA

    loop every 1 second
        N->>P: scrape pod memory
        N->>N: normalize to 0-1 ratio
        N->>M: push rolling window (IPC)
        M->>M: LSTM forecast → LightGBM classify
        M-->>N: predicted_ratio, label
    end
    N->>H: export kubeguard_predicted_memory_ratio
    alt label = 1 and pod is recoverable
        N->>G: pod context + predicted 1.24 vs baseline 0.91
        G-->>N: "kubectl set resources ... --limits=memory=512Mi"
        N-->>N: surface command for operator review
    else CrashLoopBackOff
        N-->>N: escalate to cluster administrator
    end
    H->>H: scale out before OOMKill
```

---

## Why predictive, not reactive

| Approach | When it fires |
|---|---|
| Static threshold alerts | **After** usage crosses the line |
| Prometheus alerting rules | **After** the rule window elapses |
| **KubeGuard** | **Before** the limit is breached — LSTM forecast + LightGBM classification on a 1 Hz window |

## Exported metrics

| Metric | Used by |
|---|---|
| `kubeguard_predicted_memory_ratio` | Prometheus Adapter → HPA |
| `kubeguard_anomaly_score` | Dashboards and alerting |
| `kubeguard_inference_latency_seconds` | Agent health |

## Tech stack

**Cluster** Kubernetes API · kubectl proxy · HPA · Prometheus Adapter · Deployment/Service YAML
**Metrics** Prometheus · PromQL · Datadog
**Exporter** Go 1.22 · `client_golang`
**Agent** Node.js 20 · Express · subprocess IPC
**ML** Python · TensorFlow/Keras LSTM · LightGBM · NumPy
**LLM** LangChain · Google Gemini 1.5 Pro
**Packaging** Docker multi-stage builds

## Quick start

```bash
git clone https://github.com/Brijesh03032001/KubeGuard.git
cd KubeGuard
```

```bash
kubectl apply -f k8s/        # exporter, agent deployment, service, HPA
```

```bash
cp .env.example .env         # PROMETHEUS_URL, GEMINI_API_KEY, DATADOG_API_KEY
npm install && npm start
```

---

**Built by [Brijesh Kumar](https://github.com/Brijesh03032001)** · [Portfolio](https://brijeshbuilds.com) · [LinkedIn](https://linkedin.com/in/brijeshkumar03)
