# Alloy PoC — OpenShift Deploy Guide

## Overview

Deploy Grafana Alloy เป็น centralized receiver สำหรับรับ telemetry data จาก OpenShift cluster

```
CLF       → Alloy :3100  → Loki
UWM       → Alloy :8080  → Mimir
OTel App  → Alloy :4317/:4318 → Tempo
```

---

## Repository Structure

```
poc-alloy-test/
├── .github/workflows/deploy.yaml   ← GitHub Actions pipeline
└── charts/alloy/
    ├── Chart.yaml
    ├── Chart.lock
    ├── values.yaml                  ← ค่าที่ต้อง custom
    ├── charts/alloy-1.10.0.tgz     ← bundled helm chart
    ├── templates/alloy-config.yaml  ← สร้าง ConfigMap อัตโนมัติ
    └── tenants/
        └── poc/poc.alloy            ← tenant config (URL, auth)
```

---

## GitHub Secrets ที่ต้องตั้ง

| Secret | ค่า | หมายเหตุ |
|--------|-----|---------|
| `OPENSHIFT_TOKEN` | SA token | `oc get secret <sa-secret> -o jsonpath='{.data.token}' \| base64 -d` |
| `OPENSHIFT_SERVER` | cluster API URL | เช่น `https://api.xxx.ais.th:6443` |
| `OPENSHIFT_PROJECT` | namespace | เช่น `alloy` |

---

## สิ่งที่ต้อง Custom เมื่อ PoC กับ AIS

### 1. `charts/alloy/values.yaml`

```yaml
# เปลี่ยน registry จาก docker.io → AIS internal registry
image:
  registry: lib.matador.ais.co.th   # ← เปลี่ยน
  repository: grafana/alloy
  tag: "v1.17.0"                    # ← confirm กับ AIS ว่ามี tag นี้ไหม

# เปลี่ยน namespace ให้ตรงกับ AIS cluster
alloy:
  namespaceOverride: alloy          # ← เปลี่ยนให้ตรงกับ namespace จริง
```

### 2. `charts/alloy/tenants/poc/poc.alloy`

เปลี่ยน URL และ credentials ให้ตรงกับ backend ของ AIS:

```alloy
# Logs → Loki
loki.write "poc" {
  endpoint {
    url = "https://LOKI_URL/loki/api/v1/push"   # ← เปลี่ยน
    basic_auth {
      username = "USERNAME"                       # ← เปลี่ยน (หรือใช้ Vault)
      password = "PASSWORD"                       # ← เปลี่ยน (หรือใช้ Vault)
    }
    headers = { "X-Scope-OrgID" = "TENANT_ID" } # ← เปลี่ยน
  }
}

# Metrics → Mimir
prometheus.remote_write "poc" {
  endpoint {
    url = "https://MIMIR_URL/api/v1/push"        # ← เปลี่ยน
    basic_auth {
      username = "USERNAME"                       # ← เปลี่ยน
      password = "PASSWORD"                       # ← เปลี่ยน
    }
    headers = { "X-Scope-OrgID" = "TENANT_ID" } # ← เปลี่ยน
  }
}

# Traces → Tempo
otelcol.exporter.otlphttp "poc" {
  client {
    endpoint = "https://TEMPO_URL/otlp"          # ← เปลี่ยน
    auth     = otelcol.auth.basic.poc.handler
    headers = { "X-Scope-OrgID" = "TENANT_ID" } # ← เปลี่ยน
  }
}
```

### 3. `.github/workflows/deploy.yaml`

```yaml
# เปลี่ยน runner ให้ตรงกับ AIS runner
runs-on: dev-onprem-oci   # ← เปลี่ยนให้ตรงกับ runner ของ AIS
```

---

## วิธี Deploy

1. ตั้ง GitHub Secrets ครบ (TOKEN, SERVER, PROJECT)
2. ไปที่ **Actions → Deploy Alloy PoC → Run workflow**
3. เลือก action:
   - `check-only` — ทดสอบ connection ก่อน
   - `dry-run` — render chart โดยไม่ deploy จริง
   - `deploy` — deploy จริง
   - `delete` — ลบ release

---

## ทดสอบด้วย curl

Service DNS (ใช้แทน IP ได้เลย ไม่ต้อง `oc get svc`):
```
alloy-poc.<namespace>.svc.cluster.local
```

### Logs (CLF → Alloy)
```bash
curl -X POST http://alloy-poc.alloy-poc.svc.cluster.local:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{"streams":[{"stream":{"job":"poc-test-app","env":"poc","service":"my-service"},"values":[["1750000000000000000","ERROR: something went wrong in poc-test-app"]]}]}'
# expect: 204
# ดูใน Grafana → Explore → Loki → {job="poc-test-app"}
```

### Traces (OTel → Alloy)
```bash
curl -X POST http://alloy-poc.alloy-poc.svc.cluster.local:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"poc-test-app"}}]},"scopeSpans":[{"spans":[{"traceId":"0af7651916cd43dd8448eb211c80319c","spanId":"b7ad6b7169203331","name":"test-operation","kind":2,"startTimeUnixNano":"1750000000000000000","endTimeUnixNano":"1750000001000000000","attributes":[{"key":"http.method","value":{"stringValue":"GET"}}]}]}]}]}'
# expect: 200
# ดูใน Grafana → Explore → Tempo → TraceID: 0af7651916cd43dd8448eb211c80319c
```

### Metrics (UWM → Alloy)

Prometheus remote_write ใช้ protobuf + snappy encoding ทดสอบด้วย raw curl ไม่ได้ ต้องใช้วิธีใดวิธีหนึ่งด้านล่าง

**วิธีที่ 1: Python script** (จาก pod ที่มี Python)

```bash
# สร้าง pod ที่มี Python
oc run debug-py -n alloy-poc --image=python:3.11-slim --restart=Never --command -- sleep 3600

# เข้าไปใน pod แล้วรันสคริปต์
oc exec -it debug-py -n alloy-poc -- bash

pip install requests snappy prometheus-client -q

python3 - <<'EOF'
import time, requests, snappy
from prometheus_client.exposition import choose_encoder
from prometheus_client import CollectorRegistry, Gauge
from prometheus_client.openmetrics.exposition import generate_latest

# สร้าง metric จำลอง
registry = CollectorRegistry()
g = Gauge("poc_test_metric", "PoC test metric", ["env"], registry=registry)
g.labels(env="poc").set(42)

# ใช้ remote_write via prometheus_client internal
import struct
from prometheus_client.bridge.graphite import GraphiteBridge

# ส่งแบบง่าย: remote write โดยตรง
import urllib.request

url = "http://alloy-poc.alloy-poc.svc.cluster.local:8080/api/v1/metrics/write"
print(f"Endpoint: {url}")

# Build minimal WriteRequest protobuf manually (label + sample)
# แนะนำใช้ write_requests library แทน
EOF
```

**วิธีที่ 2: `promtool` (ถ้ามี Prometheus binary)**

```bash
# ถ้ามี promtool อยู่ใน runner/node
echo 'poc_test_metric{env="poc"} 42' | \
  promtool push metrics \
  http://alloy-poc.alloy-poc.svc.cluster.local:8080/api/v1/metrics/write
```

**วิธีที่ 3: ตั้ง UWM remote_write (วิธีจริงสำหรับ AIS)**

สร้าง/แก้ ConfigMap `user-workload-monitoring-config` ใน namespace `openshift-user-workload-monitoring`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-workload-monitoring-config
  namespace: openshift-user-workload-monitoring
data:
  config.yaml: |
    prometheus:
      remoteWrite:
        - url: http://alloy-poc.alloy-poc.svc.cluster.local:8080/api/v1/metrics/write
```

```bash
oc apply -f uwm-config.yaml
# UWM Prometheus จะ remote_write metrics ไปหา Alloy ทุก 60 วินาที
```

```
# ดูใน Grafana → Explore → Mimir → metric: up{namespace="openshift-user-workload-monitoring"}
```

---

## Ports Summary

| Port | Protocol | ใช้รับจาก |
|------|----------|----------|
| 3100 | HTTP | CLF (Cluster Log Forwarder) |
| 4317 | gRPC | OpenTelemetry (traces/metrics) |
| 4318 | HTTP | OpenTelemetry (traces/metrics) |
| 8080 | HTTP | UWM (Prometheus remote_write) |
| 12345 | HTTP | Alloy UI (debug) |

---

## Production Checklist (หลัง PoC)

- [ ] เปลี่ยน credentials ไปใช้ **Vault** แทน hardcode
- [ ] เปลี่ยน image registry เป็น `lib.matador.ais.co.th`
- [ ] confirm image tag `v1.17.0` มีใน registry
- [ ] เปลี่ยน runner เป็น runner จริงของ AIS (`itqad-on-premise-dev`)
- [ ] ตั้ง UWM `remote_write` ชี้มาที่ Alloy `:8080/api/v1/metrics/write`
- [ ] ตั้ง CLF ชี้มาที่ Alloy `:3100`
- [ ] ใช้ 2 repo ตาม pattern จริง (pipeline template + alloy helm)
- [ ] integrate ArgoCD (ถ้าต้องการ)
