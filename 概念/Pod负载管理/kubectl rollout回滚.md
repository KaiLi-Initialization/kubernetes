`kubectl rollout` 是 Kubernetes 用来管理 **资源版本发布状态** 的命令，主要针对 **Deployment / DaemonSet / StatefulSet** 等控制器。

下面我整理一个 **常用命令清单 + 示例**，你可以直接上手。

------

## 1. 查看 rollout 状态

查看更新是否完成：

```bash
kubectl rollout status deployment <name>
kubectl rollout status statefulset <name>
kubectl rollout status daemonset <name>
```

例如：

```bash
kubectl rollout status statefulset web
```

------

## 2. 查看 rollout 历史

```bash
kubectl rollout history deployment <name>
kubectl rollout history statefulset <name>
```

加上 `--revision` 可以看某个具体版本：

```bash
kubectl rollout history deployment web --revision=2
```

⚠️ 注意：

- **Deployment** 支持记录多个版本，方便回滚。
- **StatefulSet / DaemonSet** 也有 ControllerRevision，但 **`kubectl rollout undo` 不支持**，需要手动回滚（修改镜像或 YAML）。

------

## 3. 回滚（仅 Deployment 支持）

将 Deployment 回滚到上一个版本：

```bash
kubectl rollout undo deployment <name>
```

指定版本回滚：

```bash
kubectl rollout undo deployment <name> --to-revision=2
```

------

## 4. 暂停 / 恢复 更新

暂停滚动更新：

```bash
kubectl rollout pause deployment <name>
```

恢复更新：

```bash
kubectl rollout resume deployment <name>
```

这个功能常用于 **分批更新 / 灰度发布**。
 例如：

1. 先更新一部分副本。
2. `pause` 停止，观察运行状态。
3. 没问题再 `resume` 继续。

------

## 5. 示例流程（Deployment）

```bash
# 部署
kubectl create deployment web --image=nginx:1.19

# 更新镜像
kubectl set image deployment web nginx=nginx:1.21

# 查看更新状态
kubectl rollout status deployment web

# 查看历史版本
kubectl rollout history deployment web

# 回滚到上一个版本
kubectl rollout undo deployment web
```

------

✅ 总结：

- **Deployment** → 完整支持 `kubectl rollout`（状态、历史、暂停、恢复、回滚）。
- **StatefulSet / DaemonSet** → 只能用 `status` / `history`，不支持 `undo`，回滚要靠手动修改镜像或 `partition`。

------

要不要我帮你做一个 **Deployment vs StatefulSet 的 rollout 对比表格**，这样你一眼就能看出区别？