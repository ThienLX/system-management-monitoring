# BA — system-management-monitoring (Personal Project)

**Version:** 1.0  
**Owner:** (Bạn)  
**Date:** 2025-11-03

---

## 1) Tóm tắt điều hành
- **Mục tiêu:** Giám sát máy cá nhân & container bằng Prometheus + Grafana + Alertmanager; cảnh báo cơ bản; sẵn sàng triển khai tự động qua CI/CD lên Kubernetes.
- **Kết quả mong đợi:**
  - 01 dashboard tổng hợp (Host + Containers).
  - ≥ 06 cảnh báo nền tảng (CPU, RAM, Disk, Node down, Docker/cAdvisor down, Container restarts).
  - Pipeline CI/CD 1-nhánh (`main`) auto deploy vào namespace `monitoring`.

## 2) Phạm vi
### In scope
- Thu thập metrics: **Node Exporter**, **kubelet/cAdvisor** (container-level).
- Lưu trữ & query: **Prometheus** (retention 15–30 ngày).
- Trực quan: **Grafana** (provisioning datasource & dashboards).
- Cảnh báo: **Alertmanager** (webhook/Telegram/email).
- Triển khai: **Helm** chart `kube-prometheus-stack` (ưu tiên), **Kustomize** (fallback).
- CI/CD: **GitHub Actions** *hoặc* **Jenkins** (1 pipeline: lint → deploy).

### Out of scope
- Centralized logging, tracing (OTel), HA/Thanos, multi-cluster.

## 3) Personas & User Stories
**Owner (bạn)**
- **US1:** Xem tải CPU/RAM/Disk theo thời gian để phát hiện quá tải.
- **US2:** Biết container nào ngốn tài nguyên nhất.
- **US3:** Nhận cảnh báo khi *CPU > 90% trong 10 phút*, *Disk < 10%*, *Node down*.
- **US4:** Xem lịch sử 30 ngày, so sánh trước/sau tối ưu.
- **US5:** One-click deploy qua CI/CD.

**Tiêu chí chấp nhận (AC)**
- **AC1:** Dashboard realtime (scrape ≤ 5s); panel không lỗi dữ liệu.
- **AC2:** ≥ 6 cảnh báo hoạt động, gửi được qua ít nhất 1 kênh.
- **AC3:** `helm upgrade --install` thành công trong ≤ 5 phút trên cluster dev.
- **AC4:** CI/CD chạy tự động khi push vào `main` và rollout OK (exit 0).
- **AC5:** Mật khẩu Grafana không mặc định; retention Prometheus cấu hình được.

## 4) Yêu cầu chức năng (FR)
- **FR1:** Thu thập metrics host (CPU, RAM, Disk, Network, Load, Uptime).
- **FR2:** Thu thập metrics container (CPU, Memory, IO, Restarts).
- **FR3:** Dashboard “System Overview” + bảng Top N containers.
- **FR4:** Cảnh báo: NodeDown, HighCPU, HighMemory, LowDisk, cAdvisorDown, ContainerRestarts.
- **FR5:** Gửi cảnh báo qua webhook/Telegram/email (chọn 1 kênh tối thiểu).
- **FR6:** CI/CD: build (nếu có image tuỳ biến), deploy Helm vào `monitoring`.

## 5) Yêu cầu phi chức năng (NFR)
- **NFR1 — Hiệu năng:** scrape 5s (node), 15s (cadvisor); dashboard mở < 3s.
- **NFR2 — Độ tin cậy:** deploy idempotent; rollback Helm trong 1 lệnh.
- **NFR3 — Bảo mật:** mật khẩu Grafana trong Secret; service nội bộ (NodePort/Ingress hạn chế).
- **NFR4 — Lưu trữ:** retention 15–30 ngày; PVC tối thiểu 5–10GB dev.
- **NFR5 — Vận hành:** backup `prom_data` & `graf_data` hàng tuần (script tay).

## 6) Kiến trúc giải pháp
```
Node Exporter + kubelet/cAdvisor  -->  Prometheus (rules)  -->  Alertmanager --> (Webhook/Telegram/Email)
                                             |
                                             +--> Grafana (datasource Prometheus, dashboards)

Triển khai: Helm (kube-prometheus-stack) trên K8s; CI/CD: GitHub Actions/Jenkins
```

## 7) Lộ trình (Roadmap) & Mốc nghiệm thu
### Giai đoạn 1 — Foundation (Dev)
- Tạo cluster dev (kind/minikube), namespace `monitoring`.
- Helm install `kube-prometheus-stack` với `values-dev.yaml`.
- Import 1–2 dashboard (Node Exporter Full, Containers TopN).
- **M1:** AC1, AC3 đạt.

### Giai đoạn 2 — Alerting & Runbook
- Bật rules cảnh báo + gửi ra kênh chọn.
- Viết runbook ngắn cho từng alert.
- **M2:** AC2, AC5 đạt.

### Giai đoạn 3 — CI/CD
- GitHub Actions *hoặc* Jenkins pipeline: checkout → helm upgrade.
- Secrets: kubeconfig + webhook/token quản lý an toàn.
- **M3:** AC4 đạt.

### Giai đoạn 4 — Tối ưu & Ổn định
- Tinh chỉnh retention, PVC, scrape; tài liệu vận hành.
- **M4:** Ổn định & tài liệu đầy đủ.

## 8) Thiết kế CI/CD (mô tả BA)
- **Nhánh:** `main` (auto deploy dev).  
- **Bước:** 1) Checkout; 2) (Optional) Build image; 3) Thiết lập kubeconfig; 4) Helm repo update; 5) `helm upgrade --install` với `deploy/helm/values-dev.yaml`.
- **Secrets cần:** `KUBE_CONFIG`; (tuỳ chọn) `REGISTRY_USERNAME/PASSWORD`.

## 9) Môi trường & cấu hình
- **Dev:** NodePort (Grafana/Prometheus/Alertmanager) để test nhanh.
- **Prod (sau):** Ingress + TLS; Grafana admin từ Secret; tách `values-prod.yaml`.
- **Prometheus:** retention 30d; evaluation 15s; rules bật.
- **Grafana:** adminPassword; provisioning datasource + dashboards.
- **Alertmanager:** route mặc định + 1 receiver hoạt động.

## 10) Dashboard & Rules (phạm vi tối thiểu)
### Dashboard
- Row Health: targets up/down, scrape duration.
- CPU (gauge + time series + per-core), Load.
- Memory (used/avail/swap).
- Disk (used% per mount + inode).
- Network (rx/tx, errors).
- Containers TopN (CPU/Memory/Restarts).

### Rules
- NodeDown (5m), HighCPU (10m), HighMemory (10m),
- LowDisk (10m), cAdvisorDown (5m), ContainerRestarts (phát hiện biến mất/khởi động lại bất thường).

## 11) Rủi ro & Giảm thiểu
- **R1:** Thiếu tài nguyên cluster → *Mitigation:* scrape dài hơn, retention ngắn, PVC nhỏ; dev dùng kind.
- **R2:** Cảnh báo nhiễu → tăng `for:` & lọc label theo namespace.
- **R3:** Lộ mật khẩu Grafana → dùng Secret; không commit plaintext.
- **R4:** Hỏng dữ liệu TSDB → backup PVC; hướng dẫn delete WAL an toàn.
- **R5:** CI/CD lỗi kubeconfig → test `helm template` trước khi merge.

## 12) Tiêu chí “Done”
- Tài liệu BA được chấp thuận.
- Repo có `deploy/helm/values-dev.yaml`, pipeline CI/CD mẫu, README “cách chạy”.
- Helm deploy thành công trên cluster dev; Grafana có dashboard; ≥ 1 receiver cảnh báo hoạt động.

## 13) Kế hoạch kiểm thử chấp nhận (UAT checklist)
- [ ] Prometheus “Targets” tất cả UP (node, cadvisor, self).
- [ ] Grafana đăng nhập được (mật khẩu không mặc định).
- [ ] Import được dashboard; panel không lỗi dữ liệu.
- [ ] Trigger test alert (giả CPU load) → nhận thông báo.
- [ ] CI/CD: push commit vào `main` → helm upgrade OK, không downtime đáng kể.

## 14) RACI (gọn)
- **R** (Responsible): Owner (bạn).
- **A** (Accountable): Owner.
- **C** (Consulted): —
- **I** (Informed): —

## 15) Backlog mở rộng (sau BA)
- GitOps (Argo CD), OAuth cho Grafana, NetworkPolicy, SOPS/Sealed Secrets.
- Long-term storage (Thanos/VictoriaMetrics).
- SLO/SLA & anomaly detection panel.

---

### Phụ lục A — Tham chiếu file trong repo
- `deploy/helm/values-dev.yaml` / `values-prod.yaml` — cấu hình Helm.
- `.github/workflows/deploy.yml` — GitHub Actions deploy.
- `Jenkinsfile` — Jenkins pipeline (tuỳ chọn).
- `deploy/kustomize/...` — baseline manifests (optional).
- `RUNBOOK.md` — xử lý sự cố & quy trình vận hành.
