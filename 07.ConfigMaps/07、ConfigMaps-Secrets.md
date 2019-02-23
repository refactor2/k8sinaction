应用程序一般都需要配置，容器应用使用配置的两种方式：
  * 一般通过配置应用程序的命令行参数，如果参数有很多，一般会将配置参数放到一个配置文件里
  * 使用环境变量

在docker的容器里使用配置文件，你必须在制作镜像的时候，把配置文件写到镜像里，或者把挂载一个volume。把配置文件写到镜像里，如果想要修改配置，你必须重新构建镜像，另外，可以访问这个镜像的人都可以得到这个配置，如果里面有敏感信息，会不安全。使用volume会好一些，但是你必须在容器启动之前，把文件放到volume中。如果你还记得使用gitRepo volume当做volume可能会更好一些，因为你可以做版本控制  

Kubernetes提供了ConfigMap来存储配置数据，不管你是否使用ConfigMap来存储配置数据，你可以通过如下三种方式来配置你的应用程序：
  * 通过命令行参数传递到容器
  * 通过环境变量传递到容器
  * 通过特定的volume挂载一个配置文件到容器

通过命令行参数传递到容器
  * 在创建容器的时候，你一般是指定默认的执行的命令，但是Kubernetes允许你使用新的命令来覆盖默认的命令。在Dockerfile里有两个相关的指令：
      * ENTRYPOINT，定义容器启动时默认执行的命令
      * CMD，定义要传入的参数给ENTRYPOINT
  * 理解ENTRYPOINT的两种方式：dotnet
      * ENTRYPOINT ["dotnet", "netcorek8s.dll"]，直接使用dotnet运行，推荐使用这种方式
      * ENTRYPOINT dotnet netcorek8s.dll，使用shell来运行dotnet
      * 可以通过进入容器里，执行命令`ps x`来查看进程情况，如：`docker exec 4675d ps x`
  * 实例
      * 添加fortuneloop-args.sh文件
        ```
        #!/bin/bash
        trap "exit" SIGINT

        INTERVAL=$1
        echo Configured to generate new fortune every $INTERVAL seconds

        mkdir -p /var/htdocs

        while :
        do
        echo $(date) Writing fortune to /var/htdocs/index.html
        /usr/games/fortune > /var/htdocs/index.html
        sleep $INTERVAL
        done
        ```
      * 添加Dockerfile文件
        ```
        FROM ubuntu:16.04
        # Ali apt-get source.list
        RUN mv /etc/apt/sources.list /etc/apt/sources.list.bak && \
            echo "deb http://mirrors.aliyun.com/ubuntu/ xenial main" >/etc/apt/sources.list && \
            echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial main" >>/etc/apt/sources.list && \
            echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main" >>/etc/apt/sources.list && \
            echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main" >>/etc/apt/sources.list && \
            echo "deb http://mirrors.aliyun.com/ubuntu/ xenial universe" >>/etc/apt/sources.list && \
            echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe" >>/etc/apt/sources.list && \
            echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe" >>/etc/apt/sources.list && \
            echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe" >>/etc/apt/sources.list && \
            echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-security main" >>/etc/apt/sources.list && \
            echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main" >>/etc/apt/sources.list && \
            echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe" >>/etc/apt/sources.list && \
            echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe" >>/etc/apt/sources.list 

        RUN apt-get update ; apt-get -y install fortune
        ADD fortuneloop-args.sh /bin/fortuneloop.sh
        RUN chmod +x /bin/fortuneloop.sh
        # RUN mkdir /var/htdocs

        ENTRYPOINT ["/bin/fortuneloop.sh"]
        CMD ["10"]
        ```
      * 构建镜像，上传到dockerhub
      * `docker run -t refactor2/fortune:args`，使用默认参数，10s执行一次fortune命令
      * `docker run -t refactor2/fortune:args 5`，修改参数，5s执行一次fortune命令，如图：  
        ![docker-args.png](https://images.gitee.com/uploads/images/2019/0223/175958_e9ec7771_5849.png "docker-args.png")
  * 在Kubernetes里可以重写Dockerfile的ENTRYPOINT和CMD，类似于：
    ```
    kind: Pod
    spec:
        containers:
        - image: some/image
        command: ["/bin/command"]
        args: ["arg1", "arg2", "arg3"]
    ```
    docker和在Kubernetes对应关系如下：  
    docker    | Kubernetes | 描述
    ----------|------------|-------
    ENTRYPOINT| command    | 容器里执行的命令
    CMD       | args       | 传给命令的参数
  * 实例
      * `kubectl create -f fortune-pod-args.yml`，创建pod，修改参数，2s执行一次fortune命令
      * `kubectl port-forward fortune3s 8080:80`，在本地暴露一个端口，打开浏览器http://127.0.0.1:8080访问
      * `kubectl logs -c html-generator fortune3s`，查看pod fortune3s 容器 html-generator 的日志

设置环境变量
  * 和命令行参数一样，在pod被创建后，环境变量的值修改后，在pod里不会变
  * 实例
      * 添加fortuneloop-env.sh文件
        ```
        #!/bin/bash
        trap "exit" SIGINT

        echo Configured to generate new fortune every $INTERVAL seconds

        mkdir -p /var/htdocs

        while :
        do
        echo $(date) Writing fortune to /var/htdocs/index.html
        /usr/games/fortune > /var/htdocs/index.html
        sleep $INTERVAL
        done
        ```
      * 添加Dockerfile文件，构建镜像refactor2/fortune:env，上传镜像
      * `kubectl create -f fortune-pod-env.yml`，创建pod
      * `kubectl exec fortune-env -c html-generator env`，查看环境变量，如图：
        ![pod-env.png](https://images.gitee.com/uploads/images/2019/0223/180007_fefcc755_5849.png "pod-env.png")

ConfigMaps
  * ConfigMaps对象存储配置信息，应用程序无需直接读取ConfigMap，而是通过环境变量读取，ConfigMap的内容会被加载到环境变量里。
  * 可以在不同环境（qa，test，stage，product）里定义一个名称相同的ConfigMaps，但是ConfigMaps里的内容根据环境的变化而变化。pod引用ConfigMaps是通过名称，这样达到了在不同环境使用同一份pod定义
  * 创建ConfigMap的几种方式
      * `kubectl create configmap fortune-config --from-literal=sleep-interval=25`，创建名称为fortune-config的ConfigMap，来源是文本串，key是sleep-interval，value 是25
      * `kubectl create configmap myconfigmap --from-literal=foo=bar --from-literal=bar=baz --from-literal=one=two`，可以指定多个
      * `kubectl create configmap my-config --from-file=config-file.conf`，可以通过文件创建ConfigMap，在执行kubectl命令的目录里找到config-file.conf，key为文件名称
      * `kubectl create configmap my-config --from-file=customkey=config-file.conf`，可以指定key为customkey
      * `kubectl create configmap my-config --from-file=/path/to/dir`，可通过目录创建ConfigMap
      * `kubectl create configmap my-config --from-file=foo.json --from-file=bar=foobar.conf --from-file=config-opts/--from-literal=some=thing`
  * `kubectl create -f fortune-pod-env-configmap.yml`，创建一个pod，引用configmap
  * 当pod引用的configmap不存在时，pod里容器将不会启动成功，如果你创建了丢失的configmap，启动失败的容器会重新启动，不用你重建pod
  * 如果configmap的内容很多，手动指定env会比较烦，可以通过如下方式处理
    ```
    spec:
    containers:
    - image:some-image
      envFrom:
      - prefix: CONFIG_
        configMapRef:
          name: my-config-map
    ```
    如果my-config-map里有三个key，FOO，BAR，FOO-BAR，你可以通过envFrom将它们全部加载到环境变量里，并且使用前缀`CONFIG_`，前缀是可选的，如果不指定，默认取key名称。运行pod后，检查环境变量，会发现没有CONFIG_FOO-BAR，这是因为FOO-BAR不符合环境变量的命名规则，被跳过了
  * 使用configmap为命令行参数赋值，`kubectl create -f fortune-pod-args-configmap.yml`
  * configMap volume暴露每一个key做为一个文件
      * 实例
          * 新建一个文件夹fortune-config-file，里面包含两个文件my-nginx-config.conf，sleep-interval
          * my-nginx-config.conf，开启gzip压缩，内容为：
            ```
            server {
                listen              80;
                server_name         www.kubia-example.com;

                gzip on;
                gzip_types text/plain application/xml;

                location / {
                    root   /usr/share/nginx/html;
                    index  index.html index.htm;
                }

            }
            ```
          * sleep-interval文件的内容为25
          * `kubectl create configmap fortune-config-files --from-file=fortune-config-file`，从文件夹fortune-config-file创建configmap
          * `kubectl get configmap fortune-config-files -o yaml`，查看fortune-config-files
          * `kubectl create -f fortune-configmap-volume.yml`，创建pod，引用fortune-config-files
          * `kubectl port-forward fortune-configmap-volume 8080:80`，发现已经gzip压缩了，如图：  
            ![configmap-volume.png](https://images.gitee.com/uploads/images/2019/0223/175946_d3afb908_5849.png "configmap-volume.png")
          * `kubectl exec fortune-configmap-volume -c web-server ls /etc/nginx/conf.d`，进入容器里查看文件夹，发现两个文件，如图：  
            ![pod-nginx.png](https://images.gitee.com/uploads/images/2019/0223/180019_ba0bd7b0_5849.png "pod-nginx.png")
          * 文件sleep-interval并没有实际作用
  * configMap volume暴露指定的文件，如上例的只想加载my-nginx-config.conf文件，`kubectl create -f fortune-configmap-volume-with-items.yml`，创建一个pod，使用fortune-config-files里的my-nginx-config.conf文件，指定路径的gzip.conf。执行`kubectl exec fortune-configmap-volume-with-items -c web-server ls /etc/nginx/conf.d`，发现只有gzip.conf文件
  * 特别注意，如果使用configMap volume挂载文件，会隐藏容器里自身的文件，如上例的容器文件夹/etc/nginx/conf.d里只有gzip.conf文件，容器本身的/etc/nginx/conf.d文件夹里的文件被隐藏了。可以通过subPath来实现即挂载文件，又不影响原来容器里的文件，如：
    ```
    spec:
      containers:
      - image: some/image
        volumeMounts:
        - name: myvolume
          mountPath: /etc/someconfig.conf
          subPath: myconfig.conf
    ```
  * 使用环境变量和命令行参数的配置都不能在应用程序运行中被更改，使用configMap可以在不重建pod，并且不需要容器重启的情况下，更新配置。如果你更新了configMap，使用这个configMap的所有文件都会被更新，然后会告诉应用程序配置被改变了，然后重新加载配置。注意，从你更新configMap到引用configMap的文件被更新，需要比较长的时间，有可能需要1min
      * 实例
          * `kubectl edit configmap fortune-config-files`，把gzip由on改成off
          * `kubectl exec fortune-configmap-volume -c web-server cat /etc/nginx/conf.d/my-nginx-config.conf`，查看文件内容，发现gzip已经变成off了，但是浏览器访问还是有`Content-Encoding: gzip`，这是因为nginx没有重新加载配置
          * `kubectl exec fortune-configmap-volume -c web-server -- nginx -s reload`，再次访问，发现就没有`Content-Encoding: gzip`了
      * 当configMap被更新时，对应的文件内容也会被更新，Kubernetes是通过symbolic links实现的。`kubectl exec -it fortune-configmap-volume -c web-server -- ls -lA /etc/nginx/conf.d`，如图：  
        ![symbolic-links.png](https://images.gitee.com/uploads/images/2019/0223/180039_2c90906f_5849.png "symbolic-links.png")
      * 如果你是使用subPath来挂载，而不是将当configMap volume整个挂载，这时候configMap配置变更，并不会自动更新subPath挂载的文件
  * 使用configMap自动更新配置需要注意
      * 当你的应用程序不支持重新加载配置，这样会导致多个同时运行的副本会因配置的不同，可能产生一些问题。因为pod可能随时会被重建，新的pod会使用新的配置，老的pod会使用老的配置
      * 当你的应用程序支持重新加载配置，你也要注意因为多个副本间重新加载配置的时间会相差比较久，也可能会产生一些问题

Secrets
  * Secrets和configMap用法类似。有一些比较敏感的信息需要使用Secrets来存储。Kubernetes通过只分发给需要访问的Secrets的pod所在的node节点上，并且使用内存来存储Secrets来保证Secrets的安全
  * 在Kubernetes的master节点，一般指etcd。在1.7版本之前，etcd存储Secrets时是明文的，在1.7版本之后变成加密的了
  * 可以使用Secrets存储不敏感的信息，但是需要注意Secrets的存储大小被限制在1M以内
  * 容器在使用secret volume时，secret的值会被自动解密
  * `kubectl describe pod fortune-configmap-volume`，可以看到pod的Volumes里有一个默认的token Secret：default-token-mn7jc。可以看到`/var/run/secrets/kubernetes.io/serviceaccount from default-token-mn7jc (ro)`，执行`kubectl exec fortune-configmap-volume -c html-generator ls /var/run/secrets/kubernetes.io/serviceaccount/`下有3个文件 
  * `kubectl get secrets`   
  * `kubectl describe secrets`
  * 实例
      * `openssl genrsa -out https.key 2048`，创建一个私钥
      * `openssl req -new -x509 -key https.key -out https.cert -days 3650 -subj /CN=www.refactor-example.com`，根据私钥创建证书
      * 在证书和秘钥同级目录创建一个文件foo，内容为bar，仅为测试用
      * `kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert --from-file=foo`，创建fortune-https secret
      * `kubectl get secret fortune-https -o yaml`，会发现foo也被加密了
      * `kubectl edit configmap fortune-config-files`，编辑configmap，支持tls，核心代码：
        ```
        server {
            listen              80;
            listen              443 ssl;
            server_name         www.refactor-example.com;
            ssl_certificate     certs/https.cert;
            ssl_certificate_key certs/https.key;
            ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
            ssl_ciphers         HIGH:!aNULL:!MD5;

            location / {
                root   /usr/share/nginx/html;
                index  index.html index.htm;
            }
        }
        ```
      * `kubectl create -f fortune-pod-https.yml`，创建pod
      * `kubectl port-forward fortune-https 8443:443`，访问https://127.0.0.1:8443/
      * `kubectl exec fortune-https -c web-server -- mount | grep certs`，可以看出Secrets是存在内存里的，如图：  
        ![pod-secrets.png](https://images.gitee.com/uploads/images/2019/0223/180030_ad1cc519_5849.png "pod-secrets.png")
  * 暴露Secrets到环境变量，代码如下：
    ```
    env:
    - name: FOO_SECRET
        valueFrom:
          secretKeyRef:
            name: fortune-https
            key: foo
    ```
    虽然Kubernetes支持将Secrets暴露给环境变量，但是不建议这样做。因为应用程序在崩溃时会dump环境变量信息，有的应用还会在启动时，把环境变量打到日志里，会造成敏感信息泄露；另外子进程会继承主进程所有的环境变量，如果你的应用运行了第三方的类库，也可能会造成敏感信息泄露

image pull Secrets
  * 私有镜像的拉取需要认证，当部署一个pod时，Kubernetes需要拉取镜像，如果是私有镜像，则Kubernetes需要提供证书
  * 拉取私有镜像需要做两件事
      * 创建一个Secret，存储Docker registry的证书
      * 在pod定义时，在imagePullSecrets字段上引用创建的Secret
  * 实例
      * `kubectl create secret docker-registry mydockerhubsecret --docker-username=yourdockername --docker-password=yourpassword --docker-email=youremail`，创建一个名为mydockerhubsecret的secret
      * 使用以前的dockerfile，创建一个镜像refactor2/fortuneprivate，上传到dockerhub，在dockerhub上把这个镜像标为私有
      * `kubectl create -f fortune-pod-private.yml`，创建pod，设置imagePullSecrets
      * `kubectl get pod`，可以发现fortuneprivate正在运行，如果不指定imagePullSecrets，查看日志会报拉取镜像失败
  * 如果在每一个pod上都指定imagePullSecrets会比较麻烦，可以通过把Secrets绑定到ServiceAccount上，这样使用这个ServiceAccount的pod会被自动添加上imagePullSecrets