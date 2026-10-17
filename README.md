# LearnHub Infra

Repo **GitOps** của LearnHub: chứa Helm chart cho từng service, add-on và manifest Kubernetes dùng chung, cùng các
ArgoCD Application. Trên EKS, ArgoCD theo dõi nhánh `main` của repo này và tự đồng bộ lên cluster (namespace
`learnhub`).

Luồng deploy:

```
push code (repo backend) → GitHub Actions: test → build & push image (ECR) → sửa image.tag trong helm-charts/<service>/values.yaml
                                                                                   │
                                                   ArgoCD phát hiện commit mới ◄──┘ → helm render → apply lên cluster
```

## Cấu trúc

```
infra/
├── k8s/
│   ├── namespaces/learnhub.yaml        # namespace cho toàn bộ service
│   ├── storage/gp3.yaml                # StorageClass mặc định (EBS gp3)
│   ├── addons/                         # values Helm cho LBC, ExternalDNS, External Secrets, ArgoCD + versions.env
│   ├── external-secrets/               # ClusterSecretStore + ExternalSecret dùng chung (shared, DB)
│   └── db/                             # Job restore/backup database (S3)
├── helm-charts/
│   ├── platform-deps/                  # RabbitMQ, Redis, Jaeger
│   └── <service>/                      # Chart.yaml, values.yaml (mặc định/kind), values-prod.yaml (EKS), templates/
└── argocd/
    ├── root.yaml                       # App of Apps
    └── applications/                   # 10 service + platform-deps + monitoring (sync-wave, auto-sync, self-heal)
```

| Chart | Port | Ghi chú |
|---|---|---|
| `api-gateway` | 8080 | Ingress `api.learnhub-edu.site` (chỉ prefix `/api`) |
| `identity-service` | 8081 | |
| `user-service` | 8082 | |
| `course-service` | 8083 | Pod Identity: đọc/ghi bucket media |
| `enrollment-service` | 8084 | Chạy được nhiều replica: job churn dùng ShedLock (bảng `shedlock` trong enrollment_db) |
| `payment-service` | 8085 | |
| `notification-service` | 8086 | SMTP Amazon SES qua Secret `notification-service-app` |
| `assessment-service` | 8087 | |
| `ai-service` | 8000 | Python/FastAPI; DB cần extension pgvector |
| `frontend` | 3000 | Next.js; Ingress `learnhub-edu.site` |
| `platform-deps` | — | RabbitMQ (StatefulSet + PVC), Redis, Jaeger |

## Triển khai trên EKS (production)

Hạ tầng AWS do Terraform tạo; script `up.sh` / `down.sh` cài add-on, áp dụng `k8s/`, chạy Job restore database rồi
`kubectl apply -f argocd/root.yaml`. ArgoCD cài các chart với `values.yaml` + `values-prod.yaml`. Secret lấy từ AWS
Secrets Manager qua External Secrets Operator — không có secret nào trong repo này.

## Chạy thử trên cluster local (kind)

```bash
# 1. Tạo cluster 3 node (file cấu hình nằm ở repo backend)
kind create cluster --name learnhub --config ../backend/docker/kind-config.yml
kubectl config use-context kind-learnhub

# 2. Namespace và Secret (không nằm trong Git)
kubectl apply -f k8s/namespaces/learnhub.yaml
kubectl -n learnhub create secret generic learnhub-shared-secrets \
  --from-literal=RABBITMQ_USER=learnhub --from-literal=RABBITMQ_PASSWORD=<password> --from-literal=JWT_SECRET=<secret>
kubectl -n learnhub create secret generic ai-service-secrets --from-literal=GEMINI_API_KEY=<key>

# 3. Cài chart với values.yaml mặc định (không dùng values-prod.yaml / ArgoCD)
helm install platform-deps helm-charts/platform-deps -n learnhub
helm install user-service helm-charts/user-service -n learnhub
```

Kiểm tra chart trước khi commit:

```bash
helm lint helm-charts/course-service -f helm-charts/course-service/values.yaml -f helm-charts/course-service/values-prod.yaml
helm template course-service helm-charts/course-service -f helm-charts/course-service/values.yaml -f helm-charts/course-service/values-prod.yaml
```

## Lưu ý

- Giá trị trong `env` là cấu hình không bí mật. Mật khẩu DB, JWT secret, Cognito, Stripe, SMTP, API key LLM nằm trong
  Kubernetes Secret (`secretNames`), không ghi thẳng vào `values.yaml`.
- `image.tag` do CI cập nhật tự động trong `values.yaml` — không sửa tay, không đặt `image.tag` trong `values-prod.yaml`,
  và không thêm dòng nào khác chứa chuỗi `tag: ` vào `values.yaml`.
