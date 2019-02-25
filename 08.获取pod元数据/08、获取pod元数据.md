应用程序经常需要获取它们运行的环境，包括它们自己的详细信息和集群内其他组件的信息。在 Kubernetes里可以通过两种方式：
  * Downward API
  *  Kubernetes API server

Downward API
  * ConfigMap，Secret可以通过环境变量来向应用程序传递配置数据，这种配置数据在pod被调度node之前就已经明确了。但是pod ip，运行应用程序的node名称，pod名称(被ReplicaSet自动生成的pod名称)，pod的lable等等信息都是在pod被调度完成后，才能知道。这时可以通过Downward API来获取
  * Downward API允许你通过**环境变量**或**downwardAPI volume的文件**来传递元数据，它并不是你要请求一个REST API接口来获取数据。目前支持如下信息传递到容器里：
      * pod名称
      * pod IP
      * pod属于的namespace
      * 运行pod的node名称
      * 运行pod的serviceaccount 名称
      * 每一个容器的CPU and memory requests 
      * 每一个容器的CPU and memory  limits 
      * pod的lables
      * pod的annotations  
    其中pod的lables，annotations只能通过downwardAPI volume来传递，而不能通过环境变量，这是因为lables，annotations可能在pod运行中被更改，环境变量在容器启动后就不能被更改了
  * 通过环境变量获取pod和容器的元数据
      * `kubectl create -f downward-api-env.yml`，创建一个名为downward-api-env的pod，设置环境变量
      * `kubectl exec downward-api-env env`，查看容器的环境变量
  * 通过downwardAPI volume获取pod和容器的元数据
      * `kubectl create -f  downward-api-volume.yml`，创建一个名为downward-api-volume的pod，挂载downward volume
      * `kubectl exec downward-api-volume -- ls -lL /etc/downward`，查看挂载的文件
      * `kubectl exec downward-api-volume cat /etc/downward/labels`，查看labels文件内容

链接Kubernetes API server  
  * Downward API只是提供了pod和pod内部容器的元数据，如果需要其它pod或者集群内其它资源的信息，这时可以通过链接Kubernetes API server来获取信息，也可以通过Kubernetes API server更新一些信息
  * `kubectl cluster-info`，获取集群api server ip，`curl -k https://192.168.99.102:8443`，访问地址，报错"message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  * 通过`kubectl proxy`代理来处理验证信息，访问 API server
      * `kubectl proxy`，运行代理，浏览器运行http://127.0.0.1:8001/api/v1/namespaces/default/pods/downward-api-volume，如图：  
        ![kubectlproxy.png](https://images.gitee.com/uploads/images/2019/0225/214105_d7243a45_5849.png "kubectlproxy.png")
      * `kubectl get pod downward-api-volume -o json`，发现两个结果，返回的pod json定义相同
  * 在pod内链接API server  
      * 在本地机器可以通过kubectl proxy来链接API server，在pod内链接API server需要处理三个事情：
          * 找到API server的地址
          * 验证API server证书，确保你在链接对的API server，而不是其它中间代理
          * 登录API server
      * 实例
          * `kubectl create -f curl.yml`，运行一个包含curl命令的pod
          * `kubectl exec -it curl bash`，进入容器内部
          * `env | grep KUBERNETES_SERVICE`，获取KUBERNETES_SERVICE的IP和端口，发现kubernetes服务绑定的是https的默认端口443，如图：  
            ![curl-env.png](https://images.gitee.com/uploads/images/2019/0225/214052_de8f5cdf_5849.png "curl-env.png")
          * `curl https://kubernetes`，错误提示显示如果需要关闭验证证书，可以使用`-k`。尽管最简单的方式是使用`-k`跳过证书验证，但是生产环境不建议这样做，因为这样可能会暴露你的authentication token到使用中间人攻击的攻击者
          * `ls /var/run/secrets/kubernetes.io/serviceaccount/`，查找默认的Secret，包含证书文件，namespace和token
          * `curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes`，错误消息提示需要进一步验证
          * `export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`，为curl设置CURL_CA_BUNDLE环境变量，再次执行`curl https://kubernetes`
          * `TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)`，设置TOKEN变量
          * `curl -H "Authorization: Bearer $TOKEN" https://kubernetes`，将token传给kubernetes服务，发现还是报错，这是因为Kubernetes集群默认启用了RBAC，pod使用的service account不被允许访问API server，为了测试，需使用`kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --group=system:serviceaccounts`命令暂时为所有的serviceaccount赋cluster-admin的权限，再次访问就能得到正确结果，如图：
            ![pod-api-server.png](https://images.gitee.com/uploads/images/2019/0225/214118_021a2dbe_5849.png "pod-api-server.png")
        * `NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)`，设置NS变量
        * `curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/$NS/pods/downward-api-volume`，得到pod的定义
      * 整体流程：使用ca.crt文件，验证API server的证书是不是正确的，使用token文件里的内容放在Authorization header里，使用namespace文件里的内容获取pod所在的namespace
  * 使用代理容器链接API server  
      * 在pod里处理证书验证，登录验证，获取命名空间比较复杂。可以创建一个代理容器，运行kubectl proxy，由kubectl proxy来自动处理和API server之间的验证，因为同一pod内的多个容器都使用相同的网络，所以可以使用localhost来访问API server，流程：localhost:8001(http)-->kubectl proxy(https)-->API server
      * 实例
          * 添加dockerfile文件，代码如下：
            ```
            FROM alpine
            ADD kubectl /kubectl
            ADD kubectl-proxy.sh /kubectl-proxy.sh
            RUN chmod +x /kubectl-proxy.sh && chmod +x /kubectl
            ENTRYPOINT /kubectl-proxy.sh
            ```
          * 添加kubectl-proxy.sh文件，代码如下：
            ```
            #!/bin/sh
            API_SERVER="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
            CA_CRT="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
            TOKEN="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
            /kubectl proxy --server="$API_SERVER" --certificate-authority="$CA_CRT" --token="$TOKEN" --accept-paths='^.*'
            ```
        * 准备好kubectl文件，需要翻墙下载，可使用我下载好的文件，[下载地址](https://dl.k8s.io/v1.10.0/kubernetes-client-linux-amd64.tar.gz)
        * 构建镜像refactor2/kubectl-proxy:1.10.0，上传至dockerhub
        * `kubectl create -f curl-with-proxy.yml`，创建一个pod，包含两个容器
        * `kubectl exec -it curl-with-proxy -c main bash`，进入容器内部，执行`curl localhost:8001`，如图： 
            ![pod-proxy.png](https://images.gitee.com/uploads/images/2019/0225/214129_0ea5b49d_5849.png "pod-proxy.png")




