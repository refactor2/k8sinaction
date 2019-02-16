1、Services
--
Kubernetes通过暴露Services来访问Services后面的pods，可以通过kubectl expose来暴露一个Services，也可以通过yml创建   
  1. 创建一个名为netcorek8ssvc的Services，对外监听13300端口，向容器43300端口转发，选择标签apprs: netcorek8s的pod，`kubectl create -f 05、netcorek8s-svc.yml`  
  2. `kubectl get svc`，得到刚创建的netcorek8ssvc的服务IP为`10.106.70.60`，如图：
      * ![kubectl-get-rc](https://images.gitee.com/uploads/images/2019/0202/173537_99c11aed_5849.png "04、getrc.png")  
      * 这个地址只能在集群内部外访问
  3. 测试服务地址，通过`kubectl get pod -l apprs=netcorek8s`获取pod列表，如图：
      * ![kubectl-get-rc](https://images.gitee.com/uploads/images/2019/0202/173537_99c11aed_5849.png "04、getrc.png")  
      * 随便选一个pod名称，`kubectl exec netcorek8srs-4bhv6 -- curl -s http://10.106.70.60:13300/api/values`，使用'--'号是告诉kubectl后面的命令在pod中执行
  4. 配置sessionAffinity为ClientIP，保证同一个客户端访问到固定的pod上
      * 如果访问多次，会发现每次返回的服务器名称会不同，因为service默认会随机转发到后端的pod上
      * sessionAffinity只有两个值`None`，`ClientIP`，没有基于cookie的会话保持。这是因为Kubernetes的service不是操作HTTP7层协议的
      * service是处理TCP,UDP包，不关心包携带的payload数据  
服务发现，创建好服务后，访问该服务的pod该如何知道这个服务暴露的IP和端口呢？有如下两种方式：  
  1. 通过环境变量
      * 当一个pod启动时，Kubernetes会将当时已经存在的service初始化一系列的环境变量到pod里
         * `kubectl delete pod -l apprs=netcorek8s`，先删除在netcorek8ssvc服务创建之前的pod，由ReplicaSet重建pod
         * `kubectl exec env`，在pod里执行`env`命令，如图：![kubectl-get-rc](https://images.gitee.com/uploads/images/2019/0202/173537_99c11aed_5849.png "04、getrc.png") 
         * 会发现存在

  2. 通过DNS
      * 在kube-system的namespace下有一个pod叫kube-dns，这个pod运行DNS server，其它运行在这个集群里的pod会自动运用这个dns（Kubernetes通过修改每个容器的/etc/resolv.conf文件实现）
      * 访问服务的pod可以通过域名`{服务名}.{namespace}.svc.cluster.local`来访问服务，如`netcorek8ssvc.default.svc.cluster.local`。其中`.svc.cluster.local`是可配置的
      * 访问dns域名可以简化，如直接省略`.svc.cluster.local`。如果client和service在同一namespace下，可以省略namespace。进入pod，执行curl，如下三种方式是等价的：
        * curl http://netcorek8ssvc.default.svc.cluster.local:13300/api/values
        * curl http://netcorek8ssvc.default:13300/api/values
        * curl http://netcorek8ssvc:13300/api/values
      * `kubectl exec -it bash`，进入pod里
        * 执行`cat /etc/resolv.conf`，如图：![kubectl-get-rc](https://images.gitee.com/uploads/images/2019/0202/173537_99c11aed_5849.png "04、getrc.png") 
        * 执行`ping`，会发现ping不通，这是因为service的ClusterIP是一个虚IP，只有和service的端口一起使用才能访问
      * client通过dns获取到服务IP后，如果服务不是用的默认端口（如http的80端口），则仍需要通过环境变量获取到端口