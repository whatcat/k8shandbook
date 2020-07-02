### Taint 和 Toleration

**节点亲和性，是POD的一种属性(偏好或硬性要求)，它使pod被吸引到一类特定的节点。taint则相反，它使*节点*能够*排斥*一类特定的pod**

**Taint和toleraation相互配合，可以用来避免pod被分配到不合适的节点上。每个节点上都可以应用一个或多个taint，这表示对于那些不能容忍这些taint的pod,是不会被节点接受的。如果将toleration应用于pod上，则表示这些pod可以(但不要求)被调度到具有匹配taint的节点上**



#### 污点（taint）

1、污点(Taint)的组成

使用`kubectl taint`ingling可以给某个Node节点设置污点，Node被设置上污点之后和pod之间存在了一种互斥的关系，可以让Node拒绝pod的调度执行，甚至将Node已经鵆的pod驱逐出去

每个污点的组成如下：

```yaml
key=value:effect
```

每个污点有一个key和value作为污点的标签，其中value可以为空，effect描述污点的左右。当前taint effect支持如下三个选项：

* **NoSchedule**:表示k8s将不会将pod调度到具有该污点的node上
* **preferNoSchedule**: 表示k8s将尽量避免将pod调度到具有污点的Node上
* **NoExecute**:表示k8s将不会将pod调度到具有该污点的Node上，同事会将Node上已经存在的pod驱逐出去

2、污点的设置，查看和去除

```powershell
# 设置污点
kubectl taint nodes node1 key1=value1:NoSchedule

# 节点说明中，查看Taint字段
kubectl describe pod pod-name

# 去除污点
kubectl taint nodes node1 key1=value1:NoSchedule-
```

#### 容忍（Tolerations）

**设置了污点的Node将根据taint的effect:NoSchedule、PreferNotSchedule、NoExecute和pod之间产生互斥的关系，pod将在一定程度上不会被调度到Node上。但我们可以在Pod上设置容忍(Toleration)，意思是设置了容忍的Pod将可以容忍污点的存在，可以被调度到存在污点的Node上**

**pod.spec.tolerations**

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
- key: "key2"
  operator: "Exists"
  effect: "NoSchedule"


```

* 其中key, value, effect要与Node上设置的taint保持一致
* operator的值为Exists将会忽略value值
* tolerationSeconds 用于描述当Pod需要被驱逐时可以在Pod上继续保留运行的时间

1、当不指定key值时，表示容忍所有的污点key

```yaml
tolerations:
- operator: "Exists"

```

2、当不指定effect值时，表示容忍所有的污点作用

```yaml
tolerations:
- key: "key"
  operator: "Exists"

```

3、有多个master存在时，防止资源浪费，可以如下设置

```yaml
kubectl taint node Node-Name node-role.kubernetes.io/master=:PreferNoSchedule

```



#### 实验

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-4
  labels:
    app: pod-4
spec:
  containers:
  - name: pod-4
    image: nginx:1.18.0
  tolerations:
  - key: "check"
    operatot: "Equal"
    value: "zhang"
    effect: "NoExecute"
    tolerationSeconds: 3600


```

运行3600秒后，自动关闭

```yaml
[root@k8s taint]# cat  deployment_tain.yaml 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-taint
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      release: stabel
  template:
    metadata:
      labels:
        app: myapp
        release: stabel
        env: test
    spec:
      tolerations:
      - key: "check"
        operator: "Equal"
        value: "zhangjianfeng"
        effect: "NoSchedule"
        #tolerationSeconds: 360
      containers:
      - name: myapp
        image: wangyanglinux/myapp:v2
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80


```











