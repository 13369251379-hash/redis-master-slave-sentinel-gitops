# Redis 主从集群 — Prod 部署核查清单

> 逐项执行，全部 [x] 通过后方可投产。

## 前置环境

- [ ] 1. `kubectl get ns prod` — 确认 prod 命名空间已创建
- [ ] 2. `kubectl get sc alicloud-disk-essd` — 确认 ESSD StorageClass 可用
- [ ] 3. `kubectl get pods -n kube-system | grep csi` — 确认阿里云 CSI driver 已安装
- [ ] 4. 确认 `imagepullsecret.yaml` 中 docker 凭据已替换为真实值
- [ ] 5. `kubectl get nodes -o wide` — 确认节点可用区分布

## 代码审查

- [ ] 6. `kubectl kustomize overlays/prod/` — 确认无报错
- [ ] 7. `git diff --stat` — 确认文件变更符合预期

## 部署 — 基础验证

- [ ] 8. 在 ArgoCD 中创建 Application（手动模式）或 `kubectl apply -f apps/redis-prod.yaml`
- [ ] 9. `kubectl get pods -n prod` — 6 个 Pod 全部 Running
  ```
  redis-master-0     1/1   Running
  redis-slave-0      1/1   Running
  redis-slave-1      1/1   Running
  redis-sentinel-0   1/1   Running
  redis-sentinel-1   1/1   Running
  redis-sentinel-2   1/1   Running
  ```
- [ ] 10. `kubectl get pvc -n prod` — 3 个 PVC 全部 Bound（data-redis-master-0, data-redis-slave-0, data-redis-slave-1）
- [ ] 10b. Headless 服务验证 — `kubectl get svc -n prod redis-master-headless redis-slave-headless redis-sentinel`，CLUSTER-IP 均为 `None`
- [ ] 10c. 反亲和验证 — `kubectl get pods -n prod -o wide | grep redis`，master 与 2 个 slave 分布在 3 个不同节点，3 个 sentinel 分布在 3 个不同节点

## Redis 功能验证

```bash
PASS=$(kubectl get secret -n prod redis-secret -o jsonpath='{.data.redis-password}' | base64 -d)
```

- [ ] 11. Master 状态验证
  ```
  kubectl exec -n prod redis-master-0 -- redis-cli -a $PASS --no-auth-warning INFO replication | grep role
  # 期望: role:master
  # 期望: connected_slaves:2
  ```

- [ ] 12. Slave 状态验证
  ```
  kubectl exec -n prod redis-slave-0 -- redis-cli -a $PASS --no-auth-warning INFO replication | grep -E "role|master_host"
  # 期望: role:slave, master_host:redis-master-headless
  ```

- [ ] 13. Sentinel 监控验证
  ```
  kubectl exec -n prod redis-sentinel-0 -- redis-cli -p 26379 -a $PASS --no-auth-warning SENTINEL MASTER mymaster
  # 期望: 返回 master 的 Pod FQDN（redis-master-0.redis-master-headless.prod.svc.cluster.local）和端口 6379，而非 IP
  ```

- [ ] 14. 读写验证
  ```
  kubectl exec -n prod redis-master-0 -- redis-cli -a $PASS --no-auth-warning SET testkey "ok"
  kubectl exec -n prod redis-master-0 -- redis-cli -a $PASS --no-auth-warning GET testkey
  # 期望: "ok"
  ```

## 故障切换验证（非生产时间执行）

- [ ] 15. `kubectl delete pod -n prod redis-master-0` — 模拟 Master 宕机
- [ ] 16. 等待 30~60 秒，执行步骤 13 — 确认新 Master 选举完成
- [ ] 17. 新 Master 写入测试 key，确认可写
- [ ] 18. 旧 Master Pod 恢复后确认为 slave 角色
- [ ] 19. 故障转移**域名广播**验证
  ```
  kubectl exec -n prod redis-sentinel-0 -- redis-cli -p 26379 -a $PASS --no-auth-warning SENTINEL GET-MASTER-ADDR-BY-NAME mymaster
  # 期望: 返回新 master 的【域名】而非 IP，如 redis-slave-0.redis-slave-headless.prod.svc.cluster.local
  ```

## 注意事项

- **内部服务名**：生产内部交互一律使用 headless 服务名 `redis-master-headless` / `redis-slave-headless`（经集群 DNS 直接解析到 Pod IP），不暴露到外部；旧服务名 `redis-master` / `redis-slave` 已废弃，**引用方应用需同步更新**
- **反亲和依赖节点数**：master/slave 采用 required 反亲和，数据平面 3 个 Pod 必须落在 3 个不同节点，集群可用调度节点 **≥3**，否则 Pod 无法调度（sentinel 同样需要 3 个不同节点）
- **Sentinel 域名广播**：sentinel monitor 使用 master Pod FQDN，已开启 `resolve-hostnames yes` / `announce-hostnames yes`，故障转移后哨兵对外广播的是**节点域名**而非 IP；master/slave 均通过 `replica-announce-ip` 广播自身 FQDN，不要改回 IP 广播模式
- **内存红线**：`maxmemory 1280mb` + `noeviction`，超过阈值直接拒绝写入（保守起步值，压测后调整）；Pod limit 2Gi，高写入时 RDB/AOF fork 有 OOM 风险，**部署后监控 Master/Slave 内存**
- **读一致性**：`replica-serve-stale-data no`，slave 在分区分裂或全量重同步期间会拒绝读请求（保证不读到旧数据）
- Master 单副本，RollingUpdate 期间短暂不可写，**在低峰期触发更新**
- 密码变更需要**全量重启所有组件**，不能只改 Secret
- `imagepullsecret.yaml` 和 `secret.yaml` 含敏感信息，确认仓库访问权限
