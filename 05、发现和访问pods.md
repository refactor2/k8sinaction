1、Services
--
Kubernetes通过暴露Services来访问Services后面的pods，可以通过kubectl expose来暴露一个Services，也可以通过yml创建   
  1. 创建一个名为netcorek8ssvc的Services，对外监听13300端口，向容器43300端口转发，选择标签apprs: netcorek8s的pod，`kubectl create -f 05、netcorek8s-svc.yml`  

