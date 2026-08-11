# Redis GitOps

在 dev 和 test 环境中部署 Redis 的 Argo CD 配置。

## 目录结构

```
redis/
├── apps/                  # Argo CD Application 资源
├── base/                  # 共享的 Kubernetes 清单
│   ├── deployment.yaml    # Deployment（含密码启动参数）
│   ├── service.yaml       # ClusterIP Service
│   ├── secret.yaml        # Redis 密码 Secret
│   └── kustomization.yaml
└── overlays/              # 环境相关的覆盖配置
    ├── dev/
    └── test/
```

## 部署

直接应用 Argo CD Application：

```bash
kubectl apply -f apps/redis-dev.yaml
kubectl apply -f apps/redis-test.yaml
```

## 镜像

使用 `gitlab-docker.racobit.com/open/docker-reg/redis:7.4.3` 镜像，通过 `ECS-241/push-image.sh` 推送到私有仓库。

## 资源限制

- CPU：request 200m，limit 500m
- 内存：request 512Mi，limit 512Mi

## 访问方式

密码：`<REPLACE_PASSWORD>`（定义在 `base/secret.yaml`）

### 集群内部

- dev: `redis-cli -h redis.dev.svc.cluster.local -p 6379 -a <REPLACE_PASSWORD> --no-auth-warning`
- test: `redis-cli -h redis.test.svc.cluster.local -p 6379 -a <REPLACE_PASSWORD> --no-auth-warning`

### 集群外部

- dev: `redis-cli -h <node-ip> -p 31379 -a <REPLACE_PASSWORD> --no-auth-warning`
- test: `redis-cli -h <node-ip> -p 32379 -a <REPLACE_PASSWORD> --no-auth-warning`

## 注意事项

- Redis 不挂载 PVC，重启后数据会丢失，仅用于 dev/test 缓存场景。
- 不要在该实例中存储重要数据。


## DB 使用情况
- db 0, data-center-main
- db 1, bio-radar-wechat
- db 2, bio-radar-web
- db 3, yinling-backend
- db 4, data-center-cuiot
- db 5, life-radar-platform

