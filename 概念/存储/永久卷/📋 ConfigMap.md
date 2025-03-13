ConfigMap 是一个让你可以存储其他对象所需要使用的配置的API对象。（服务其他API对象）

ConfigMap 概念允许你将配置清单与镜像内容分离，以保持容器化的应用程序的可移植性。

特性：

- 服务其他API对象
- 保存配置文件
  - 以键值的方式存储配置。
  - 如Pod的环境变量、命令行参数或者存储卷中的配置文件等
- 数据不进行加密
- ConfigMap 将配置数据和应用程序代码分开。



结构

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 类文件键
  game.properties: |      # 文件名称  
    enemy.types=aliens,monsters 
    player.maximum-lives=5    
  user-interface.properties: |      # 文件名称
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```



Pod引用ConfigMap

要在一个 Pod 的存储卷中使用 ConfigMap:

1. 创建一个 ConfigMap 对象或者使用现有的 ConfigMap 对象。多个 Pod 可以引用同一个 ConfigMap。
2. 修改 Pod 定义，在 `spec.volumes[]` 下添加一个卷。 为该卷设置任意名称，之后将 `spec.volumes[].configMap.name` 字段设置为对你的 ConfigMap 对象的引用。
3. 为每个需要该 ConfigMap 的容器添加一个 `.spec.containers[].volumeMounts[]`。 设置 `.spec.containers[].volumeMounts[].readOnly=true` 并将 `.spec.containers[].volumeMounts[].mountPath` 设置为一个未使用的目录名， ConfigMap 的内容将出现在该目录中。
4. 更改你的镜像或者命令行，以便程序能够从该目录中查找文件。ConfigMap 中的每个 `data` 键会变成 `mountPath` 下面的一个文件名。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config           # 需要挂载的volume名字
        mountPath: "/config"   # 将volume挂载到哪个目录
        readOnly: true
  volumes:
  # 你可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
  - name: config
    configMap:
      # 提供你想要挂载的 ConfigMap 的名字
      name: game-demo
      # 来自 ConfigMap 的一组键，将被创建为文件
      items:
      - key: "game.properties"
        path: "game.properties"
      - key: "user-interface.properties"
        path: "user-interface.properties"

```

ConfigMap 不会区分单行属性值和多行类似文件的值，重要的是 Pods 和其他对象如何使用这些值。



被挂载的 ConfigMap数据同步

当卷中使用的 ConfigMap 被更新时，所投射的键最终也会被更新。 kubelet 组件会在每次周期性同步时检查所挂载的 ConfigMap 是否为最新。 不过，kubelet 使用的是其本地的高速缓存来获得 ConfigMap 的当前值。 高速缓存的类型可以通过 [KubeletConfiguration 结构](https://kubernetes.io/zh-cn/docs/reference/config-api/kubelet-config.v1beta1/). 的 `configMapAndSecretChangeDetectionStrategy` 字段来配置。

以环境变量方式使用的 ConfigMap 数据不会被自动更新。 更新这些数据需要重新启动 Pod。

```shell
说明：
使用 ConfigMap 作为 subPath 卷挂载的容器将不会收到 ConfigMap 的更新。
```



不可变更的ConfigMap

好处：

- 保护应用，使之免受意外（不想要的）更新所带来的负面影响。
- 降低kube-apiserver压力提升集群性能

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

一旦某 ConfigMap 被标记为不可变更，则 *无法* 逆转这一变化，，也无法更改 `data` 或 `binaryData` 字段的内容。你只能删除并重建 ConfigMap。 因为现有的 Pod 会维护一个已被删除的 ConfigMap 的挂载点，建议重新创建这些 Pods。





创建ConfigMap

格式：`kubectl create configmap <映射名称> <数据源>`

kubectl create configmap NAME [--from-file=[key=]source] [--from-literal=key1=value1] [--dry-run=server|client|none]
[options]

参数：

```shell
mkdir -p configure-pod-container/configmap/

# 将示例文件下载到 `configure-pod-container/configmap/` 目录
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
wget https://kubernetes.io/examples/configmap/ui.properties -O configure-pod-container/configmap/ui.properties
```



--from-file=[]：此选项参数可以是目录也可以是文件     

- 目录：--from-file=configure-pod-container/configmap    

  ```shell
  kubectl create configmap game-config --from-file=configure-pod-container/configmap/
  
  # 验证
  kubectl describe configmaps game-config
  kubectl get configmaps game-config -o yaml
  ```

  `configure-pod-container/configmap` 目录下的所有文件，也就是 `game.properties` 和 `ui.properties` 打包到 game-config ConfigMap 中

- 文件：

  - `configure-pod-container/configmap` 目录下的所有文件，只有 `game.properties` 打包到 game-config ConfigMap 中

    多次使用 `--from-file` 参数，从多个数据源创建 ConfigMap

    ```shell
    kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties
    
    # 验证
    kubectl describe configmaps game-config-2
    kubectl get configmaps game-config-2 -o yaml
    ```

    

  - 可以多次使用 `--from-file` 参数，从多个数据源创建 ConfigMap

    ```yaml
    kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties --from-file=configure-pod-container/configmap/ui.properties
    
    # 验证
    kubectl describe configmaps game-config-2
    kubectl get configmaps game-config-2 -o yaml
    ```

- 为创建的ConfigMap定义键

  格式：kubectl create configmap NAME --from-file=<我的键名>=<文件路径>

  ```shell
  # 创建ConfigMap
  
  kubectl create configmap configmap --from-file=configure-pod-container/configmap/ui.properties
  
  kubectl create configmap configmap-key --from-file=configmap-key=configure-pod-container/configmap/ui.properties
  
  # 查看详细
  
  root@ubuntu-master:/home/ubuntu# kubectl describe configmap configmap
  Name:         configmap
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  
  Data
  ====
  ui.properties:              
  ----
  color.good=purple
  color.bad=yellow
  allow.textmode=true
  how.nice.to.look=fairlyNice
  
  
  
  BinaryData
  ====
  
  Events:  <none>
  
  
  root@ubuntu-master:/home/ubuntu# kubectl describe configmap configmap-key
  Name:         configmap-key
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  
  Data
  ====
  configmap-key:
  ----
  color.good=purple
  color.bad=yellow
  allow.textmode=true
  how.nice.to.look=fairlyNice
  
  
  
  BinaryData
  ====
  
  Events:  <none>
  
  root@ubuntu-master:/home/ubuntu# kubectl get  configmap configmap -oyaml
  apiVersion: v1
  data:
    ui.properties: |   
      color.good=purple
      color.bad=yellow
      allow.textmode=true
      how.nice.to.look=fairlyNice
  kind: ConfigMap
  metadata:
    creationTimestamp: "2024-12-11T08:54:05Z"
    name: configmap
    namespace: default
    resourceVersion: "3534501"
    uid: 80adb264-5cdb-44bc-9ddd-c9d3cc559840
  root@ubuntu-master:/home/ubuntu# kubectl get configmap configmap-key -oyaml
  apiVersion: v1
  data:
    configmap-key: |
      color.good=purple
      color.bad=yellow
      allow.textmode=true
      how.nice.to.look=fairlyNice
  kind: ConfigMap
  metadata:
    creationTimestamp: "2024-12-11T08:44:18Z"
    name: configmap-key
    namespace: default
    resourceVersion: "3533605"
    uid: 245b8a04-d23e-4fa2-a6a6-7bde8f572e38
  ```

  

--from-literal=[]：基于字面值生成 ConfigMap

```shell
kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
```



--from-env-file=[]：基于 env 文件创建 ConfigMap

Env 文件包含环境变量列表。其中适用以下语法规则:

- Env 文件中的每一行必须为 VAR=VAL 格式。
- 以＃开头的行（即注释）将被忽略。
- 空行将被忽略。
- 引号不会被特殊处理（即它们将成为 ConfigMap 值的一部分）。

```shell
# 将示例文件下载到 `configure-pod-container/configmap/` 目录
wget https://kubernetes.io/examples/configmap/game-env-file.properties -O configure-pod-container/configmap/game-env-file.properties
wget https://kubernetes.io/examples/configmap/ui-env-file.properties -O configure-pod-container/configmap/ui-env-file.properties

# Env 文件 `game-env-file.properties` 如下所示
cat configure-pod-container/configmap/game-env-file.properties
enemies=aliens
lives=3
allowed="true"


# 创建configmap

kubectl create configmap game-config-env-file \
       --from-env-file=configure-pod-container/configmap/game-env-file.properties
       
# 从 Kubernetes 1.23 版本开始，kubectl 支持多次指定 --from-env-file 参数来从多个数据源创建 ConfigMap。

kubectl create configmap config-multi-env-files \
        --from-env-file=configure-pod-container/configmap/game-env-file.properties \
        --from-env-file=configure-pod-container/configmap/ui-env-file.properties
```



示例演示

使用 ConfigMap 来配置 Redis：https://kubernetes.io/zh-cn/docs/tutorials/configuration/configure-redis-using-configmap/

