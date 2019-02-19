如同Docker的Volume一样，Kubernetes提供类似的功能。保证容器在重启后可以读到上个容器写入到Volume的数据。  
Kubernetes volumes是定义在pod上，如同container一样。 pod里的所有container都可以使用pod里定义的volume，但是每个container都需要挂载volume，才能读写volume数据  
Kubernetes目前支持的volume类型有：
  * emptyDir，一个空目录，用于存储临时数据
  * gitRepo，初始一个空目录，然后clone一个git地址
  * hostPath，挂载一个node节点的目录
  * nfs，NFS网络存储
  * gcePersistentDisk，azureDisk等，特定的云厂商提供
  * cephfs，glusterfs等，其它类型的网络存储
  * configMap, secret, downwardAPI，特殊类型的volume，用作暴露Kubernetes的元数据给pod里的应用
  * persistentVolumeClaim，动态提供存储的声明

emptyDir
  * emptyDir卷在pod启动时会挂载一个空目录，pod里的应用可以读写这个目录，这个目录的生命周期和pod的一致，emptyDir卷里的数据会随着pod的删除而消失
  * emptyDir卷在同一个pod里不同容器之前共享数据时非常有用。如果容器被设置了不可写，也可以通过挂载emptyDir卷做一些临时操作
  * 实例：
      * 实现效果，一个pod里有两个容器，一个容器使用命令`fortune`(这个命令每次执行会返回一个英文谚语)往自己的文件`/var/htdocs/index.html`写入一段话，另一个容器使用nginx加载目录下的`/usr/share/nginx/html`html
      * 准备脚本，fortuneloop.sh
        ```
        #!/bin/bash
        trap "exit" SIGINT
        mkdir /var/htdocs
        while :
        do
          echo $(date) Writing fortune to /var/htdocs/index.html
          /usr/games/fortune > /var/htdocs/index.html
          sleep 10
        done
        ```
      * 准备Dockerfile
        ```
        FROM ubuntu:latest
        RUN apt-get update && apt-get -y install fortune
        ADD fortuneloop.sh /bin/fortuneloop.sh
        ENTRYPOINT /bin/fortuneloop.sh
        ```