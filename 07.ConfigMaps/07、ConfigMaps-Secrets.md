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
      * 