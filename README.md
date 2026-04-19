# one-docs-gitops

基于 Kustomize + ArgoCD 的 GitOps 部署仓库，用于管理 `one-docs` 应用在 ACK 集群上的部署清单。

## 目录结构

```
one-docs-gitops/
├── base/                        # 基础层：所有环境共用的资源模板
│   ├── namespace.yaml
│   ├── deployment.yaml          # 镜像不含 tag，由 overlay 注入
│   ├── service.yaml             # ClusterIP，端口 80 → 8090
│   ├── hpa.yaml                 # 水平自动扩缩（CPU 70% / 内存 80%）
│   └── kustomization.yaml
├── overlays/
│   └── dev/                     # dev 环境覆盖层
│       └── kustomization.yaml   # Jenkins CI 自动更新 newTag
├── argocd-app.yaml              # ArgoCD Application 清单
├── harbor-secret-template.yaml  # Harbor 镜像拉取凭证创建说明
└── README.md
```

## 快速开始

### 1. 创建 Harbor 镜像拉取凭证

```bash
kubectl create secret docker-registry harbor-registry-secret \
  --docker-server=harbor.onepoker.cc \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --namespace=dev
```

### 2. 在 ArgoCD 中注册 GitOps 仓库

```bash
argocd repo add https://github.com/your-org/one-docs-gitops.git \
  --username <git-user> \
  --password <git-token>
```

### 3. 创建 ArgoCD Application

```bash
kubectl apply -f argocd-app.yaml
```

### 4. Jenkins CI 自动更新镜像 tag

Jenkins 在构建完镜像后，执行以下命令更新 `overlays/dev/kustomization.yaml`：

```bash
cd overlays/dev
kustomize edit set image harbor.onepoker.cc/dev/one-docs:<new-tag>
git add kustomization.yaml
git commit -m "ci: update one-docs image tag to <new-tag> [skip ci]"
git push origin main
```

ArgoCD 检测到仓库变更后，自动将新版本同步到 ACK 集群。

## 环境说明

| 环境 | Overlay 路径 | 命名空间 | 副本数 |
|---|---|---|---|
| dev | `overlays/dev` | dev | 1（HPA 最大 3）|
