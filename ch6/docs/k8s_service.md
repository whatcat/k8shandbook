### Service的整理

**代理模式分类**

#### iptables 代理方式

#### ipvs代理方式

#### 服务分类

##### ClusterIP



##### NodePort



##### LoadBalance

云厂商提供

##### headless service

无头服务

##### ExternalName

引入外部服务使用的

#### 代码实践模块

创建`deployment`模块

```yaml
# 部署deployment服务
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 10
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
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
      dnsPolicy: ClusterFirst
      containers:
      - name: myapp
        image: nginx:1.16.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        readinessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

##### ClusterIP

部署`ClusterIP`服务

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-deploy-cluster-svc
  namespace: default
spec:
  type: ClusterIP
  selector:  #标签选择器主要是去匹配了template.metadata.labels下的标签
    app: myapp
    release: stabel
    env: test
  port:
  - name: http
    port: 80
    targetPort: 80
```

验证操作,因为网络是扁平的，所以执行的验证操作使用的是`curl`

```powershell
[root@devops01 ch6]# kubectl get svc 
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes                 ClusterIP   10.96.0.1        <none>        443/TCP   4d3h
myapp-deploy-cluster-svc   ClusterIP   10.111.189.114   <none>        80/TCP    2m
[root@devops01 ch6]# curl 10.111.189.114:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
```

验证操作2，使用的是`coredns`提供的域名方式来验证。操作如下

使用`busybox`镜像

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
  namespace: default
spec:
  selector:
    matchLabels:
      name: busybox
  template:
    metadata:
      labels:
        name: busybox
    spec:
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      containers:
      - name: busybox
        image: busybox:latest
        imagePullPolicy: Always
        command:
          - sleep
          - "3600"
      
# 命令行方式
[root@devops01 ch6]# kubectl exec -it busybox-5db8999685-smk7d -- nslookup myapp-deploy-cluster-svc.default.svc.cluster.local.
Server:         10.96.0.10
Address:        10.96.0.10:53
```

##### NodePort

部署`nodeport`服务，`nodePort`的原理在于在node上开一个端口，将该端口的流量导入到`kube-proxy`，然后由`kube-proxy`进一步导给对应的`pod`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-deploy-nodeport-svc
  namespace: default
spec:
  type: NodePort
  selector:
    app: myapp
    env: test
  ports:
  - name: http
    port: 80
    nodePort: 30001
    targetPort: 80
```

查看操作

```powershell
[root@devops01 ch6]# curl 10.99.192.72:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

[root@devops01 ch6]# curl 127.0.0.1:30001
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

[root@devops01 ch6]# curl http://47.107.107.186:30001/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# 端口查看
[root@devops01 ch6]#  netstat -anpt | grep 30001
tcp        0      0 0.0.0.0:30001           0.0.0.0:*               LISTEN      28398/kube-proxy 
```

##### headless service

有时不需要或者不想要负载均衡，以及单独的`Service IP`。遇到这种情况，可以通过制定的`Cluster IP（spec.clusterIP）`的值为"None"来创建`Headless Service`。这类Service并不会分配Cluster IP. kube-proxy不会处理他们，k8s也不会为他们进行负载均衡和路由

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: default
spec:
  selector:
    app: myapp
  clusterIP: "None"
  ports:
  - port: 80
    targetPort: 80

[root@devops01 ch6]# yum -y install bind-utils

[root@devops01 ch6]# dig -t A myapp-headless.default.svc.cluster.local. @10.96.0.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.2 <<>> -t A myapp-headless.default.svc.cluster.local. @10.96.0.10
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 22353
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;myapp-headless.default.svc.cluster.local. IN A

;; ANSWER SECTION:
myapp-headless.default.svc.cluster.local. 30 IN A 172.17.0.9
myapp-headless.default.svc.cluster.local. 30 IN A 172.17.0.8
myapp-headless.default.svc.cluster.local. 30 IN A 172.17.0.10

;; Query time: 0 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Fri May 08 17:13:25 CST 2020
;; MSG SIZE  rcvd: 237

```

##### LoadBalancer 负载均衡

##### ExternalName

```yaml
apiVersion: v1
kind: Service
meatdata:
  name: my-service-ex
  namespace: default
spec:
  type: ExternalName
  externalName: www.baidu.com
 
 # 查看操作
[root@devops01 ch6]# dig -t A my-service-ex.default.svc.cluster.local. @10.96.0.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.2 <<>> -t A my-service-ex.default.svc.cluster.local. @10.96.0.10
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48448
;; flags: qr aa rd; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;my-service-ex.default.svc.cluster.local. IN A

;; ANSWER SECTION:
my-service-ex.default.svc.cluster.local. 30 IN CNAME www.baidu.com.
www.baidu.com.          30      IN      CNAME   www.a.shifen.com.
www.a.shifen.com.       30      IN      A       14.215.177.38
www.a.shifen.com.       30      IN      A       14.215.177.39

;; Query time: 1 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Fri May 08 17:29:00 CST 2020
;; MSG SIZE  rcvd: 241

```









