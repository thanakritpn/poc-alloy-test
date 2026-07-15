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

หา Service IP ก่อน:
```bash
oc get svc -n <namespace>
```

### Logs (CLF → Alloy)
```bash
curl -X POST http://<ALLOY_SVC_IP>:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{
    "streams": [{
      "stream": {"job": "test", "env": "poc"},
      "values": [["'"$(date +%s)000000000"'", "test log message"]]
    }]
  }'
# expect: 204
```

### Traces (OTel → Alloy)
```bash
curl -X POST http://<ALLOY_SVC_IP>:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[]}'
# expect: 200
```

### Metrics (UWM → Alloy)
```bash
curl -X POST http://<ALLOY_SVC_IP>:8080/api/v1/metrics/write \
  -H "Content-Type: application/x-protobuf" \
  -H "X-Prometheus-Remote-Write-Version: 0.1.0"
# expect: 400 (body ว่าง) หรือ 204 (มี data จริง)
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
