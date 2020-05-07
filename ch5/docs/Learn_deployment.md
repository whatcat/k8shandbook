#### pod控制器学习

##### ReplicationController 和 ReplicaSet

`RC`用来确保容器副本数始终维持在用户定义的副本数，即如果容器有异常退出，会自动创建新的pod来代替，而如果异常多出来的容器也会自动回收

RS与RC本质上没有不同，只是支持了集合式的`selector`

##### Deployment

`Deployment`为`pod`和`ReplicaSet`提供了一个声明式定义(declarative)方法，用来代替以前的`RC`

* 定义`Deployment`来创建`pod`和`ReplicaSet`
* 滚动升级和回滚应用
* 扩容与缩容
* 暂停和继续`Deployment`

样例代码

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-test
spec:
  replicas: 3
  minReadySeconds: 10 #这里需要估计一个合理的值，从容器启动到应用正产提供服务
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 # 更新时允许最大激增的容器数 默认为replicas的1/4向上取整数
      maxUnavailable: 0 # 更新时允许的最大 unavailable 容器数， 默认为replicas数的1/4向下取整数
  selector:
    matchLabels:
      app-1: nginx
      app-2: busybox
  template:
    metadata:
      labels:
        app-1: nginx
        app-2: busybox
    spec:
      containers:
        - name: app-1
          image: nginx:1.16.0
          imagePullPolicy: Always
          ports:
            - containerPort: 80
            - containerPort: 8080
        - name: app-2
          image: busybox
          imagePullPolicy: Never
          command: ['/bin/sh', '-c']
          args:
            - while :;do sleep 20; done

```

###### 滚动升级和回滚操作

`deployment`通过控制`RS`的副本文件，来实现滚动更新和回滚，具体的形式如图1所示

![doployment调度流程](images\deployment.png)

从图中可知的结论如下

* deployment调度replicaset, replicaset调度pod
* deployment管理多个replicaset版本，可用于回滚
* replicaset控制pod的行为，包括新增pod、删除pod

执行滚动升级操作的流程如下

1. 准备的docker image 容器如下

   ```yaml
   nginx:latest
   nginx:1.16.0
   nginx:1.17.0
   ```

   

2. 使用set image操作

   ```powershell
   [root@devops01 ~]# kubectl get rs
   NAME                        DESIRED   CURRENT   READY   AGE
   deployment-test-569cbbc8d   10        10        10      30m
   
   [root@devops01 ~]# kubectl set image deployment/deployment-test  app-1=nginx:1.17.0
   deployment.apps/deployment-test image updated
   
   [root@devops01 ~]# kubectl get rs
   NAME                         DESIRED   CURRENT   READY   AGE
   deployment-test-569cbbc8d    7         7         7       31m
   deployment-test-685d5587c9   6         6         1       30s
   
   [root@devops01 ~]# kubectl get pods
   NAME                               READY   STATUS              RESTARTS   AGE
   deployment-test-569cbbc8d-dxd7p    2/2     Running             0          3m9s
   deployment-test-569cbbc8d-kxrht    2/2     Running             0          3m9s
   deployment-test-569cbbc8d-lh58r    2/2     Running             4          31m
   deployment-test-569cbbc8d-rd8ls    2/2     Running             0          3m9s
   deployment-test-569cbbc8d-svhhc    2/2     Terminating         0          3m9s
   deployment-test-569cbbc8d-v4hpp    2/2     Running             0          3m9s
   deployment-test-569cbbc8d-zf5fm    2/2     Running             2          31m
   deployment-test-569cbbc8d-zws9t    2/2     Running             2          31m
   deployment-test-685d5587c9-2pvfb   0/2     ContainerCreating   0          13s
   deployment-test-685d5587c9-5284z   0/2     ContainerCreating   0          41s
   deployment-test-685d5587c9-d6vld   0/2     ContainerCreating   0          41s
   deployment-test-685d5587c9-lxsrv   0/2     ContainerCreating   0          41s
   deployment-test-685d5587c9-p7r4p   2/2     Running             0          41s
   deployment-test-685d5587c9-vl2nm   2/2     Running             0          41s
   [root@devops01 ~]# kubectl rollout status deployment/deployment-test
   Waiting for deployment "deployment-test" rollout to finish: 7 out of 10 new replicas have been updated...
   Waiting for deployment "deployment-test" rollout to finish: 7 out of 10 new replicas have been updated...
   Waiting for deployment "deployment-test" rollout to finish: 7 out of 10 new replicas have been updated...
   Waiting for deployment "deployment-test" rollout to finish: 8 out of 10 new replicas have been updated...
   
   [root@devops01 ~]# kubectl rollout history deployment/deployment-test
   deployment.apps/deployment-test 
   REVISION  CHANGE-CAUSE
   3         kubectl apply --filename=deploy1.yaml --record=true
   4         kubectl apply --filename=deploy1.yaml --record=true
   
   # 在将镜像还没有全部修改为1.17.0版本的时候，再一次将奖项版本升级到latest
   [root@devops01 ~]# kubectl set image deployment/deployment-test app-1=nginx:latest
   deployment.apps/deployment-test image updated
   [root@devops01 ~]# kubectl get rs
   NAME                         DESIRED   CURRENT   READY   AGE
   deployment-test-569cbbc8d    1         1         1       35m
   deployment-test-685d5587c9   7         7         7       4m6s
   deployment-test-86b74bfb69   5         5         0       2s
   
   [root@devops01 ~]# kubectl get rs
   NAME                         DESIRED   CURRENT   READY   AGE
   deployment-test-569cbbc8d    0         0         0       37m
   deployment-test-685d5587c9   4         4         4       6m19s
   deployment-test-86b74bfb69   9         9         5       2m15s
   ```

   本次操作的特点如下，首先是将replicas的数量提升为10个。然后通过`set image`操作去修改镜像名为`app-1`的容器镜像，分别为`1.17.0`，`latest`两个版本，这室，deployment会创建3个`rs`，这三个`rs`对应着3个不同的版本。

   最后，在`1.17.0`版本时，pod没有全部都到达时，有执行了`latest`操作。kubernetes并不会继续完成为完成的`1.17.0`的升级，而是马上开始`latest`的升级。它会继续关闭`1.16.0`的版本，直到为0，然后在继续关闭`1.17.0`的版本，直到`latest`版本的pod数量为10.

   

3. 使用patch操作

4. 使用pause操作和resume操作







###### 扩缩容操作

```powershell
[root@devops01 ~]# kubectl apply -f deployment1.yaml --record
[root@devops01 ~]# kubectl scale deployment/deployment-test --replicas=10
deployment.apps/deployment-test scaled
[root@devops01 ~]# kubectl get pods
NAME                              READY   STATUS              RESTARTS   AGE
deployment-test-569cbbc8d-dxd7p   0/2     ContainerCreating   0          6s
deployment-test-569cbbc8d-kxrht   0/2     ContainerCreating   0          6s
deployment-test-569cbbc8d-lh58r   2/2     Running             2          28m
deployment-test-569cbbc8d-rd8ls   0/2     ContainerCreating   0          6s
deployment-test-569cbbc8d-svhhc   0/2     ContainerCreating   0          6s
deployment-test-569cbbc8d-v4hpp   0/2     ContainerCreating   0          6s
deployment-test-569cbbc8d-vt8dw   0/2     ContainerCreating   0          6s
deployment-test-569cbbc8d-w9xxv   0/2     ContainerCreating   0          6s
deployment-test-569cbbc8d-zf5fm   2/2     Running             2          28m
deployment-test-569cbbc8d-zws9t   2/2     Running             2          28m
```









###### 暂停和继续操作



##### DaemonSet

Daemonset确保全部或者一些Node上运行一个Pod的副本，可以通过标签选择和污点来实现不分配pod到node上。随着DaemonSet随着node的变化，动态变化

应用场景

* promeutheus node exporter

* filebeat fluentd

* ceph-osd



##### Job



###### Job

Job负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束

###### CronJob

CronJob管理的是基于时间的Job,即（基于时间的循环创建执行）

* 在给定时间点只运行一次
* 周期性地在给定时间点运行

应用场景

周期性备份、提醒等事件

##### StatefulSet

StatefulSet作为Controller为pod提供唯一的标识。它可以保证部署和scale的顺序

statefulset是为了解决有状态服务的问题，其应用场景包括：

* 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于pvc实现
* 稳定的网络标志，即pod重新调度后其PodName和HostName不变，基于Headless Service(即没有cluster IP的Service)来实现
* 有序部署，有序扩展，即pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次进行(从0到N-1，在下一个pod运行之前，其之前的pod必须是Running和Ready状态)，基于init containers来实现
* 有序收缩，有序删除(从N-1到0)

##### Horizaontal Pod Autoscaling

应用的资源使用率通常有高峰和低谷的时候，如何消峰填谷，提高集群的整体资源利用率，让service中的pod个数自动调整。让pod水平自动缩放











