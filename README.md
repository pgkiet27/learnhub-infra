# LearnHub Infra

Repo **GitOps** của LearnHub: chứa Helm chart cho từng service và các ArgoCD Application. ArgoCD theo dõi repo này và
tự đồng bộ lên cluster Kubernetes (namespace `learnhub`).

Luồng deploy:

```
push code (repo backend) → GitHub Actions: test → build & push image → sửa image.tag trong helm-charts/<service>/values.yaml
                                                                              │
                                              ArgoCD phát hiện commit mới ◄──┘ → helm render → apply lên cluster
```

## Cấu trúc

```
infra/
├── k8s/namespaces/learnhub.yaml        # namespace cho toàn bộ service
├── helm-charts/<service>/              # Chart.yaml, values.yaml, templates/{deployment,service}.yaml
└── argocd/applications/<service>.yaml  # ArgoCD Application trỏ tới chart tương ứng (auto-sync, self-heal)
```

| Chart | Port | Ghi chú |
|---|---|---|
| `identity-service` | 8081 | |
| `user-service` | 8082 | |
| `course-service` | 8083 | |
| `enrollment-service` | 8084 | Chạy được nhiều replica: job churn dùng ShedLock (bảng `shedlock` trong enrollment_db) |
| `payment-service` | 8085 | |
| `notification-service` | 8086 | Cần SMTP thật (ví dụ Amazon SES) qua Secret |
| `assessment-service` | 8087 | |
| `ai-service` | 8000 | Python/FastAPI; API key lấy từ Secret `ai-service-secrets`; DB cần extension pgvector |

## Yêu cầu

- Docker Desktop
- [kind](https://kind.sigs.k8s.io/), `kubectl`, `helm` (v3+)

## Các bước triển khai trên cluster local (kind)

```bash
# 1. Tạo cluster 3 node (file cấu hình nằm ở repo backend)
kind create cluster --name learnhub --config ../backend/docker/kind-config.yml
kubectl config use-context kind-learnhub

# 2. Cài ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w                       # chờ tất cả Running

# 3. Namespace cho LearnHub
kubectl apply -f k8s/namespaces/learnhub.yaml

# 4. Secret (không nằm trong Git) — ví dụ cho ai-service
kubectl -n learnhub create secret generic ai-service-secrets --from-literal=GEMINI_API_KEY=<key>

# 5. Đăng ký các ứng dụng với ArgoCD
kubectl apply -f argocd/applications/
```

Mở giao diện ArgoCD:

```bash
kubectl port-forward svc/argocd-server -n argocd 8090:443        # https://localhost:8090, user: admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"   # mật khẩu (base64)
```

Kiểm tra chart trước khi commit:

```bash
helm lint helm-charts/ai-service
helm template ai-service helm-charts/ai-service
```

## Lưu ý

- Repo này **chỉ chứa các service ứng dụng**. PostgreSQL (mỗi service một DB, `ai-db` cần image `pgvector/pgvector`),
  RabbitMQ và Redis phải được triển khai riêng (ví dụ Bitnami Helm charts hoặc dịch vụ managed như RDS, Amazon MQ);
  hostname của chúng được khai báo trong `env` của từng `values.yaml`.
- Giá trị trong `env` là cấu hình mặc định không bí mật. Mật khẩu DB, JWT secret, Cognito, Stripe, SMTP, API key LLM
  phải đưa vào Kubernetes Secret, không ghi thẳng vào `values.yaml`.
- `image.tag` do CI cập nhật tự động — không sửa tay.
