1、下载安装必要的软件
--
由于国内墙的原因，可以使用我下载好的版本
* 下载windows版[kubectl](https://pan.baidu.com/s/10HoOk8QWLyGJXCciCp_nIg)
* 下载windows[阿里云团队](https://github.com/AliyunContainerService/minikube)修改版[minikube](https://pan.baidu.com/s/10HoOk8QWLyGJXCciCp_nIg)，对应minikube版本0.30.0，Kubernetes 1.10+ release
* 下载安装[VirtualBox](https://download.virtualbox.org/virtualbox/6.0.2/VirtualBox-6.0.2-128162-Win.exe)，也可以使用win10自带的Hyper-V，这里使用VirtualBox  

2、添加环境变量
--
* 将下载好的kubectl.exe，minikube.exe 放到文件夹里，如C:\Kubernetes\Minikube\
* 把文件夹C:\Kubernetes\Minikube\加到环境变量里

3、启动minikube&验证
--
* 用管理员启动cmd，执行命令：    
`minikube start --registry-mirror=minikube start --registry-mirror=https://registry.docker-cn.com --kubernetes-version v1.10.0 --cpus 2 --memory 4096`    
注意需要指定registry-mirror，避免被墙
* 命令执行过程中，需要等一段时间，可以查看执行进度，如图:  
![minikubestart.png](https://images.gitee.com/uploads/images/2019/0122/170754_38f09a3e_5849.png "minikubestart.png")
* 验证安装情况  
`minikube dashboard`，打开dashboard  
`minikube ip`，获取集群IP  
`kubectl version`，查看客户端，服务端版本信息 

4、停止&再次启动
--
* `minikube stop`  
* `minikube start`  

5、排错指南
-- 
* `minikube start`启动，Error restarting cluster:  restarting kube-proxy: waiting for kube-proxy to be up for configmap update: timed out waiting for the condition
    * 方案一：  
        * `minikube delete`  
        * `minikube start`  
    * 方案二： 
        * `minikube start`
        * `minikube ssh` 进入minikube VM
        * `ifconfig` 获取minikube VM IP地址，类似与192.168.xx.x，记住IP地址，exit退出minikube VM
        * `minikube start --registry-mirror=http://f1361db2.m.daocloud.io --kubernetes-version v1.10.0 --cpus 2 --memory 4096 --docker-env HTTP_PROXY=http://192.168.99.115:8118 --docker-env HTTPS_PROXY=http://192.168.99.115:8118 --docker-env NO_PROXY=127.0.0.1/24`
