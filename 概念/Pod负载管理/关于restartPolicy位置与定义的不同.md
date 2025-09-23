关于restartPolicy位置与定义的不同

restartPolicy 所在位置影响的对象有所不同，下面我们以POD为例来简单说明一下。

POD下共有两处可设置startPolicy参数

一：针对所有POD进行管理

```shell

root@ubuntu-master:~# kubectl explain pod.spec.restartPolicy
KIND:       Pod
VERSION:    v1

FIELD: restartPolicy <string>
ENUM:
    Always
    Never
    OnFailure

DESCRIPTION:
    Restart policy for all containers within the pod. One of Always, OnFailure,
    Never. In some contexts, only a subset of those values may be permitted.
    Default to Always. More info:
    https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy

    Possible enum values:
     - `"Always"`
     - `"Never"`
     - `"OnFailure"`

     
# 定义POD控制下的所有container的重启策略，切有三中方式：`"Always"`，`"Never"`，"OnFailure"`。
```

二：针对Deployment控制器下的initContainer进行管理

```shell

root@ubuntu-master:~# kubectl explain pod.spec.containers.restartPolicy
KIND:       Pod
VERSION:    v1

FIELD: restartPolicy <string>


DESCRIPTION:
    RestartPolicy defines the restart behavior of individual containers in a
    pod. This field may only be set for init containers, and the only allowed
    value is "Always". For non-init containers or when this field is not
    specified, the restart behavior is defined by the Pod's restart policy and
    the container type. Setting the RestartPolicy as "Always" for the init
    container will have the following effect: this init container will be
    continually restarted on exit until all regular containers have terminated.
    Once all regular containers have completed, all init containers with
    restartPolicy "Always" will be shut down. This lifecycle differs from normal
    init containers and is often referred to as a "sidecar" container. Although
    this init container still starts in the init container sequence, it does not
    wait for the container to complete before proceeding to the next init
    container. Instead, the next init container starts immediately after this
    init container is started, or after any startupProbe has successfully
    completed.

    
# RestartPolicy 定义了 Pod 中各个容器的重启行为， 此字段只能为 Init 容器设置，并且唯一允许的值为“Always”。对于非 Init 容器或未指定此字段的情况，重启行为由 Pod 的重启策略和容器类型决定。此 Init 容器将在退出时持续重启，直到所有常规容器终止。所有常规容器完成后，所有 restartPolicy 为“Always”的 Init 容器都将被关闭。

```

