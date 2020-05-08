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
  minReadySeconds: 10 #这里需要估计一个合理的值，从容器启动到应用正常提供服务
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
          readinessProbe: # 添加存活探针，用来保证，当新起来的pod可用，在删掉旧的pod
            tcpSocket:
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
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

3. 使用patch操作, 补丁操作，参数可以是`yaml`或者`json`格式

4. 使用pause操作和resume操作



###### 扩缩容操作

```powershell
[root@devops01 ~]# kubectl apply -f deployment1.yaml --record

## 扩容操作
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

## 缩容操作
[root@devops01 ~]# kubectl scale deployment/deployment-test --replicas=3
deployment.apps/deployment-test scaled
[root@devops01 ~]# kubectl get pods
NAME                               READY   STATUS        RESTARTS   AGE
deployment-test-6b9cd7d8fb-8bw7h   2/2     Running       0          4m47s
deployment-test-6b9cd7d8fb-j6gtq   2/2     Running       0          8m4s
deployment-test-6b9cd7d8fb-k6d4x   2/2     Running       0          5m11s
deployment-test-6b9cd7d8fb-ljnb5   2/2     Terminating   0          12s
deployment-test-6b9cd7d8fb-nnwrf   1/2     Terminating   0          12s
```



###### 回滚到指定的历史版本

`deployment`会保存指定数量的`RS`版本，如下的命令。但是在多次执行`kubectl rollout undo`会陷进两个版本的循环，那么如何回滚到指定的版本呢？带参数`--to-revison`来实现。代码案例如下

```powershell
[root@devops01 ~]# kubectl rollout history  deployment/deployment-test
deployment.apps/deployment-test 
REVISION  CHANGE-CAUSE
4         kubectl apply --filename=deploy1.yaml --record=true
6         kubectl apply --filename=deploy1.yaml --record=true
7         kubectl apply --filename=deploy1.yaml --record=true

[root@devops01 ~]# kubectl rollout undo  deployment/deployment-test --to-revision=7
deployment.apps/deployment-test skipped rollback (current template already matches revision 7)
[root@devops01 ~]# kubectl rollout undo  deployment/deployment-test --to-revision=6
deployment.apps/deployment-test rolled back
```



###### 暂停和继续操作

在对`deploymnet`执行滚动更新操作的过程中，可以暂停`pause`这个操作。暂停后，需要继续操作是使用`resume`

```powershell
## 更新镜像
[root@devops01 ~]# kubectl set image deployment/deployment-test app-1=nginx:1.17.0
deployment.apps/deployment-test image updated

## 暂停操作
[root@devops01 ~]# kubectl rollout pause deployment/deployment-test
deployment.apps/deployment-test paused

## 查看操作
[root@devops01 ~]# kubectl get pods -o wide
NAME                               READY   STATUS    RESTARTS   AGE    IP           NODE       NOMINATED NODE   READINESS GATES
deployment-test-6b9cd7d8fb-j6gtq   2/2     Running   0          25s    172.17.0.7   devops01   <none>           <none>
deployment-test-6dcd86cd9f-q8pwj   2/2     Running   0          2m9s   172.17.0.4   devops01   <none>           <none>
deployment-test-6dcd86cd9f-qsfmb   2/2     Running   0          2m9s   172.17.0.3   devops01   <none>           <none>
deployment-test-6dcd86cd9f-wpg77   2/2     Running   0          2m9s   172.17.0.2   devops01   <none>           <none>

## 继续操作
[root@devops01 ~]# kubectl rollout resume deployment/deployment-test 
deployment.apps/deployment-test resumed
## 查看
[root@devops01 ~]# kubectl get pods -o wide
NAME                               READY   STATUS              RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
deployment-test-6b9cd7d8fb-j6gtq   2/2     Running             0          2m56s   172.17.0.7   devops01   <none>           <none>
deployment-test-6b9cd7d8fb-k6d4x   0/2     ContainerCreating   0          3s      <none>       devops01   <none>           <none>
deployment-test-6dcd86cd9f-q8pwj   2/2     Terminating         0          4m40s   172.17.0.4   devops01   <none>           <none>
deployment-test-6dcd86cd9f-qsfmb   2/2     Running             0          4m40s   172.17.0.3   devops01   <none>           <none>
deployment-test-6dcd86cd9f-wpg77   2/2     Running             0          4m40s   172.17.0.2   devops01   <none>           <none>
```

###### patch 补丁操作

支持deployment node等api资源

```powershell
# 更新镜像补丁
[root@devops01 ~]# kubectl patch deployment/deployment-test \
> --patch '{"spec": {"template": {"spec": {"containers": [{"name": "app-1","image":"nginx:latest"}]}}}}' 
deployment.apps/deployment-test patched

# 查看rs变化
[root@devops01 ~]# kubectl get rs
NAME                         DESIRED   CURRENT   READY   AGE
deployment-test-6b9cd7d8fb   1         1         1       21m
deployment-test-6dcd86cd9f   0         0         0       23m
deployment-test-7775dfb68b   3         3         2       61s

# 查看pod变化
[root@devops01 ~]# kubectl get pods
NAME                               READY   STATUS        RESTARTS   AGE
deployment-test-6b9cd7d8fb-j6gtq   2/2     Running       0          21m
deployment-test-6b9cd7d8fb-k6d4x   2/2     Terminating   0          18m
deployment-test-7775dfb68b-5bmlp   2/2     Running       0          65s
deployment-test-7775dfb68b-bkccn   2/2     Running       0          21s
deployment-test-7775dfb68b-mc4cd   2/2     Running       0          45s
```



##### DaemonSet

Daemonset确保全部或者一些Node上运行一个Pod的副本，可以通过标签选择和污点来实现不分配pod到node上。随着DaemonSet随着node的变化，动态变化

应用场景

* promeutheus node exporter
* filebeat fluentd
* ceph-osd

详细的部署使用，参看`prometheus`的部署方式

##### Job

任务的使用

###### Job

Job负责批处理任务，即仅执行**一次**的任务，它保证批处理任务的**一个**或**多个**Pod成功结束

**Job Spec**格式

* spec.template格式与pod一致
* RestartPolicy 仅支持Never和OnFailure
* 单个pod时，默认Pod成功运行后Job即结束
* `.spec.completions`标志Job结束需要成功运行的pod个数，默认为1
* `.spec.parallelism`标志并行运行的pod的个数，默认为1
* `.spec.activeDeadlineSeconds`标志失败pod的重试最大时间，超过这个时间，不会再重试

样例代码

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    metadata:
      name: pi
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never


```

执行查看

```powershell
# 创建 任务
[root@devops01 ~]# kubectl apply -f job1.yaml

# 查看状态
[root@devops01 ~]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           10s        3m27s

# 查看pod
[root@devops01 ~]# kubectl get pods
NAME                               READY   STATUS      RESTARTS   AGE
deployment-test-7775dfb68b-5bmlp   2/2     Running     0          20m
deployment-test-7775dfb68b-bkccn   2/2     Running     0          20m
deployment-test-7775dfb68b-mc4cd   2/2     Running     0          20m
pi-qhclm                           0/1     Completed   0          10s

# 查看日志
[root@devops01 ~]# kubectl logs   pi-qhclm
3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608640344181598136297747713099605187072113499999983729780499510597317328160963185950244594553469083026425223082533446850352619311881710100031378387528865875332083814206171776691473035982534904287554687311595628638823537875937519577818577805321712268066130019278766111959092164201989380952572010654858632788659361533818279682303019520353018529689957736225994138912497217752834791315155748572424541506959508295331168617278558890750983817546374649393192550604009277016711390098488240128583616035637076601047101819429555961989467678374494482553797747268471040475346462080466842590694912933136770289891521047521620569660240580381501935112533824300355876402474964732639141992726042699227967823547816360093417216412199245863150302861829745557067498385054945885869269956909272107975093029553211653449872027559602364806654991198818347977535663698074265425278625518184175746728909777727938000816470600161452491921732172147723501414419735685481613611573525521334757418494684385233239073941433345477624168625189835694855620992192221842725502542568876717904946016534668049886272327917860857843838279679766814541009538837863609506800642251252051173929848960841284886269456042419652850222106611863067442786220391949450471237137869609563643719172874677646575739624138908658326459958133904780275901
```



###### CronJob

CronJob管理的是基于时间的Job,即（基于时间的循环创建执行）

* 在给定时间点只运行一次
* 周期性地在给定时间点运行

应用场景

* 周期性备份、提醒等事件

**CronJob的知识点须知 CronJob Spec**

* **.spec.schedule**: 调度，必需字段，指定任务的运行周期，格式同`Crontab`

* **.spec.jobTemplate**: **Job模板**，必须字段，指定需要运行的任务，格式同Job

* **.spec.startingDeadlineSeconds**: 启动Job的期限(秒级别)，该字段可选的(限制这个job的执行时间)
  * 如果因为任何原因而错过了被调度的事件，那么错过执行时间的JOB将被认为是失败的
  * 如果没有指定，则没有限制

* **.spec.concurrencyPolicy**: 并发策略，该字段是可选的
  * 它指定了如何处理被`Cron Job`创建的`Job`的并发执行。只允许指定下面策略中的一种
    * Allow: 允许并发运行job
    * Forbid: 禁止并发运行，如果前一个没有完成，则直接跳过下一个
    * Replace: 取消当前正在运行的job,用一个新的来替换
* **.spec.suspend**: 挂起，该字段可选
  * 如果设置为true,后续所有执行都会被挂起
  * 它对已经开始执行的job不起作用，默认为false
* **.spec.successfulJobHistoryLimit** 和 **.spec.failedJobHistoryLimit**: 历史限制
  * 它指定了可以保留多少个完成和失败的job
  * 默认没有限制，所有成功和失败的job都会保留
  * 当运行一个Cron Job时，Job可以很快就堆积很多，推荐设置这两个字段的值
  * 设置限制的值为`0`，相关类型的job完成后将不会被保留

代码案例如下

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure # 重启策略为 Never 或者 OnFailure
  startingDeadlineSeconds: 30      # 限制30sec内完成
  concurrencyPolicy: Forbid        # 禁止并行cronjob
  successfulJobsHistoryLimit: 10   # 保留10个历史成功记录
  failedJobsHistoryLimit: 5        # 保留5个历史失败记录
  
```

部署执行，查看

```powershell
[root@devops01 ch5]# kubectl apply  -f  cronjob1.yaml 
cronjob.batch/hello created

# 查看
[root@devops01 ~]# kubectl get cronjob
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        <none>          22s
[root@devops01 ~]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           10s        34m
[root@devops01 ~]# kubectl get pod
NAME                               READY   STATUS      RESTARTS   AGE
deployment-test-7775dfb68b-5bmlp   2/2     Running     0          55m
deployment-test-7775dfb68b-bkccn   2/2     Running     0          54m
deployment-test-7775dfb68b-mc4cd   2/2     Running     0          55m
pi-qhclm                           0/1     Completed   0          34m

# 一分钟以后
# 查看pod
[root@devops01 ~]# kubectl get pod
NAME                               READY   STATUS      RESTARTS   AGE
deployment-test-7775dfb68b-5bmlp   2/2     Running     0          56m
deployment-test-7775dfb68b-bkccn   2/2     Running     0          55m
deployment-test-7775dfb68b-mc4cd   2/2     Running     0          56m
hello-1588903800-xjhg2             0/1     Completed   0          37s
pi-qhclm                           0/1     Completed   0          35m

# 查看job
[root@devops01 ~]# kubectl get job
NAME               COMPLETIONS   DURATION   AGE
hello-1588903800   1/1           9s         101s
hello-1588903860   1/1           8s         40s
pi                 1/1           10s        36m
```

**CronJob的限制**

Cron Job在每次调度运行时间内*大概*会创建一个job对象。这些创建job操作是**幂等**的。

**删除 crob job**

```powershell
[root@devops01 ~]# kubectl deltet cronjob hello
```

**该操作不会停止正在运行的pod,不会删除job,不会删除pod**

如果想要删除pod，需要执行删除job操作。



##### StatefulSet

StatefulSet作为Controller为pod提供唯一的标识。它可以保证部署和scale的顺序

statefulset是为了解决有状态服务的问题，其应用场景包括：

* 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于pvc实现
* 稳定的网络标志，即pod重新调度后其PodName和HostName不变，基于Headless Service(即没有cluster IP的Service)来实现
* 有序部署，有序扩展，即pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次进行(从0到N-1，在下一个pod运行之前，其之前的pod必须是Running和Ready状态)，基于init containers来实现
* 有序收缩，有序删除(从N-1到0)

##### Horizaontal Pod Autoscaling

应用的资源使用率通常有高峰和低谷的时候，如何消峰填谷，提高集群的整体资源利用率，让service中的pod个数自动调整。让pod水平自动缩放











