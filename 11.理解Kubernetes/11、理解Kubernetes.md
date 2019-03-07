Kubernetes架构
  * Kubernetes Control Plane
      * etcd
      * API server
      *  Scheduler
      * Controller Manager
  * worker nodes
      * Kubelet
      * kube-proxy
      * Container Runtime (Docker, rkt等)
  * Add-On 组件
      * Kubernetes DNS server
      * Dashboard
      * Ingress controller

各组件之间的通信只通过API server，并不直接相互通信。也只有API server在操作etcd，其他组件通过API server来修改集群状态，如图：  
  ![inter-dependencies.png](https://images.gitee.com/uploads/images/2019/0307/194523_23a1de86_5849.png "inter-dependencies.png")

在worker nodes上的组件需要在node节点上同时运行，Kubernetes Control Plane可以在不同服务器上运行多个实例来确保高可用。etcd和API server可以同时运行多个实例，并行的执行它们要处理的任务。Scheduler和Controller Manager同一时间只能有一个实例执行它们要处理的任务，其他实例备用

使用etcd
  * Pods，ReplicationControllers，Services，Secrets的元数据等等都存在etcd中，etcd是一个快速，分布式，一致性的key-value存储
  * API server是唯一一个直接与etcd通讯的组件，其他组件都是通过API server来操作etcd。这样做的好处是：
      * 更好的并发控制，使用乐观锁
      * 校验机制
      * 向其他组件隐藏存储细节
  * 乐观锁，采用版本号的方式。当数据被更新时，客户端会传一个版本号，判断这个版本号和现在存储的版本号是否一致，如果不一致，则会被阻止更新，这样客户端必须重新读取数据，再次尝试更新。在Kubernetes里的资源都包含metadata.resourceVersion字段，客户端在更新资源时需要把这个字段的值传给API server，如果版本号和etcd里存储的不一致，会被阻止更新
 
API server
  * API server通过restful api提供CRUD接口用来查询和修改集群状态，然后将数据存储到etcd中
  * API server客户端，如kubectl在post 数据到API server后，会经过三个插件校验
      * authenticate plugins，API server通过多个authenticate plugins来依次验证客户端发送的请求，直到有一个插件验证通过。通过HTTP header里的Authorization字段来获取到客户端登录的用户ID，用户名称和用户所属的组名称，供后面的authorization plugins使用
      * authorization plugins，验证用户是否有权限执行特定操作，如登录用户在创建pod时，API server通过authorization plugins来依次验证，决定用户是否可以在请求的namespace里创建pod，如果有一个authorization plugin验证通过，则进入Admission Control plugins
      * Admission Control plugins，在创建，修改，删除资源时，请求会传到Admission Control plugins做验证，API server会设置很多个Admission Control plugins，这些插件可以修改请求的资源定义，如初始化未传入的字段，或者重写值，Admission Control plugins包含如下：
          * AlwaysPullImages，重写pod的imagePullPolicy为Always，强制在pod部署时每次都拉取镜像
          * ServiceAccount，在pod没有指定ServiceAccount时，使用默认的ServiceAccount
          * NamespaceLifecycle，阻止pod在不存在的namespaces，已删除的namespaces中创建
          * ResourceQuota，确保在某个namespaces中的pod最多只能使用在namespaces中指定的cpu和内存
  * 三个插件都校验通过后，API server将资源的定义存储到etcd中

Scheduler
  * Scheduler通过API server的通知机制，找到新创建的pod，为没有分配node节点的pod指定node。Scheduler所做的仅仅是通过API server更新pod的yaml，将node节点更新进去。在被分配node节点上的Kubelet通过API server的通知机制得到pod被调度到自己的node节点上，Kubelet开始创建并运行pod容器
  * Scheduler仅仅做pod分配node，默认的分配机制如下：
      * 找到适合的node列表
          * node现有资源是否满足pod请求的资源，如cpu和内存
          * pod定义里是否指定了node名称
          * node里的label是否包含pod里定义的lable
          * 如果pod里指定host port，对应的node相应端口是否被占用
          * 如果pod里指定特定的volume，对应的node是否可以提供这样的volume
          * pod是否容忍node的taints 
      * 在node列表中选择最合适的
          * 属于同一个Service或ReplicaSet的pod默认会被分配到不同的node上，如果一个node挂了，不影响应用。但是并不能保证都是这样，可以通过设置pod affinity 和 anti-affinity规则来保证
  * 可以自定义Scheduler，如果要使用自定义的Scheduler，可以pod的定义里指定schedulerName 

Controller Manager  
Controller监听API server资源的变化，为资源的变化执行操作。Controller会检查实际状态和期望状态是否一致，若不一致，则会执行循环，直到状态一致为止。Controller之间不会直接相互通信，每一个Controller都是通过API server的通知机制来操作资源
  * Replication Manager，管理ReplicationController，如果实际pod数量小于期望数量，ReplicationController每次循环会增加一个实例，这并不是ReplicationController实际操作的，而是它创建一个新的pod定义，把pod定义post到API server，让Scheduler调度node，最终由Kubelet运行容器。
  * ReplicaSet， DaemonSet，Job controllers，类似Replication Manager，它们从它各自的pod template中创建pod资源，把pod定义post到API server，让Scheduler调度node，最终由Kubelet运行容器
  * Deployment controller，当Deployment对象被修改后（引起pod被重新部署），Deployment controller通过创建ReplicaSet来操作pod
  * StatefulSet controller，类似ReplicaSet Controller，除了操作pod定义之外，StatefulSet controller还要为每一个pod定义PersistentVolumeClaims
  * Node controller，管理集群的node资源，监控每个node的健康状态，将pod从不健康的node上删除
  * Service controller，管理Service 
  * Endpoints controller，Service并不是直接链接pod，而是通过endpoints(IPs and ports)，Endpoints controller更新endpoints。它同时监控Services和pod的通知，当Services被添加或更新，它会把符合Services选择器的pod ip和端口添加到endpoints中
  * Namespace controller，当namespace被删除时，Namespace controller通过API server删除属于这个namespace的资源
  * PersistentVolume controller，当创建一个pvc时，PersistentVolume controller会找到一个合适的PersistentVolume进行绑定

Kubelet  
Kubelet在worker node上基本负责所有事情
  * 它监听API server，当pod被分配node节点后，通知容器运行时（docker）运行容器
  * 它将容器的运行状态，事件，资源消耗告知API server
  * 它执行容器的liveness探针，如果探针返回失败，重启容器
  * 当pod在API server上被删除时，终止容器运行，并告知API server，pod已被终止

Kubernetes Service Proxy
  * 和Kubelet一样，每个work node上还运行kube-proxy，它来确保客户端可以链接你通过API server定义的service
  * kube-proxy初始实现，使用一个服务进程接受请求，并将请求转给pods。为了拦截连接到service ip的请求，proxy通过配置iptables规则将请求转发给服务进程，由服务进程再转发给pods
  * 为了提高性能，可以直接通过配置iptables规则将请求转发给随机一个pod

Kubernetes add-ons
  * add-ons可以通过提交yaml文件到API server进行部署。在minikube里，Ingress controller，dashboard和DNS使用 Deployment进行部署，可以通过`kubectl get deploy -n kube-system`查看
  * DNS Server，在集群内的所有pod都被配置使用集群内部的DNS Server，DNS Server pod通过kube-dns service暴露出来，在集群内运行的容器里文件/etc/resolv.conf的nameserver都被指定为kube-dns service的IP地址。如图：  
  ![dns-server.png](https://images.gitee.com/uploads/images/2019/0307/194512_ce082f21_5849.png "dns-server.png")
  kube-dns pod使用API server的通知机制发现services和endpoints的变化，然后更新DNS records，这样客户端可以得到近乎实时的DNS数据。在services和endpoints的变化到kube-dns pod监听到变化会有一段很小的时间，DNS records会不可用
  * Ingress controller，运行一个反向代理服务（类似与nginx），通过集群里的services，endpoints资源来配置Ingress。Ingress controller通过使用API server的通知机制发现services和endpoints的变化来改变代理服务的配置。尽管Ingress资源的定义使用了services，Ingress controller是直接转发流量到后端pod，而不是通过services的IP。

各组件之间的相互协作  
当通过kubectl提交一个Deployment yaml文件到API server后，所涉及组件之间的交互如下：   
![yamlsubmitevent.png](https://images.gitee.com/uploads/images/2019/0307/194550_ed8ed94f_5849.png "yamlsubmitevent.png")
  1. Deployment controller创建ReplicaSet资源发送给API server。所有的API server客户端通过API server的通知机制监听新创建的Deployment资源，其中Deployment controller处理Deployment资源
  2. ReplicaSet controllers创建Pod资源发送给API server
  3. Scheduler分配node到新创建的pod定义上
  4. Kubelet监听API server的pod改变，当pod被调度到node节点上，它检查pod定义的有效性，然后告诉docker，由docker启动容器

理解运行中的pod  
  * `kubectl run nginx --image=nginx:alpine`，运行一个deployment
  * `minikube ssh`登录node节点，`docker ps`查看刚创建的容器
  * 可以看到启动了一个Nginx容器，和一个附加容器pause-amd64，每一个pod都会有pause容器，用来占用网络和Linux namespaces，pod里的其它容器共用pause容器占用的网络和Linux namespaces
  * 应用容器可能挂掉或重启，重启后的应用容器仍然共用pause容器占用的网络和Linux namespaces。pause容器的生命周期和pod相同，如果pause容器被意外删除，Kubelet会重启pause容器和pod里的所有容器

services内部网络实现
  * services由每个node节点的kube-proxy处理
  * 当services在API server中被创建，虚拟IP地址就会被分配，API server 通知kube-proxy，每一个node节点的kube-proxy通过配置iptables规则，来确保发给service ip和端口的流量被截获，然后修改目的地址为后端的随机pod地址
  * kube-proxy不仅需要监控service资源的变更，还要监控endpoint资源的变更
  * services图例：  
    ![proxy.png](https://images.gitee.com/uploads/images/2019/0307/194532_6a160a89_5849.png "proxy.png")
  * 实例
      * `kubectl describe svc netcorek8s-nodeport`，获得服务的IP
      * `minikube ssh`，登录到node节点，`sudo iptables-save`，保存iptables查看转发规则