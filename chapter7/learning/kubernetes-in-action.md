## 【阅读笔记】Kubernetes in Action

---

出版时间：2018年

书本长度：628页

载体：PDF电子版

语言：英文版

开始阅读时间：2019年4月3日

完成阅读时间：2019年8月18日

tag：阅读笔记，重点内容摘录，kubernetes

主要收获：对k8s有了一个比较完整的了解，知道了重要特性及使用的最佳实践，能做到选择最佳的方案来使用k8s，方便与需要上k8s的业务相结合。

---

重要前提：

第一部分：Overview --P1/33

* 第一章：k8s介绍 -P1/33
* 第二章：Docker和K8S相关的入门步骤 --P25/57

第二部分：核心组件 --P55/87

* 第三章：Pod --P55/87
* 第四章：复制集及其他控制器：管理Pod --P84/116
* 第五章：服务，让client发现并连接Pod --P 120/152
* 第六章：挂载存储到容器 --P159/191
* 第七章：ConfigMaps和Secrets，配置应用 --P191/223
* 第八章：从应用中访问Pod的metadata和其他资源 --225/257
* 第九章：deployment：陈述式地更新应用 --P250/282
* 第十章：StatefulSets：部署多副本、有状态的应用 --P280/312

第三部分：基础之外的内容 --P309/341

* 第十一章：理解k8s内部 -P309/341
* 第十二章：让k8s的API server变得安全 --P346/378
* 第十三章：让k8s集群的node及网络变得安全 --P375/407
* 第十四章：管理Pod的计算资源 --P404/436
* 第十五章：自动扩展Pod和集群的Node --P437/469
* 第十六章：高级调度 --457/489
* 第十七章：开发APP的最佳实践 --P477/509
* 第十八章：扩展Kubernetes --P508/540

---

### 重要前提： {#k8s-qt}

本书主要内容描述Kubernetes 1.9.0版本

### 第一部分：Overview --P1/33 {#k8s-part1}

#### 第一章：k8s介绍 -P1/33

Linux Namespace实现资源的隔离

Linux control groups（cgroups）实现资源限制 --P11/43

如果app依赖某个内核功能（或者模块），则app不一定能任意在Docker上运行；如果app是针对ARM硬件架构的，不一定能在x86架构的Docker上运行。但是这些情况，通过VM都可以解决。 --P15/47

#### 第二章：Docker和K8S相关的入门步骤 --P25/57

所有的image都是由名字：tag组成的，如果不指定tag，则默认是latest这个tag --28/60

k8s一个很基础的原则：只要告诉k8s你想要什么，不需要告诉k8s具体要做什么（declarative） --P49/81

### 第二部分：核心组件 --P55/87po {#k8s-part2}

#### 第三章：Pod --P55/87

一个Pod永远在一个node上，不可能跨越两个node

3.1.1、为什么需要Pod呢？

多个进程如果运行在同一个container内，会有很多问题：管理进程的启停、日志搜索等都需要自己去处理和解决

有一定相关性的container运行在一起，同时又有一定的隔离性

同一个Pod内的Container共享Network/UTS/IPC namespace，共享相同的主机名、网卡，可以通过IPC进行进程间通信。

Pod之间都是可以通过IP互联互通的，没有经过任何的NAT处理

将多个进程拆成不同的Pod的好处：

充分利用多个node的资源，提高资源利用率

方便每个Pod根据自己的scale requirement特点做扩展（scaling）

多个容器放到一个Pod内：

一个主容器+多个辅助容器，比如内容更新、日志切割、日志搜集、数据处理、通讯组件等

判断多个进程是否需要放到同一个Pod，问自己以下问题：

* 是否需要运行在一起，还是可以在不同的host上运行？
* 他们是一个整体？还是独立的组件？
* 他们需要一起scale还是单独的？

61/93

编写一个yaml文件，一般分为四个部门：

* apiVersion
* kind
* metadata
* spec

port-forward可以把本地端口转发到Pod的端口上，比如：

kubectl port-forward kubia-manual 8888:8080

将localhost:8888转发到kubia-manual这个Pod的8080端口上；很方便用来做测试等工作

label：任意的key-value对，用来分类管理pod和其他k8s相关资源

annotation（翻译为：注释）可以是很长的数据（最多256KB）

namespace：不同的namespace允许存在相同名字的资源；实现权限和资源管理的隔离；

注意：namespace本身并不意味着pod之间的网络隔离

删掉一个Pod：发送SIGTERM-》默认30s-》发送SIGKILL，所以进程需要处理好SIGTERM来优雅地结束

删除某个namespace下所有的资源（Pod，RC，Service等）

$ kubectl delete all --all

第一个all代表所有的资源类型

第二个--all代表所有的资源instance，而不是某个名字或者label

注意：Secrets这种资源就不会被删除，需要明确指定才能删除。

#### 第四章：复制集及其他控制器：管理Pod --P84/116

liveness probes（判断pod是否healthy）：不依赖app进程本身是否存活或者信号，而是从外部、功能上进行判断；用来检测和判断Pod里面的container到底是不是正常的（健康的），以此来决策是否需要重启（对容器发送SIGKILL 9信号，强制杀死进程）容器之类的（Pod本身不会被重启）

目前有两种判断方式：

* * HTTP GET（返回2xx或3xx代表成功）
  * TCP Socket建立连接方式（连接建立成功代表成功）
  * Exec方式，运行任意命令（返回值是0代表成功）

看老的、被重启过的容器的日志：

$ kubectl logs mypod --previous

生成环境建议都配置有用的liveness probes，否则k8s不知道如何判断Pod是否正常

delay=0s timeout=1s period=10s \#success=1 \#failure=3

最好要设置好initialDelaySeconds，要考虑到app的启动时间

liveness probe最好是能通过一个URL（比如/health），检查内部的所有重要的组件，确保后端是真正的健康

liveness probe不要太消耗资源（比如不要启动JVM虚拟机）

liveness probe不用考虑循环或者重复，因为k8s本身会重复调用liveness probe的

liveness probe以及重启容器，是通过节点上的Kubelet进行的

ReplicationController包含三个部分：

* label selector
* replica count
* pod template

只修改rc的label selector或者pod template，不会影响已有的Pod（但是已有的Pod会不再由ReplicationController管理）

API server会拒绝RC资源中label selector与template内label不一致的情况，用以避免陷入无限创建新的Pod的境地（推荐的、更简单的办法是：直接不谢selector，这样就会直接用template里面的label了） --P94/126

修改Pod的label，可能会使得某个Pod不再受RC的管控，而且RC会重新创建新的Pod来代替。

修改RC的template，只会影响新创建的Pod，老的Pod继续维持原状。

以上讲的ReplicationController目前已经被ReplicaSet替代了，ReplicaSet有很多更灵活的功能（比如更多的label selector选项；key是否存在；与和非等）

DaemonSet会bypass Scheduler的

DaemonSet + Node Selector

Job用来运行短期的任务，

batch

API group and

v1

API version

completions：让Job串行运行多个Pod并运行（默认是1，就是指运行一次）

parallelism：Job中多个Pod最大并发运行数量（默认是1，就是串行）

CronJob：类似linux/unix的定时任务，会根据时间策略创建Job来执行任务

#### 第五章：服务，让client发现并连接Pod --P 120/152

* Pod是短暂的，可能被删除和创建
* Pod的IP不是提前预知的，也不知道分配到那个node上
* Pod会有横向伸缩，导致数量不确定

Cluster IP只能在集群内访问到（例如：从node节点、Pod内部来访问）

k8s的Service只支持两种sessionAffinity:None和ClientIP；因为Service是工作在TCP/UDP层面的，所以无法支持类似cookie的session affinity，因为它无法识别HTTP协议。

创建Service的时候，是可以支持多个port的，但是在多个port的情况下，每个port需要给一个名字

创建Pod的时候，port也是可以赋予名字的

Pod可以通过环境变量获取相关Service的Cluster IP和Port

也可以通过DNS的FQDN来访问：

backend-database.default.svc.cluster.local

backend-database是service name

default是namespace

svr.cluster.local是可配置的、整个集群级别的、统一的后缀

如果是在同一个namespace，default和svr.cluster.local都可以省略，直接用backend-database即可访问服务了

特别注意：Cluster IP是ping不通的，因为它是虚拟IP，只有和service port结合才有用 -P 131/163

EndPoints是介于Service和Pod之间的一层，它包含了所有后端Pod的IP:Port对

通过单独创建Service（没有pod selector）和EndPoints（把address和port指向外部的IP即可），可以创建针对外部服务的Service，这样就可以利用k8s本身提供的负载均衡以及服务发现等特性了 --P133/165

通过调整Service资源的selector（间接控制EndPoints是否自动或手动维护），可以将服务的访问地址保持一致 --P133/165

通过使用ExternalName，也可以直接创建指向外部服务的内部Service（一个简单的CNAME，就不走proxy转发了）

另外一个重要问题，从外部访问集群内的服务，主要有以下三种方式：

* NodePort
* LoadBalancer
* Ingress

jsonpath 过滤和获取想要的Pod的信息

LoadBalancer是在NodePort上面再包了一层，所以使用LB的场景下，NodePort也是启用了的，机器上也有端口可以访问

spec: externalTrafficPolicy: Local

通过配置以上annotation，可以禁止NodePort（包括LB）将流量转发到其他Node上的Pod上去

\* 如果本机没有Pod，整个连接会hang住。

\* 可能造成Pod之间的不均衡（Node级别是均衡的） --P 142/174

通过上面的配置：NodePort模式下，Pod可以获取到client的访问源IP（因为不需要SNAT，其他情况下会SNAT） （（Host网络模型，也可以直接获取到client的源IP地址））

Ingress不会把请求转发给Service，而是通过Service获取Pod的IP，直接转发给Pod

readiness probes三种类型：

* Exec
* HTTP GET
* TCP Socket

readiness probes不会杀掉Pod、启动新的Pod，只会从EndPoint里面移除异常的Pod

headless service --P154/186

不分配ClusterIP，请求DNS的时候直接返回所有的Pod的IP

kubectl run来启动Pod，会产生一个RC（RS）来管理Pod，加上-generator=run-pod/v1参数就直接创建Pod了，跳过RC（RS）

#### 第六章：挂载存储到容器 --P159/191

emptyDir可以设置为Memory medium，这样就存储在内存里面了 --P166/198

gitRepo类型的volume，不会实时同步，只有在Pod创建的时候才会同步

sidecar container：一个帮助主容器运行的的容器

先阶段，gitRepo volume还不支持使用私有的仓库

hostPath：访问宿主机的本地磁盘

PersistentVolume和PersistentVolumeClaim，将对存储的需求和底层的存储技术解耦；将底层存储技术的配置交给k8s集群管理员，而应用开发人员（即k8s使用者）只需要提需求即可

PersistentVolume是集群级别的资源，不受namespace限制

PVC的AccessMode

* RWO—ReadWriteOnce
* ROX—ReadOnlyMany
* RWX—ReadWriteMany

注意：此处的Once和Many是指的Node，而不是Pod

干净的PV（以前没有被PVC关联过的、状态是AVAILABLE的）在PVC创建的时候就BOUND了

persistentVolumeReclaimPolicy有三种策略（每种不同的存储会有不同的策略）：--P183/215

* Retain：如果你删除一个PVC，相关联的PV不会被删除，也不能再次被使用，数据会保留，状态是Released，但是真实的存储不会被删除掉。
* Recycle：删除数据，然后可以被PVC重新关联
* Delete：删除PVC时，直接删除底层的存储

StorageClass：动态地创建PV，避免上面那样需要集群管理员手动创建PV --P185/217

如果有设置默认的StorageClass，即使没有写StorageClass，默认的StorageClass会比已存在的PV优先级更高，即：即使已经有PV能满足条件，也会动态地创建新的PV。如果要禁止StorageClass创建，而是直接使用已有的PV，需要明确指定storageClass为空（即：“”） --P189/221

可以设定一个default（默认的）StorageClass

#### 第七章：ConfigMaps和Secrets，配置应用 --P191/223

配置应用的方式：

* 容器加命令行参数
* 为容器配置环境变量
* 把配置文件通过volume的形式mount进容器

Docker下：

ENTRYPOINT：是容器启动时候的命令

CMD：是容器启动时候命令后面加的参数，可以没有参数，也可以覆盖默认的参数

ENTRYPOINT下，一般有两种格式：

* SHELL格式：ENTRYPOINT node app.js
* EXEC格式：ENTRYPOINT \["node", "app.js"\]

SHELL格式，PID1进程是shell，而且会多一个shell进程，一般不建议使用

Docker的ENTRYPOINT和CMD分布可以被k8s的command和args强制覆盖

环境变量是设置在container级别的，不是Pod级别

ConfigMap： --P202/234

* --from-literal：在命令行指定key=value
* --from-file：value就是文件的内容，文件名就是key；也可以指定目录，一次性创建多个key=file的map了

ConfigMap也可以直接映射成文件挂载到container里面去（类似nginx的conf.d目录里面的文件）

ConfigMap的volume使用，要注意：如果是目录，目录内以前的文件不可访问了（类似Linux文件系统）。使用mountPath+subPath可以把单独的一个configMap key映射到某一个单独的文件，避免覆盖整个目录。

默认，ConfigMap挂载的文件的权限是644，可以自定义修改。

通过文件volume的方式mount ConfigMap到目录的，ConfigMap更新了，文件也会更新（可能有延迟，可能长达一分钟）但是注意，程序是否生效（如：reload文件），需要单独reload程序的。 --212/244

挂载目录：是通过软链接实现的，所以更新一定是原子操作。但是不同的Node更新文件不是同步的，不同Node之间更新文件可能有延迟（可能长达一分钟）。

挂载单独的文件：ConfigMap更新，文件也不会更新！

Secret存储在etcd中，且是加密存储的（加密算法未知）。

Secret只会被分发到需要的Node上，而且只存储在内存里，不会落地到磁盘。 --P214/246

所有的Pod默认都有一个default-token，用来和k8s API通讯

Secret的key/value内容，value是Base64 encoded，可以支持binary数据，最大不超过1MB。

Secret最好不要通过环境变量进入container（日志、报错、子进程继承环境等，可能导致泄密），建议使用secret volume来访问Secret --p222/254

Secret类型：

* general
* docker-registry

把Secret添加到ServiceAccount，可以避免每个Pod都需要指定imagePullSecrets的麻烦 --P223/255

#### 第八章：从应用中访问Pod的metadata和其他资源 --225/257

Downward API，通过环境变量或者downward API volume，把Pod及环境相关的信息暴露给应用，比如pod的IP、Pod的主机名、host的主机名、Pod的label和annotation等。 230

Pod的label以及annotation只能通过volume的形式挂载进container（原因在于：label和annotation在Pod运行的时候是可以修改的，所以通过volume能及时更新；但是通过environment方式无法获得更新） --P233/265

注意：volume是Pod级别的，所以如果container级别的metadata（比如resource request 和 limit）需要暴露出来，volume上必须写清楚container的名字。 --P233/265

使用kubectl+curl访问API：启用kubectl proxy，通过HTTP的形式，代理访问HTTPS的kubernetes API --235/267

从Pod内访问API，三项注意： --P239/271

* 找到API的地址（kubectl get svr）
* 如何确保访问的真的是我们的API？\(cacert证书\)
* 如何与API服务进行认证/授权（token）

#### 第九章：deployment：陈述式地更新应用 --P250/282

imagePullPolicy的默认策略依赖镜像的tag的：如果tag是latest（明确指示或者完全不加tag的情况下），则默认策略是Always；如果tag是某个值（比如v1），则默认策略是IfNotPresent。 --P256/288

kubectl rolling-update的问题：

* 会修改RS的label
* 依靠kubectl客户端进行所有操作，维持状态，调用API实现，容易受网络影响（使用Deployment就是在Server端处理的）
* 是命令式的（imperative），而非描述式的（declarative）

Deployment通过协调多个ReplicaSets来管理最终的Pod

Deployment管理版本，它在版本之上，所以它的名字一般不要反映出版本信息。 --P263/295

Deployment默认的升级策略是RollingUpdate，还有一个是Recreate（一次性删除老的Pod，重新创建新的Pod） --P

通过记录多个历史版本（revisionHistoryLimit参数可以调整数量），使用rollout undo命令，可以很快的回滚到上一个版本或者指定的历史版本。 --P271/303

maxSurge：允许超过desired数量的Pod最大值

maxUnavailable：允许低于desired数量的Pod最大值，可以是0

可以pause和resume一个rollout过程，间接实现canary release（金丝雀发布\)；当然，目前最合理的canary release是部署两个Deployment来实现。 --P274/306

readiness probe+minReadySeconds：相当于是双保险，readiness probe如果在minReadySeconds之前报错，那么整个rollout是不会进行下去的（被blocked）。minReadySeconds：确保container的readiness probe成功后，保持一段时间都是成功的，才继续整个rolling update。 --P274/306

#### 第十章：StatefulSets：部署多副本、有状态的应用 --P280/312

ReplicaSets下创建的Pod都是一样的，所以挂载的目录也是同一个volume，共享volume数据。

StatefulSets:

* 每个副本有自己独立的PV存储
* 每个副本可以保持一致的identity（比如Pod name，hostname，Pod IP等）
* 每个Pod的内部DNS域名也保持一致

StatefulSet下的Pod，相对来讲比较可预期一点，比如scale up和scale down都是按数字顺序来的

StatefulSet只允许一个Pod接一个Pod地scale down；而且，当有一个Pod是unhealthy的时候，不允许做scale down的操作。

scale down的时候，PVC不会被删除；当重新scale up的时候，没有被删除的PVC可以被自动重新挂载上去。

yaml文件中，三个dash\(---\)分割多个资源；也可以用List object来处理，两者是一样的。 --P292/324

StatefulSets能够被访问到，需要一个Headless service作为GOVERNING Service；主要是因为headless service运行Pod之间的peer discovery --P292/324

创建StatefulSet的时候，Pod也是一个接一个启动的，不会像RS那样一次性启动。

对一个headless service，在k8s集群内（在一个Pod上）发起一个SRV DNS请求就可以获取到所有后端的Pod的DNS及IP信息（peer discovery） --P300/332

StatefulSet的副本所在的node如果出行问题了，k8s不会自动替换，因为（most-one）需求，必须要管理员手动告诉k8s（比如删除Pod，或者删除Node），才会替换有故障的副本

### 第三部分：基础之外的内容 --P309/341 {#k8s-part3}

#### 第十一章：理解k8s内部 -P309/341

管理控制台组件：

* etcd分布式、持久化存储
* API server
* Scheduler
* Controller Manager

Worker Node的组件：

* Kubelet
* Kubernetes Service Proxy\(kube-proxy\)
* Container运行时（Docker,rkt等\)

其他Add-On组件：

* DNS server
* Dashboard
* Ingress Controller
* Heapster
* Container Network Interface network plugin

所有的系统组件，都只通过API server进行交互和沟通，而不是互相通讯；API server和etcd通讯，其他的组件都不直接访问etcd（主要是防止产生inconsistent data）。 --P311/343

etcd和API server可以同时又多个在活动，但是Scheduler和Controller Manager只能有一个是active的，其他的只能是standby。

所有的系统组件本身，也可以运行在Pod中，但是Kubelet永远都是一个常规的系统进程。

etcd只和API Server通讯：乐观锁、认证更健壮、屏蔽底层存储细节（更容易替换）等好处。

乐观锁：通过metadata.resouceVersion的传递和校验来确定资源是否已经被修改过。 --P313/345

所有的k8s数据都存放在etcd的 /registry目录下

etcd的多个实例之间，通过RAFT consensus 算法保证一致性（多数节点才能进入下一个状态，少数节点只能返回上一个正确状态的数据，不能变更进入下一个状态）

etcd一般要求时奇数个实例部署：因为偶数实例部署，反而增加了故障的风险，而在健壮性上的收益和减少一个节点是一样的（参考3vs4，3节点需要2个quorum，4节点需要3个quorum，都只能忍受一个节点失效;失效后，如果再失去一个节点，整个集群也都进入不可用状态了）。 --P316/348

大集群一般建议使用5个或者7个节点，理论上足够了

Admission Control Plugin：主要用来验证相关的数据操作（写、删除、新建等）是否合理： --P317/349

* AlwaysPullImages
* ServiceAccount
* NamespaceLifecycle
* ResourceQuota

--watch：API server支持watch功能，其他的组件可以通过watch某个资源的变化来获得相关信息。

Scheduler、Kubelet都是通过watch API server来获得需要做的操作的信息的

默认schedule algorithm：过滤出可以接受的node列表 ，然后对剩下的打分、排序

Pod里面可以在spec里面设置schedulerName属性，可以使用其他的Scheduler，也可以自己写一个Scheduler；也可以部署多个默认的Scheduler，但是使用不同的配置文件。 --P321/353

每种资源基本上都在Controller Manager里面有对应的controller，controller不光watch变化，还会隔段时间re-list所有的操作，以避免丢失需要做的事情。 --P322/354

Kubelet本身也可以直接通过Pod的manifest文件创建静态的Pod，k8s的服务组件如果跑在Pod里面，就是通过这种方式实现的。 --P326/358

kube-proxy：两种模式：userspace模式（client -&gt; iptables -&gt; kube-proxy -&gt; pod\)以及iptables模式（client -&gt; iptables -&gt; pod） --P328/360

Ingress controller虽然后端是指向Service，但是实际是把流量直接传递给Pod的 --P329/362

kubectl get events --watch：用来盯着cluster在做什么，比较方便

Pod的pause容器是第一个创建的，它来hold住所有的namespace之类的信息；如果它被Kill，整个Pod下所有的容器都会被重新创建。 --P334/366

同一个node上的container，他们的veth pair都挂在同一个bridge上，所以通讯很简单，可以直接通讯。 --P337/369

跨Node的container需要互访，如果是用plain L3+路由模式，需要在同一个交换机下才能实现（因为container的IP是private的），当然，也可以把中间路由器配置，但是太过于复杂了。所以，使用SDN会比较好，无论中间的网络架构如何，Node都看上去是连接在同一个交换机上（通过包的封装实现的） --P338/370

Container Network Interface \(CNI\)

* Calico
* Flannel
* Romana
* Weave Net
* And others

Service的ClusterIP是一个虚拟IP（没有和任何主机、veth绑定，也没有任何服务），无法ping通；它只存在云kube-proxy控制的iptables规则中，用来做DNAT，指向后端的某个Pod。 --P341/373

通过sidecar模式，k8s可以协助app做leader election。 --P341/373

API Server前面用一个LB来做负载均衡或者高可用，都是可以的，因为它整体上是无状态的；但是Scheduler和Controller Manager一次只能有一个在active，所以他们内部默认实现了--leader-elect来选举leader，只有leader才会干活，其他默认是standby状态（选举过程是通过写一个kube-scheduler的EndPoint资源实现的，期间通过乐观锁来确保只有一个写入成功）。 --P343/375

#### 第十二章：让k8s的API server变得安全 --P346/378

每一个Pod都会有一个ServiceAccount，如果没有指定就使用default；SA的token使用的是JSON Web Tokens。

通过SA的Mountable secrets，可以控制Pod可以mount哪些Secrets。 --P350/382

一个Pod的ServiceAccount必须要有，而且后期是不能修改的。

k8s除了RBAC之外，还有Attribute-based access control \(ABAC\)等plugin；从k8s 1.8起，RBAC就默认启用了，所以Pod内默认的token不能再访问API server了（读写都不行）

RoleBinding虽然也是某一个namespace相关的，但是它可以关联到另外一个namespace的ServiceAccount。 --P362/394

API server重启的时候，默认的ClusterRoles和ClusterRoleBindings会被重新创建

#### 第十三章：让k8s集群的node及网络变得安全 --P375/407

hostNetwork = true，这样Pod就使用Node的网卡namespace了，而不是自己的虚拟网络namespace；k8s的control组件，通过Pod模式部署，就是走的这种模式。

hostPort模式：主机的端口直接转发给本机后面的Pod（注意：不同于NodePort）；类似的还有hostPID以及hostIPC，分别用来查看node的进程或与node上的进程通过IPC通讯。

security-Context可以配置很多和container相关的安全信息：

* 设置container内进程的uid
* 防止以root用户运行
* 在priviledged mode下运行，有完全的对node内核的控制权（比如运行kube-proxy的Pod，因为需要修改node的iptables；比如需要访问node的/dev目录等）
* 增加或去掉一些capabilities（比如CAP\_SYS\_TIME，透过Pod修改所在Node的时间）
* SELinux
* 防止进程写入container的文件系统（只能写mounted volume，防止存放在image的文件系统中的代码被修改）这个值一般是推荐设置的

fsGroup和supplementalGroups：两个使用不同的Uid运行的container，可以通过mounted volume实现文件的共享（共享读写）

PodSecurityPolicy+PodSecurityPolicy admission control plugin可以管控用户可以授予Pod的权限，防止用户滥用上述权限给Pod，造成整个集群的危险。 --P390/422

修改PodSecurityPolicy对已有的Pod不产生影响，因为只有在Pod创建或更新的时候PodSecurityPolicy才有效果。

PodSecurityPolicy可以覆盖一个container里面写死的uid（通过Dockerfile的USER directive\)

PodSecurityPolicy关于CAP的几点功能：

* allowedCapabilities：（允许）用户可以加到Pod里面的CAP列表
* defaultAddCapabilities：（自动）每一个Pod会自动添加的CAP列表（还是可以通过drop操作报这个CAP drop掉的）
* requiredDropCapabilities：（自动）从每一个Pod的能力中drop掉这些（如果用户有add这些CAP，整个Pod会被reject掉）

不同的PodSecurityPolicy通过ClusterRole以及ClusterRoleBinding与user/group等绑定 --P398/430

kubectl config set-credentials &lt;name&gt;

：可以设置多个context；方便使用不同的用户（或token/certificates）来创建资源。

NetworkPolicy：（通过PodSelector或者ipBlock等）管理Pod的入（ingress）及出（egress）的访问权限控制（需要CNI plugin之类的网络插件支持，否则不会有效果）。 --P399/431

#### 第十四章：管理Pod的计算资源 --P404/436

requests（最低资源需求）和limits（最高使用资源限制）都是container级别的限制，不是Pod级别的。

200m（200 millicores，相当于五分之一个cpu core）

如果没有requests，Pod的资源容易被其他的Pod抢占。

Scheduler也是按照Node上的所有container的requests的值来调度新的Pod的，而不是当前Node的实际资源使用情况。Scheduler只参考requests，不管实际使用的。

container的requests，在没有设置limits的情况下，多个container抢占空闲资源，也会因为requests的大小比例来分配空闲资源。 --P411/443

k8s支持给Node添加自定义的资源，可以让k8s根据自定义的资源的情况来选择Node

如果你只设置了limits而没有设置requests，则默认requests会和limits一样的值。 --P413/445

limits是可以超过Node的资源的，当真正使用的资源超过Node所有的资源时，会有Pod会被kill；kubelet杀Pod，间隔会翻倍地增加（马上，20s，40s，80s，一直到300s）

要注意：目前很多进程是看不到Pod的Limits的，如果是动态分配资源（如Java的Heap内存、某些进程启动时候的线程数会根据CPU个数来确定等），会根据Node的资源来设置一些参数，这种类型的进程上k8s要特别注意，很容易误伤（OOMKilled之类的）

k8s提供三种QoS的资源保障机制：

* BestEffort
* Burstable
* Guaranteed

Pod里的container完全没有设置requests和limits的，自动分配BestEfforst机制；

一个Pod内所有的container、CPU和内存都设置了requests和limits的，且requests和limits一致的（也可以只设置limits），则自动分配Guaranteed； --P418/450

如果Pod内的container里面的QoS不一样，则Pod级别是Burstable，container的QoS完全一样，则Pod的QoS级别就是container的QoS class

同一QoS级别的Pod，在Burstable QoS Class上，按实际/requests的内存比例，谁用的比例最高就先kill谁，造成OOMKilled。 --P421/453

为了避免对每个container都需要设置request和limits，有一个LimitRange的资源，主要两个功能：

* 设置requests/limits的最大值和最小值范围，超过这个范围的Pod会被拒绝。
* 为没有设置requests/limits的Pod赋予默认的限制值。

通过LimitRange Admission Control plugin来做实现限制的。

可以设置CPU、内存、request/limit的比例、PVC等到的最大最小值等。

整个Pod以及单独的container都可以分别设置requests/limits的限制。

LimitRange只限制单个Pod的资源，不限制namespace下所有的Pod的资源；要限制namespace下所有的资源，需要使用ResourceQuota资源来处理。

---

如果某种资源设置了ResourceQuota，那么创建Pod的时候必须要有requests及limits，否定会被拒绝。

cAdvisor是和Kubelet集成在一起的；Heapster则需要单独安装的addon组件；都只能保存很短一段时间；需要InfluxDB来长期保存数据。

#### 第十五章：自动扩展Pod和集群的Node --P437/469

HorizontalPodAutoscaler \(HPA\)依赖Heapster和cAdvisor采集和汇聚的数据，才能正常执行功能。

当用多个metric来做autoscale的时候，对每个metric单独计算需要多少个Pod，然后取最大值作为最终的Pod数量即可。

HPA只会修改Scale resource，从而达到修改相应的资源（Deployments，ReplicaSets，ReplicationControllers，StatefulSets等）的replica数量的目标； --P440/472

HPA需要Pod设置有requests（或者通过LimitRange资源），HPA只关心Pod的被保证的资源数量的利用率。 --P442/474

HPA对于升降Pod数量的个数，以及每次升降Pod操作的间隔都是有规定的，以减少频繁波动。 --447/479

不光有Resource作为判断metric的HPA，也可以根据Pod（比如QPS）和Object（比如Ingress的延迟）等来做HPA判断的metric --P450/482

idling和un-idling：当完全没有请求的时候Pod数量变成0；当有请求过来的时候动态创建Pod并处理请求（该功能目前暂未支持） --P451/483

有一个实验性质的功能：InitialResource，k8s通过Admission Control plugin记录某一个Pod（通过镜像名字和tag）的资源使用情况，在未来新创建Pod的时候，适当调整相关的资源的requests数值。 --P451/483

Cluster Autoscaler：在Node资源不足时，创建新的云主机来满足Pod的调度需求。Kubelet会创建一个Node Resource，这样集群就可以将新的Pod调度到新增加的Node上了。 --453/485

PodDisruptionBudget \(PDB\)：确保在Cluster scale down的过程中，某些Pod至少存在相应的个数（避免业务收到太大的影响，以及有最小Pod数量要求的应用受到影响） --455/487

#### 第十六章：高级调度 --457/489

node's taint + Pod's toleration

Node selector以及Node affinity：是Pod主动写清楚自己想要什么和自己不想要什么；taint是Node主动对外宣布，我不能接受哪些Pod，大部分Pod本身不需要做任何修改。

taint的Effect主要有三种：

* NoSchedule：不允许一般的Pod被分配到这个Node
* PreferNoSchedule：比上面那种限制少一点，如果没有其它合适的节点了，也可以分配
* NoExecute：不止影响分配，还会把Node上已有的Pod给逐出

Node Selector是要被淘汰的了，以后都是用Node affinity来实现相关的功能了。

多个Node affinity可以设置不同的weight，表示某个affinity的相对重要性 --467/499

自己定义的Node affinity只是一方面，k8s的Scheduler也有其他的优先级函数（比如SelectorSpreadPriority，用来防止同一个RS下的Pod都调度到一个Node上）

---

Pod Affinity：解决Pod和Pod之间尽量靠近部署的问题

通过Topology key来决定Pod Affinity的行为（hostname、rack、region等，可自行定义，只要有这个label即可，比如rack/region/zone/hostname等；比如rack=rack1;rack=rack2，然后设置toplogyKey为rack） --P469/501

Pod Affinity一旦定义了，不但自己会靠近目标Pod；如果目标Pod被删除后重新调度，目标Pod也会向自己靠近（InterPodAffinity会得分更高） --P471/503

有Pod Affinity，也有podAntiAffinity，让自己远离目标Pod来部署。

#### 第十七章：开发APP的最佳实践 --P477/509

app需要准备好随时被kill以及被调度到其他的节点上

即使Pod里面的container持续不断地crash，kubelet也会认为这个Pod是Running状态的。 --P483/515

通过Init Container，可以间接让container有不同的启动顺序。 -485/517

readyness probe可以考虑到依赖其他的服务，以避免加入Endpoint中，以及避免Deployment迭代到一个有问题的版本。 --P485/517

init Container是针对整个Pod的，而post-start/pre-stop是针对单个container的

不建议通过pre-stop来发送SIGTERM给APP进程，而是不要通过shell的方式启动APP，直接启动APP就好了。 --P489/521

防止client的连接出现connection refused之类的最佳实践：

启动Pod时，有readiness probe

关闭Pod是，等待几秒一般是一个优化用户体验的方法（比如：通过preStop来sleep 5） --P497/529

生产环境一定不要使用latest作为镜像的tag来使用

给资源打多个维度的label --P498/530

通过annotation添加相关属性

程序奔溃或者退出时，把相关信息写入/dev/termination-log，方便ops通过describe pod来查看信息

FluentD和LogStash都是基于单行的，可以让APPS输出JSON格式的日志，这样更好解析和处理 --P502/534

ksonnet：一个帮助生成k8s的yaml/json的配置文件，方便管理的工具

关于CI/CD的课题 ，可以参考一下Fbric8.io项目以及Goole的案例：

[https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes)

--506/538

#### 第十八章：扩展Kubernetes --P508/540

CustomResourceDefinition object \(CRD\)：可以定义更高级的Resource，方便使用，比如定义一个Queue的Resource，就不需要考虑Secrets/Deployment/Services之类的定义了，相当于现有Resource的整合。

除了定义CRD之外，还可以定义Controller，可以监听并创建Service、其他Deployment等

Service Catalog：类似为某一类对外提供的服务而定义的自助创建的模板，方便把k8s集群的能力对外输出。

OpenShift，Deis Workflow：基于kubernetes的工具

---

核心篇章完整阅读完毕：2019年8月18日

