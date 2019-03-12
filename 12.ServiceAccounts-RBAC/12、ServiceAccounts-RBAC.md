ServiceAccounts
  * `kubectl create -f dockerhubsecret-sa.yml`，创建一个名为dockerhubsecret-sa的ServiceAccount，其中包含dockerhub密钥
  * `kubectl get sa`，查看创建的ServiceAccount
  * `kubectl create -f pod-private-sa.yml`，创建一个应用私有镜像的pod，`kubectl get pod`，验证pod是否执行成功
  * `kubectl describe sa dockerhubsecret-sa`，得到ServiceAccount的token为dockerhubsecret-sa-token-mlfbt
  * `kubectl describe secret dockerhubsecret-sa-token-mlfbt`，得到token值
  * `kubectl exec -it pod-private-sa -c html-generator cat  /var/run/secrets/kubernetes.io/serviceaccount/token`，进入容器里查看token和上部得到的一致
  
RBAC
  * 1.8版本，RBAC常见正式被集群默认使用。使用RBAC可以防止没有权限的用户查看或修改集群状态
  * RBAC验证有4个资源
      * Roles、ClusterRoles，指定什么样的动词可以在什么样的资源上执行
      * RoleBindings、 ClusterRoleBindings，绑定用户，用户组，ServiceAccount到特定的Roles或ClusterRoles上
  * `kubectl delete clusterrolebinding permissive-binding`，删除权限绑定
  * 实例
      * `kubectl create ns foo`，创建namespace
      * `kubectl run test --image=refactor2/kubectl-proxy:1.10.0 -n foo`，名foo namespace里创建一个deployment
      * `kubectl create ns bar`，创建namespace
      * `kubectl run test --image=refactor2/kubectl-proxy:1.10.0 -n bar`，名foo namespace里创建一个deployment
      * `kubectl get pod -n foo`，在另一个终端中执行`kubectl get pod -n bar`，查找到pod
      * `kubectl exec -it test-5d565d7dcc-2k49t -n foo sh`，进入容器内部，`apk add --no-cache curl`，先安装curl，`curl localhost:8001/api/v1/namespaces/foo/services`，提示services is forbidden，可见RBAC起作用了，默认的ServiceAccount不允许查找和修改资源
      * `kubectl create -f service-reader.yml -n foo`，创建一个读取service的Role。类似命令`kubectl create role service-reader --verb=get --verb=list  --resource=services -n bar`
      * `kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo`，绑定角色
      * 再次进入容器内部，发现可以访问services资源
      * `kubectl edit rolebinding test -n foo`，编辑角色绑定，绑定bar namespace的serviceaccount，这样可以在bar namespace的pod可以访问foo的service，添加如下代码：
        ```
        subjects:
        - kind: ServiceAccount
          name: default 
          namespace: bar 
        ```  
  * ClusterRole只能使用ClusterRoleBinding来绑定，实例如下：
      * `kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes`，创建一个clusterrole，读取persistentvolumes资源
      * `kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default`，绑定
      * `curl localhost:8001/api/v1/persistentvolumes`，验证
  * 查看系统角色
      * `kubectl get clusterrole system:discovery -o yaml`，`kubectl get clusterrolebinding system:discovery -o yaml`
      * `kubectl get clusterrolebindings`，`kubectl get clusterroles`

