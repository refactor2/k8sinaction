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
      * 实现效果，一个pod里有两个容器，一个容器使用命令`fortune`(这个命令每次执行会返回一个英文谚语)往自己的文件`/var/htdocs/index.html`写入一段话，另一个容器使用nginx加载目录下的`/usr/share/nginx/html`html文件
      * 准备脚本，fortuneloop.sh，需注意在windows上拷贝代码保存为文件，可能会导致sh文件不执行，导致容器启动失败。进入pod里调试发现，在执行fortuneloop.sh脚本时，提示： bad interpreter: No such file or directory。这是因为windows和linux系统编码格式不同，在windows系统中编辑的.sh文件可能有不可见字符，在Linux系统下执行会报以上异常信息。 可以在linux系统上新建文件，拷贝代码。如图：  
      ![06、windows不可见字符.png](https://images.gitee.com/uploads/images/2019/0222/221519_7f1fbc4e_5849.png "06、windows不可见字符.png")
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
        ADD fortuneloop.sh /bin/fortuneloop.sh
        RUN chmod +x /bin/fortuneloop.sh
        # RUN mkdir /var/htdocs

        ENTRYPOINT /bin/fortuneloop.sh
        ```
      * `kubectl create -f 06、fortune-pod.yml`，创建一个pod，两个容器
      * `kubectl port-forward fortune 8080:80`，在本地暴露一个端口，让pod可以访问，用于快速调试，打开浏览器http://127.0.0.1:8080访问
  * emptyDir卷是在pod所在的node节点硬盘上创建一个临时文件夹，可以指定存储介质，如使用内存
    ```
      volumes:
      - name: html
        emptyDir: 
          medium: Memory 
    ```
gitRepo
  * gitRepo volume是基于emptyDir实现，在pod开始时，容器启动之前，初始化一个emptyDir，然后执行git clone命令，将git服务器上的文件下载到本地
  * 容器启动后，git服务器的文件变更不会同步到容器里，除非删除pod，重建
  * 实例
      * 实现效果，clone https://github.com/refactor2/k8sgitrepotest.git 的master分支，将clone的文件直接放在volume的根目录，通过nginx访问
      * `kubectl create -f 06、gitrepo-volume.yml`
      * `kubectl port-forward gitrepo-volume 8080:80`，访问http://127.0.0.1:8080
  * gitRepo目前不能克隆私有仓库，需要通过另外的机制来实现git变更同步和私有仓库拉取

 hostpath
   * hostpath指定pod所在的node节点的一个目录，挂载到容器内。在同一个node节点上的pod，如果挂载了，都可以访问。不像gitRepo和emptyDir一样，pod删除后，如果pod仍被部署在同一个node上，新的pod依然可以访问上一个pod记录的数据。如果Kubernetes集群上有多个node节点，使用hostpath需要注意
   * `kubectl describe pod kube-addon-manager-minikube -n kube-system`，可以看到，kube-addon-manager-minikube挂载了两个hostpath，一个名称为addons(路径/etc/kubernetes/)，一个名称为kubeconfig(路径/var/lib/minikube/)，如图：  
   ![06、minikubehostpath.png](https://images.gitee.com/uploads/images/2019/0222/221332_1f84f61b_5849.png "06、minikubehostpath.png")
   * 建议hostPath仅用在单个pod读写node上的文件，不要用在多个pod之间当互通的存储介质
   * 实例
      * `kubectl create -f 06、mongodb-pod-hostpath.yml`，在pod所在的node节点上目录/tmp/mongodb挂载到容器的目录/data/db
      * `kubectl exec -it mongodb mongo`，连接到mongo服务。`use refactor`，创建refactor数据库，`db.refactor.insert({name:'refactor'})`，插入数据，`db.refactor.find()`，查询数据
      * `kubectl delete -f 06、mongodb-pod-hostpath.yml`，删除mongodb pod
      * `kubectl create -f 06、mongodb-pod-hostpath.yml`，重建pod，连接到mongo服务，查询数据，确认数据还在
    
PersistentVolumes，PersistentVolumeClaims
  * 在pod定义时，直接挂载持久卷，需要开发人员了解具体的存储类型，如指定node节点的目录，如果使用nfs，则需要知道nfs的ip的地址，这样不利于将底层的存储和pod定义分开来。
  * 使用PersistentVolumes和PersistentVolumeClaims步骤：
      1. 集群管理员（一般指运维）创建好底层存储，如nfs，hostpath
      2. 集群管理员（一般指运维）创建PersistentVolumes发送给api server，指向底层存储
      3. 开发人员创建PersistentVolumeClaims发送给api server
      4. Kubernetes找到适合这个声明的PV
      5. 开发人员声明pod时，引用PVC
  * PersistentVolumes是集群级别的，不属于任何namespace
  * PersistentVolumeClaims是namespace级别的
  * 实例
      * `kubectl create -f 06、mongodb-pv-hostpath.yml`，创建PV
      * `kubectl get pv`，查看创建的PV
      * `kubectl create -f 06、mongodb-pvc.yml`，创建pvc
      * `kubectl get pv`，查看创建的PV，发现PV的状态是Bound，如图：  
        ![06、pvstatus.png](https://images.gitee.com/uploads/images/2019/0222/221452_b807ee93_5849.png "06、pvstatus.png")
          * ACCESS MODES的类型
              * RWO，ReadWriteOnce，只有一个node节点可以读写volume
              * ROX，ReadOnlyMany，多个node节点可以读volume
              * RWX，ReadWriteMany，多个node节点可以读写volume
      * `kubectl delete pod mongodb`，先删除以前的创建的mongodb，避免挂载hostpath相同目录存在问题
      * `kubectl create -f 06、mongodb-pod-pvc.yml`，创建pod，使用PVC。连接到mongo服务，查询数据，确认数据还在
  * PersistentVolumes的回收策略
      * 实例
          * `kubectl delete pod mongodbpvc`，删除pod
          * `kubectl delete pvc mongodb-pvc`，删除pvc
          * `kubectl create -f 06、mongodb-pvc.yml`，再次创建pvc
          * `kubectl get pvc`，发现状态是Pending
          * `kubectl get pv`，发现状态是Released，而不是Available。这是因为在创建pv的时候，给的回收策略是Retain，如图：  
            ![06、pvcstatus.png](https://images.gitee.com/uploads/images/2019/0222/221440_66226351_5849.png "06、pvcstatus.png")
      * 回收策略
          * Retain，当pvc删除的时候，不删除volume里的数据。要想使pv再次被绑定，需要手动删除pv，然后创建pv。如果删除pv，pv里指定的volume数据，你可以不删除，让下一个pod再次使用
          * Recycle，当pvc删除的时候，会删除volume里的数据，其他pvc可以再次绑定
          * Delete，当pvc删除的时候，会删除volume里的数据，也会删除pv资源
      * 可以随时改变pv的回收策略，如修改persistentVolumeReclaimPolicy为Delete，`kubectl apply -f 06、mongodb-pv-hostpath.yml`应用

StorageClass
  * PersistentVolumes可以隐藏开发人员对底层的存储，但是需要运维人员提前定义好。Kubernetes提供了一个更简便的方式。运维人员只需提前准备好PersistentVolume provisioner和定义好一个或多个StorageClass，开发人员在PVC里引用StorageClass，由Kubernetes根据StorageClass自动创建PersistentVolumes
  * Kubernetes为云厂商默认提供了provisioner，运维人员不用自己创建provisioners，但是如果自己部署Kubernetes集群，则需要提供provisioner，一般采用nfs
  * 运维人员会根据不同的场景创建不同的StorageClass，如有的使用SSD，有的使用普通硬盘。由开发人员选择使用哪个StorageClass
  * StorageClass和PersistentVolumes一样是集群级别的，不属于任何namespace
  * 实例
      * `kubectl create -f 06、storageclass-fast-hostpath.yml`，创建一个名为fast的StorageClass，使用k8s.io/minikube-hostpath provisioner
      * `kubectl create -f 06、mongodb-pvc-sc.yml`，创建一个pvc，绑定fast StorageClass
      * `kubectl get pvc mongodb-pvc-sc`，查看创建的pvc
      * `kubectl get pv`，查看由StorageClass创建的pv
  * 如果pvc中不指定StorageClass，会使用集群中默认的StorageClass。如果不想使用StorageClass，想手动指定pvc绑定的pv，可以设置StorageClass为""，如文件06、mongodb-pvc.yml
  * `kubectl get sc`，获取集群中存在哪些StorageClass，哪个StorageClass的默认的，如图：  
    ![06、sc.png](https://images.gitee.com/uploads/images/2019/0222/221506_737deaeb_5849.png "06、sc.png")
  * `kubectl explain sc`，查看StorageClass的定义，可以指定reclaimPolicy，默认是Delete。`reclaimPolicy: Retain`
  * 使用StorageClass的步骤：
      1. 集群管理员（一般指运维）创建PersistentVolume provisioner
      2. 集群管理员（一般指运维）创建一个或多个StorageClass，并指定哪个是默认的StorageClass
      3. 开发人员创建PersistentVolumeClaims，引用提供的StorageClass，StorageClass查找引用的provisioner，provisioner根据pvc的请求创建一个pv，然后将创建的pv和pvc进行绑定
      4. 开发人员声明pod时，引用PVC
      