# java-cicd — Spring Boot + Docker + k3s + ArgoCD + GitHub Actions

Microservice Java nhỏ. Mỗi lần push vào `main`:

1. GitHub Actions build JAR → test → đóng Docker image → push lên `ghcr.io`.
2. CI sửa tag image trong `k8s/kustomization.yaml` rồi commit lại repo.
3. ArgoCD watch repo, thấy tag đổi → tự kéo manifest mới và rolling-update vào k3s.
4. K8s rolling update đảm bảo zero-downtime: pod cũ chỉ bị xoá khi pod mới Ready.

Không ai SSH vào server. Không ai gõ `kubectl apply`. Đó là GitOps.

## Sơ đồ luồng

```
[Mac dev]                    [GitHub]                       [Laptop Ubuntu / k3s]
   |                            |                                    |
   | git push main              |                                    |
   +--------------------------->|                                    |
                                |  Actions: test + build + push      |
                                |  → ghcr.io/<owner>/java-cicd:sha-xxx
                                |  → commit kustomization.yaml       |
                                |                                    |
                                |   (ArgoCD poll repo mỗi 3 phút     |
                                |    hoặc webhook)                   |
                                |<-----------------------------------+
                                |                                    |
                                |  ArgoCD sync → kubectl apply -k    |
                                |                                    |
                                |                              Pod cũ v1
                                |                              Pod mới v2 (chờ Ready)
                                |                              Drop pod cũ
                                |                                    |
                                |                              curl http://<svc>:80
```

## Cấu trúc repo

```
java-cicd/
├── pom.xml                      # Maven build
├── src/main/java/...            # Spring Boot app (1 controller)
├── src/test/java/...            # 2 test
├── Dockerfile                   # multi-stage: Maven build → Temurin JRE
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml          # rolling update + liveness/readiness probes
│   ├── service.yaml             # ClusterIP
│   ├── hpa.yaml                 # tự scale 2→6 pod khi CPU > 70%
│   └── kustomization.yaml       # CI patch newTag ở đây
├── argocd/
│   └── application.yaml         # ArgoCD Application
└── .github/workflows/ci.yml     # pipeline
```

## Endpoints

- `GET /` → text `Hello from java-cicd v1.0.0`
- `GET /actuator/health` → JSON `{"status":"UP", ...}`
- `GET /actuator/health/liveness` → liveness (K8s gọi)
- `GET /actuator/health/readiness` → readiness (K8s gọi)

## Chạy thử local — không cần Docker / K8s

```bash
cd /Users/deepi/projects/agent/java-cicd
mvn spring-boot:run
# tab khác:
curl localhost:8080/
curl localhost:8080/actuator/health
```

## Build & chạy bằng Docker

```bash
docker build -t java-cicd:dev .
docker run --rm -p 8080:8080 java-cicd:dev
curl localhost:8080/actuator/health
```

---

## Setup CI/CD từ đầu — làm theo thứ tự

> Quy ước: `[MAC]` chạy trên máy Mac dev, `[SERVER]` chạy trên laptop Ubuntu.

### Bước 1 — Push code lên GitHub `[MAC]`

```bash
cd /Users/deepi/projects/agent/java-cicd
git init
git add .
git commit -m "init java-cicd"

# Tạo repo private trên GitHub trước, rồi:
git remote add origin git@github.com:<OWNER>/java-cicd.git
git branch -M main
git push -u origin main
```

Lần push đầu, workflow sẽ chạy job `test` (passed) còn job `build-push` sẽ **chỉ build & push image** — ArgoCD chưa cài nên cluster chưa nhận gì cả. Đó là bình thường.

### Bước 2 — Bật quyền ghi cho GitHub Actions `[GITHUB UI]`

CI cần commit ngược lại repo (step "Update kustomization image tag"). Mặc định `GITHUB_TOKEN` không có quyền write.

**GitHub repo → Settings → Actions → General → Workflow permissions**:
- Chọn **Read and write permissions**
- Tick **Allow GitHub Actions to create and approve pull requests** (không bắt buộc nhưng tiện sau)
- Save

### Bước 3 — Cài k3s trên laptop Ubuntu `[SERVER]`

k3s = K8s thật, đầy đủ, gọn nhẹ, single-node tốt cho học.

```bash
curl -sfL https://get.k3s.io | sh -

# Cho user thường dùng được kubectl (không cần sudo)
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

kubectl get nodes
# NAME      STATUS   ROLES                  AGE   VERSION
# server    Ready    control-plane,master   1m    v1.31...
```

### Bước 4 — Cài metrics-server (cho HPA hoạt động) `[SERVER]`

k3s đã có sẵn metrics-server bật mặc định từ v1.21+. Kiểm tra:

```bash
kubectl top nodes
# Nếu báo error, cài lại:
# kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Bước 5 — Cài ArgoCD `[SERVER]`

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Đợi argocd-server Ready
kubectl -n argocd rollout status deploy/argocd-server --timeout=180s
```

Lấy admin password (mặc định):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Mở UI bằng port-forward (chạy nền cũng được):

```bash
kubectl -n argocd port-forward svc/argocd-server 8443:443 &
# Mở https://<server-ip>:8443  user: admin  password: <từ lệnh trên>
```

### Bước 6 — Cho phép k3s pull image private từ ghcr.io `[SERVER]`

Image trên `ghcr.io` thuộc repo private → cluster cần Personal Access Token (PAT) của GitHub để pull.

1. **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**
   - Generate new token (classic)
   - Scope chỉ cần: `read:packages`
   - Copy token (chỉ hiện 1 lần).

2. `[SERVER]`:

```bash
kubectl create namespace java-cicd

kubectl -n java-cicd create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=<GITHUB_USERNAME> \
  --docker-password=<PAT_TOKEN> \
  --docker-email=<email>

# Gán secret này làm imagePullSecrets mặc định cho serviceaccount của namespace
kubectl -n java-cicd patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"ghcr-pull"}]}'
```

> Nếu repo public thì bỏ qua bước này — không cần secret.

### Bước 7 — Sửa placeholder trong manifests `[MAC]`

Mở 3 file, replace `REPLACE_OWNER` thành GitHub username/org của anh (vd `deepiai112001-svg`):

- `k8s/deployment.yaml` — field `image:`
- `k8s/kustomization.yaml` — field `images[].name`
- `argocd/application.yaml` — field `repoURL`

(CI cũng có sed tự sửa 2 file đầu, nhưng lần đầu chưa chạy CI nên sửa tay.)

Commit & push:

```bash
git commit -am "set owner in manifests"
git push
```

### Bước 8 — Tạo ArgoCD Application `[SERVER]`

```bash
kubectl apply -f https://raw.githubusercontent.com/<OWNER>/java-cicd/main/argocd/application.yaml
# hoặc nếu repo private, copy file về server rồi:
# kubectl apply -f argocd/application.yaml
```

Mở ArgoCD UI → Application `java-cicd` xuất hiện. Vì image lúc này có thể chưa tồn tại (lần push đầu CI mới chạy), ArgoCD sẽ báo `ImagePullBackOff` — không sao, chờ workflow xong là tự fix.

### Bước 9 — Trigger deploy thật `[MAC]`

Đổi `app.version` trong `src/main/resources/application.yml` thành `1.0.1`, rồi:

```bash
git commit -am "bump version 1.0.1"
git push origin main
```

Theo dõi trên GitHub Actions tab. Khi `build-push` xong:

- Bot sẽ commit `chore: bump image to sha-xxxxx [skip ci]` vào `main`.
- ArgoCD detect (auto-sync 3 phút, hoặc bấm **Sync** trên UI cho nhanh).
- Pod mới được tạo, rolling update — pod cũ chỉ chết sau khi pod mới pass readiness.

Verify:

```bash
# [SERVER]
kubectl -n java-cicd get pods -w
kubectl -n java-cicd get hpa
kubectl -n java-cicd port-forward svc/java-cicd 8080:80
# tab khác:
curl localhost:8080/
# Hello from java-cicd v1.0.1
```

---

## Zero-downtime đến từ đâu?

K8s Deployment với `strategy.rollingUpdate` + `readinessProbe` đã là zero-downtime mặc định:

```
Pod v1.0.0 [Ready]   ──── traffic ────►
Pod v1.0.1 [Starting]   (chưa nhận traffic)
Pod v1.0.1 [Ready]   ──── traffic ────►
Pod v1.0.0 [Terminating]  (graceful 30s)
```

`maxUnavailable: 0` đảm bảo lúc nào cũng có pod cũ Ready cho đến khi pod mới Ready. `terminationGracePeriodSeconds: 30` + `spring.lifecycle.timeout-per-shutdown-phase: 20s` cho Spring kịp xử lý nốt request đang dở.

## Mở rộng tiếp — khi đã quen

**Blue-Green / Canary thật sự** (không phải rolling): cần Argo Rollouts hoặc Flagger — chúng giới thiệu `kind: Rollout` thay cho `Deployment`, cho phép switch traffic theo % hoặc duyệt tay. Thêm sau khi đã hiểu rolling update.

**SonarCloud**: trong `.github/workflows/ci.yml` đã có block comment sẵn. Tạo project trên `sonarcloud.io`, copy `SONAR_TOKEN` vào GitHub secrets, uncomment block đó.

**Webhook ArgoCD** thay vì poll 3 phút: GitHub repo → Settings → Webhooks → `https://<argocd-host>/api/webhook` (cần expose argocd-server qua public URL hoặc dùng Tailscale/Cloudflare Tunnel).

## Self-healing chứng minh thế nào

`[SERVER]` xoá thử 1 pod:

```bash
kubectl -n java-cicd delete pod -l app=java-cicd --field-selector=status.phase=Running | head -1
kubectl -n java-cicd get pods -w
# K8s sẽ tự tạo pod mới trong vài giây — đó là self-healing.
```

Hoặc thử sửa cluster bằng tay:

```bash
kubectl -n java-cicd scale deploy java-cicd --replicas=5
# ArgoCD selfHeal=true sẽ revert về 2 (giá trị trong Git) sau ~1 phút.
```

## Troubleshooting

- **CI lỗi `permission denied` khi push**: chưa bật write permission ở Bước 2.
- **Pod `ImagePullBackOff`**: thiếu secret `ghcr-pull` hoặc repo image chưa public — quay lại Bước 6.
- **ArgoCD `OutOfSync` mãi**: kiểm tra `argocd app get java-cicd` xem diff. Thường do `REPLACE_OWNER` còn sót — Bước 7.
- **HPA `<unknown>`**: metrics-server chưa lên — Bước 4.
- **Test fail trên CI**: chạy `mvn -B verify` local trước, đa số lỗi do version Java khác nhau (CI dùng 21).

## Workflow hàng ngày sau khi setup xong

```bash
# [MAC]
# sửa code...
git commit -am "..."
git push origin main
```

~2 phút sau: pod mới chạy trên server. Không touch server. Không gõ `kubectl`.
