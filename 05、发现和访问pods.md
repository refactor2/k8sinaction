Kubernetes通过暴露Services来访问Services后面的pods，可以通过kubectl expose来暴露一个Services，也可以通过yml创建   
  1. 创建一个名为netcorek8ssvc的Services，对外监听13300端口，向容器43300端口转发，选择标签apprs: netcorek8s的pod，`kubectl create -f 05、netcorek8s-svc.yml`  
  2. `kubectl get svc`，得到刚创建的netcorek8ssvc的服务IP为`10.106.70.60`，如图：
      * ![05、getsvc.png](https://images.gitee.com/uploads/images/2019/0217/003353_430aa493_5849.png "05、getsvc.png")  
      * 这个地址只能在集群内部外访问
  3. 测试服务地址，通过`kubectl get pod -l apprs=netcorek8s`获取pod列表，如图：
      * ![05、getpod.png](https://images.gitee.com/uploads/images/2019/0217/003341_b969dcdb_5849.png "05、getpod.png")
      * 随便选一个pod名称，`kubectl exec netcorek8srs-4bhv6 -- curl -s http://10.106.70.60:13300/api/values`，使用'--'号是告诉kubectl后面的命令在pod中执行
  4. 配置sessionAffinity为ClientIP，保证同一个客户端访问到固定的pod上
      * 如果访问多次，会发现每次返回的服务器名称会不同，因为service默认会随机转发到后端的pod上
      * sessionAffinity只有两个值`None`，`ClientIP`，没有基于cookie的会话保持。这是因为Kubernetes的service不是操作HTTP7层协议的
      * service是处理TCP,UDP包，不关心包携带的payload数据  
      * 测试
        * `kubectl delete svc netcorek8ssvc`，删除服务
        * `kubectl create -f 05、netcorek8s-svc-sessionAffinity.yml`，重建服务，获取服务集群IP，测试
        
服务发现，创建好服务后，访问该服务的pod该如何知道这个服务暴露的IP和端口呢？有如下两种方式：  
  1. 通过环境变量
      * 当一个pod启动时，Kubernetes会将当时已经存在的service初始化一系列的环境变量到pod里
         * `kubectl delete pod -l apprs=netcorek8s`，先删除在netcorek8ssvc服务创建之前的pod，由ReplicaSet重建pod
         * `kubectl exec netcorek8srs-6grmm env`，在pod里执行`env`命令，如图：  
         ![05、podenv.png](https://images.gitee.com/uploads/images/2019/0217/003407_e9b2363b_5849.png "05、podenv.png")
         * 会发现存在：NETCOREK8SSVC_SERVICE_HOST和NETCOREK8SSVC_SERVICE_PORT

  2. 通过DNS
      * 在kube-system的namespace下有一个pod叫kube-dns，这个pod运行DNS server，其它运行在这个集群里的pod会自动运用这个dns（Kubernetes通过修改每个容器的/etc/resolv.conf文件实现）
      * 访问服务的pod可以通过域名`{服务名}.{namespace}.svc.cluster.local`来访问服务，如`netcorek8ssvc.default.svc.cluster.local`。其中`.svc.cluster.local`是可配置的
      * 访问dns域名可以简化，如直接省略`.svc.cluster.local`。如果client和service在同一namespace下，可以省略namespace。进入pod，执行curl，如下三种方式是等价的：
        * curl http://netcorek8ssvc.default.svc.cluster.local:13300/api/values
        * curl http://netcorek8ssvc.default:13300/api/values
        * curl http://netcorek8ssvc:13300/api/values
      * `kubectl exec netcorek8srs-4djwf -it bash`，进入pod里
        * 执行`cat /etc/resolv.conf`，如图：  
        ![05、dns.png](https://images.gitee.com/uploads/images/2019/0217/003318_25c7279d_5849.png "05、dns.png")
        * 执行`ping`，会发现ping不通，这是因为service的ClusterIP是一个虚IP，只有和service的端口一起使用才能访问
      * client通过dns获取到服务IP后，如果服务不是用的默认端口（如http的80端口），则仍需要通过环境变量获取到端口

在集群内部访问外部服务
  1. 当访问service时，service会随机选中一个后面的pod提供服务，这是通过service endpoints实现的
      * `kubectl describe svc netcorek8ssvc`，如图：  
      ![05、endpoints.png](https://images.gitee.com/uploads/images/2019/0217/175852_610b2929_5849.png "05、endpoints.png")
      * `kubectl get endpoints netcorek8ssvc`
  2. 当需要访问的服务不在集群内部时，可以手动创建service和endpoints
      * endpoints  
        ```
        apiVersion: v1
        kind: Endpoints
        metadata:
          name: external-service
        subsets:
          - addresses:
            - ip: 11.11.11.11
            - ip: 22.22.22.22
            ports:
            - port: 80 
        ```
      * service  
        ```
        apiVersion: v1
        kind: Service
        metadata:
          name: external-service
        spec:
          ports:
          - port: 80
        ```
  3. 可以为外部服务的提供一个别名，需要指定service的type为ExternalName
      * service
        ```
        apiVersion: v1
        kind: Service
        metadata:
          name: external-service
        spec:
          type: ExternalName
          externalName: api.baidu.com
          ports:
          - port: 80
        ```

暴露服务到集群外部，让外部客户端可以访问  
  1. 使用NodePort service，指定service的type为NodePort
      * `kubectl create -f 05、netcorek8s-svc-nodeport.yml`，创建了一个集群内部可访问的端口13300，要转发pod的端口为43300，node上暴露的端口为32000，node上暴露的端口取值范围为：30000-32767，如图：  
      ![05、nodeport.png](https://images.gitee.com/uploads/images/2019/0217/175905_4f15c425_5849.png "05、nodeport.png")
      * 使用nodeport的坏处，当访问的node挂掉后，则不能访问对应的service了
      * 使用nodeport的转发机制：当集群外client访问node的ip和端口时，nodeport会将请求转发到对应的service上，然后由service在转发到对应的pod上
      * 创建完之后，可以通过如下方式访问
        * 在集群内部，curl -s http://10.96.71.111:13300/api/values
        * 在集群外部，curl -s http://192.168.99.130:32000/api/values， 其中 192.168.99.130为minikube ip
  2. 使用外部的load balancer，指定service的type为LoadBalancer
      * load balancer是基于NodePort做的，在NodePort上做了一个负载均衡，这个load balancer一般由像相应的云厂商提供，minikube不支持，但是仍然可以通过nodeport的方式访问
      * `kubectl create -f 05、netcorek8s-svc-loadbalancer.yml`，如通过curl -s http://192.168.99.130:32001/api/values  访问

使用Ingress暴露服务到集群外部，让外部客户端可以访问 
  1. 使用Ingress，可以将多个服务一起暴露出来，类似与nginx做代理。Ingress操作在HTTP7层协议上，可以提供基于cookie的会话保持，而service不可以
  2. minikube使用ingress是通过add-on的方式，查看目前已启用的add-on，`minikube addons list`，可以通过`minikube addons enable ingress`启用ingress
  3. `kubectl create -f 05、netcorek8s-ingress.yml`，将域名netcorek8s.refactor.com的所有流量转发到netcorek8ssvc-nodeport service上
      * `kubectl get ingress`获取到IP地址为10.0.2.15，端口为80
      * `minikube ssh`，登录到VM里，`sudo vi /etc/hosts`，编辑/etc/hosts文件，添加host：`10.0.2.15  netcorek8s.refactor.com`
      * curl -s http://netcorek8s.refactor.com/api/values，  如图：  
      ![05、ingresstest.png](https://images.gitee.com/uploads/images/2019/0217/175937_c4386efe_5849.png "05、ingresstest.png")
  4. client访问http://netcorek8s.refactor.com/api/values时，先去找netcorek8s.refactor.com对应的IP，然后向这个IP发送消息（携带header，包括Host:etcorek8s.refactor.com），被Ingress Controller接收，然后由Ingress Controller随机选择一个pod进行服务。注意Ingress并不会将请求转发给service，Ingress使用service主要是为了获取service endpoint，然后获取到pod列表
  5. 使用Ingress处理TLS请求
      * 当client打开一个TLS请求到Ingress Controller时，Ingress Controller处理这个加密链接。client和Ingress Controller的通信是加密的，但是Ingress Controller和pod之间的通信不是加密的
      * 生成私钥和证书
        * `openssl genrsa -out ingresstls.key 2048`
        * `openssl req -new -x509 -key ingresstls.key -out ingresstls.cert -days 360 -subj /CN=netcorek8s.refactor.com`
      * 在集群里创建secret，`kubectl create secret tls tls-secret --cert=ingresstls.cert --key=ingresstls.key`
      * `kubectl apply -f 05、netcorek8s-ingress-tls.yml`
      * curl -k -v https://netcorek8s.refactor.com/api/values， 如图：  
      ![05、ingresstlstest.png](https://images.gitee.com/uploads/images/2019/0217/175953_0268f9eb_5849.png "05、ingresstlstest.png")

readiness probes
  1. service后端的pod可能需要时间初始化，才能接受外部请求。这时需要判断pod是否准备好接受外部请求了，类似与liveness probes，Kubernetes提供了readiness probes。readiness 探针也包括三种：http get，tcp socket，exec
  2. 当pod里的container启动后，Kubernetes可以配置等一定的时间后，周期性的向readiness探针发送请求。如果readiness探针连续几次（可配置）返回失败，则将这个pod从service  endpoints里移除，如果返回成功，则再将pod加入到service  endpoints
  3. liveness探针，通过杀掉不健康的container，重启一个新的pod来保证可用。而readiness探针是保证只有准备好的pod才会提供服务，它在container启动时和已经运行很长时间时都很有用
  4. 实例：  
      * `kubectl apply -f 05、netcorek8s-rs-readiness.yml`，为名为netcorek8srs的ReplicaSet添加readiness，`kubectl delete pod -l apprs=netcorek8s`，删除以前pod，让ReplicaSet重建pod
      * `kubectl describe svc netcorek8ssvc-nodeport`，`kubectl describe rs netcorek8srs`，你会发现执行一段时间后，所有的pod都会变成不可用的状态
  5. 在生成环境，一定要使用readiness探针

headless service，设置clusterIP为None，客户端可以找到service后面所有的pod ip。当使用DNS查找headless service时，会返回service后面所有的pod ip
  1. 实例：
      * `kubectl create -f 05、netcorek8s-svc-headless.yml`，创建一个名为netcorek8ssvc-headless的service，`kubectl get svc netcorek8ssvc-headless`，`kubectl describe svc netcorek8ssvc-headless`，刚开始创建成功后，会有endpoints，过一会endpoints会为空，这是因为readiness探针起作用了
      * `kubectl apply -f 04、netcorek8s-rs.yml`，更新netcorek8srs定义，`kubectl delete pod -l apprs=netcorek8s`，删除以前pod，让ReplicaSet重建pod，还原readiness的影响，这是会发现endpoints会一直有值
  2. 通过DNS获取到所有的pod ip
      * 由于netcorek8s的容器镜像里没有nslookup命令，需要在集群里创建一个临时pod，可以在创建的pod里执行nslookup命令，这里选择的镜像是tutum/dnsutils
      * `kubectl run dnsutils --image=tutum/dnsutils --generator=run-pod/v1 --command -- sleep infinity`，创建一个名为dnsutils的临时pod，执行一个一直休眠的命令，保持pod的运行。`--generator=run-pod/v1`是指直接创建pod，而不是通过ReplicationController来创建
      * `kubectl exec dnsutils -- nslookup netcorek8ssvc-headless`
  3. 客户端可以通过DNS name，像访问正常service一样，访问headless service，但是headless service里，DNS直接返回的所有pod ip，客户端是直接连接到pod，而不是通过service代理
  4. headless service依然为后面的pods提供负载均衡功能，这是通过DNS round-robin机制实现的，而不是通过service代理

关于service的排错指南
  1. 确认是集群内部访问的service
  2. 不要ping service来检查service是否通，因为service的集群内IP是虚IP，一直ping不通
  3. 如果使用了readiness探针，须确保探针返回成功，否则pod不会放在service endpoint
  4. 确认pod是否在service里，可以通过`kubectl get endpoints`检查
  5. 如果通过DNS不能访问，可以看下直接通过cluster IP访问试下
  6. 检查连接service的端口是port，而不是targetPort
  7. 直接连接pod的ip和端口，保证pod可以访问
  8. 如果通过pod的ip和端口不能访问，检查应用是不是只绑定了localhost