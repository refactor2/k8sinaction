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
        ![06、minikubehostpath.png](https://images.gitee.com/uploads/images/2019/0222/221332_1f84f61b_5849.png "06、minikubehostpath.png")
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