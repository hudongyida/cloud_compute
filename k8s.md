# K8S
## 概念
### pod
最小可部署计算单元
一个pod中可以包含一个或多个容器，pod和pod之间相互隔离
pod和pod间通过API管理和调度
    ReplicaSet：用于创建和管理pod的API
    Deployment:用于管理ReplicaSet的API，可以和ReplicaSet配合达成滚动更新
    StatefulSet: 同Deployment用法相同，但Deployment是用于无状态无顺序的pod管理，StatefulSet是用于有状态有顺序的pod管理
### k8s 网络通讯方案
k8s集群采用扁平网络管理，在集群中存在3条网络，service网络，pod网络，节点网络
1. pod网络
   用于各个pod之间的通信，所有的pod都会链接到此网络
   - 在同一个pod之间的通讯采用lo网络接口+端口实现直接通信
   - 在不同pod，同主机之间使用docker0 网卡实现网络通讯
   - 在不同主机上需要使用Flannel插件实现数据报的再封装转发
      1. 先由Flannel0网卡侦听抓取Docker0 网络上的数据包
      2. 抓取指定的数据包后进行数据包封装，查表转发
      3. 数据包封装完成后有主机的物理网卡将数据包发出
      4. 目标主机的Flannel收到后会将数据包解除封装，发往docker0网卡
      5. 进docker0网络转发到相对用的pod中
2. service网络
   用于对外暴露服务的网络，只有部分的pod会连接到这网络，并且该网络可以连通到外部网络
3. 节点网络
   物理网络，实际链接到各服务器节点的网络
[flannel](./img/flannel.png)
[k8net](./img/k8s_net.png)

## K8S中的资源清单
就是控制k8s中的pod service 等资源的配置和控制文件，一般为yaml文件或者json文件
分类:
- 名称空间级别
- 集群级别
- 元数据


### pod 的生命周期
从Pending开始>running>ucceeded运行成功/Failed运行失败【详细阶段见下文】
- pod create -f 命令创建pod
pod生成过程【一般】
- 生成pasure 网络控制器，实现同一个pod间的lo端口通讯
- Init C 初始化pod容器，Init C可以出现多次
- Start容器启动
- readiness探针检测容器是否成功启动，成功后pod转态转为run
- 在readiness启动同时Liveness同时启动，并且伴随容器全部生命周期，在容器出现错误的时候会发出相对应的请求，停止或重启pod
- 在停止容器的时候会执行stop，确保容器和pod正常关闭
[pod_life_cycle](./img/pod_lifecycle.png)

pod的Phase阶段/状态
|值|描述|
|---|---|
|Pending|pod已经被K8接受，pod容器正在创建或者镜像正在下载|
|Running|已经绑定到了某个节点，pod容器正在运行或者启动，重启状态|
|Succeeded|运行结束，不会重启|
|Failed|pod容器终止，pod中有镜像为成功启动导致pod被迫终止|
|Unknown|无法获取pod状态，pod或pod所在节点和主机失联|



#### Init:
- Init本质上是一个容器，但和主容器不同的是。该容器结束时pod不会结束
- pod在运行的时候只有Init容器运行结束，并且返回值为0【正常关闭】时主容器才会启动
- Init容器可以有多个，但Init容器不能同时运行，必须按顺序依次执行


- 如果Init容器启动失败K8s会不断的重启容器，直到Init容器成功结束或者到达最大重启次数pod状态失败，若容器设置了restartPolicy除外，restartPolicy值为Never时Init为成功结束则pod转态为失败，不会反复重启
- 在Init没有成功之前pod状态为Ready
- pod重启Init也会重新执行
- 在pod.yaml文件中的Init部分的name字段名称不能重复
- Init不存在readiness和Liveness

设置Init 在pod.yaml文件中的initContainers:字段下面设置
```yaml
apiVersion: v1                ##aip版本
kind: Pod                     ##类型
metadata:                     ##跟多原数据信息
  name: myapp-pod             ##pod名称
  labels:                     ##标签
    app.kubernetes.io/name: MyApp
spec:                         ##pod中的容器信息
  containers:                 ##主容器信息
  - name: myapp-container        ##主容器名称
    image: busybox:1.28          ##主容器镜像
    command: ['sh', '-c', 'echo The app is running! && sleep 3600'] ##容器的CMD命令，启动容器
  initContainers:             ##Init容器设置
  - name: init-myservice         ##Init名称设置
    image: busybox:1.28          ##Init容器设置
    command: ['sh', '-c', "until nslookup myservice.svc.cluster.local; do echo waiting for myservice; sleep 2; done"] ##Init容器启动命令
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

#### 探针
用于主判断容器是否成功启动
基础示例
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    ##后面的探针示内容可以代替此行 **Probe：
```

探针的几种探测结果
- Success 成功，容器通过了诊断
- Failure 失败，容器未通过真的
- Unknow 未知，诊断失败，不会采用然后行动

探针有三种类型: 
##### livenessProbe存活探针
- 伴随容器全部生命周期，会根据不同的检测方案对容器进行实时检测
- 若容器出现错误或不健康【例如死锁】便会重启，但可以同配置策略实现多次错误重启或忽略或结束pod。
- 存活探针一般和就绪探针配合使用，但是存活探针不会等待就绪探针成功，可以配置initialDelaySeconds参数实现探针的延迟启动

1. ExecAction Exec检测：在容器内执行指定命令，命令退出时返回0标识成功
```yaml
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
2. HTTPGetAction HTTPGet检测: 对容器发出Http Get请求，转态码在200-400之间为成功
```yaml
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```
3. TCPSocketAction TCP存活探测:尝试和容器的指定端口建立TCP链接，若能建立成功则表示容器健康
```yaml
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
```
4. gRPC 检测:需要容器支持gRPC检测
```yaml
livenessProbe:
      grpc:
        port: 2379
      initialDelaySeconds: 10
```

##### readinessProbe就绪探针
- 该探针会伴随容器的全部生命周期
- 用于检测容器是否能正常工作，能够响应客户端的请求
- 若该探针状态为失败那么该pod将不会出现在服务列表中
- 若该探针状态为成功那么该pod将会被加载到服务列表中
- 该探针不会重启pod或容器，该探针的状态仅仅决定pod是否应该出现在服务列表中

检测方法见下文通用检测


##### startupProbe启动探针
- 该探针仅仅用于容器启动，并且在该探针运行的时候存活探针和就绪探针会被禁用
- 该探针只有成功一个转态，但检测到容器启动成功后该探针的进程就会被结束
- 由于启动探针只有一个状态，所以在设置探针的时候必须设置最大响应时间，超时会杀死该容器并重启

检测方法见下文通用检测

#### 补充【通用检测】
HTTP 探测：
```yaml
livenessProbe:
  httpGet:
    httpHeaders:
      - name: Accept
        value: application/json

startupProbe:
  httpGet:
    httpHeaders:
      - name: User-Agent
        value: MyUserAgent
```

TCP 探测：
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 1
  periodSeconds: 60
```

## 资源控制器
使用K8s中内置的API实现对pod的控制

### ReplicationController&&ReplicaSet
ReplicationController控制器: 旧版 API，现被ReplicaSet AIP代替

ReplicaSetAIP:用于确保任何时候pod都符合一定的副本数量【使pod的数量始终维持在replicas给定的参数，**并且通过标签来时识别哪些pod是需要RS管理的**】
示例:
```yaml
apiVersion: apps/v1        ##版本
kind: ReplicaSet           ##类型为ReplicaSet
metadata:                  ##细节
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:                      ##RS控制参数
  replicas: 3              ##需要维持的容器数量
  selector:
    matchLabels:
      tier: frontend       ##RS的标签信息，需要和下面的pod的标签信息一致
  template:                ##RS调控的pod的信息
    metadata:
      labels:
        tier: frontend     ##pod的标签，这里的标签要和RS中的标签一致，不然无法接受RS的控制
    spec:                  ##pod中的inage镜像信息
      containers:
      - name: php-rediss
        image: nginx:v5
```

可以使用下面命令查看
```bash
kubectl get rs
```

### Deployments
通过管理RS API来实现对pod的管理，Deployments无法直接创建pod
Deployments可以通过RS实现滚动更新，回滚，实时扩缩容
示例:
```yaml
apiVersion: apps/v1             ##版本
kind: Deployment                ##创建类型Deployment
metadata:
  name: nginx-deployment        ##名称
  labels:                       ##标签
    app: nginx
spec:                           ##设置RS的信息
  replicas: 3                   ##需要维持pod数量
  selector:
    matchLabels:
      app: nginx
  template:                     ##pod模板信息
    metadata:
      labels:
        app: nginx
    spec:                       ##pod镜像信息
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
必要字段:
.spec  .spec.template和.spec.selector


Deployments 更新触发
当通过命令修改了yaml文件下面的spec.template的值变会触发Deployments更新
示例；
```bash
##使用命令直接对Deployments spec.template
kubectl set image deployment/名称 nginx=nginx:1.16.1
##使用命令对Deployments进行更新
kubectl edit deployment/名称
```
更新流程：
1. 修改Deployments 触发更新
2. Deployments API收到更新后会建立一个新的RS
3. RS开始逐步创建pod在RS创建pod的同时 旧版的RS会开始逐渐删除pod【遵循25-25原则，及先创建25%的新pod再移除原RS的25%的pod，以此往复】直到更新完成

Deployments回滚触发:
同样需要修改yaml文件下面的spec.template值才出发更新
```bash
##获取Deployment描述信息
kubectl describe deployment
##检查Deployment上线历史
kubectl rollout history deployment/名称
##回退上一版本
kubectl rollout undo deployment/名称
##回退特定版本
kubectl rollout undo deployment/名称 --to-revision=2
```

设置Deployment缩放【扩缩容】
在必要时Deployment会新建RS和pod来分担
```bash
##扩容到10个
kubectl scale deployment/名称 --replicas=10
##设置自动扩容，当CPU占用率到80的时候自动触发扩容，最小10个pod，最大15个pod
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

暂停、恢复 Deployment 的上线过程
在服务上线的过程中可以被暂停
```bash
##暂停上线的命令
kubectl rollout pause deployment/nginx-deployment
##恢复
kubectl rollout resume deployment/nginx-deployment
```
在服务上线的过程中【服务没完全上线】触发更新，那么Deployment会直接杀死旧的RS立刻创建新的RS

#### Deployment 状态
进行中Progressing：正在创建RS或者扩缩容
完成Complete: RS操作完成
未完成: 可能因为配额不足，就绪探针失败，镜像拉取失败，权限不足等

### DaemonSet
一种基础的设施pod每个node节点只能运行一个DaemonSet副本，用于监测各节点的运行情况
示例: 
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # 这些容忍度设置是为了让该守护进程集在控制平面节点上运行
      # 如果你不希望自己的控制平面节点运行 Pod，可以删除它们
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      # 可能需要设置较高的优先级类以确保 DaemonSet Pod 可以抢占正在运行的 Pod
      # priorityClassName: important
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```
### Job&&CronJob
#### Job
job 一次性任务，运行运行完成后便会停止
示例：
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```
#### CronJob
利用Linux的Cron功能呢实现job任务的定时执行

## service服务 API
是K8s中的一种抽象，可以将pod的服务对外发布,并且可以实现4层的负载均衡
service是一个对象可以用K8s AIP 来定义service 
示例:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
```

## 存储AIP
### ConfigMap
ConfigMap 提供了一种机制来实现配置的集中管理和复用，一般是用在将配置文件从程序代码中解耦，可以更好的设置和部署应用程序
