`kubectl get pod netcorek8s-bdznl -o yaml`，获取一个pod的完整的yaml  
`kubectl explain pods`，当创建一个pod时，可以使用explain辅助查找字段  
`kubectl explain pods.spec`，当创建一个pod时，可以使用explain辅助查找字段，指定子级  
`kubectl create -f netcorek8s-manual.yml`，通过yaml文件创建资源  
`kubectl logs netcorek8s-manual`，查看pod的日志，注意Container的日志会每天自动滚动记录，或者达到10M时滚动记录，kubectl logs只显示最后一次的日志  
`kubectl port-forward netcorek8s-manual 43301:43300`，在本地暴露一个端口，让pod可以访问，用于快速调试，可以通过http://localhost:43301/api/values或http://127.0.0.1:43301/api/values访问  
`kubectl get pod --show-labels`，显示pod的所有label信息  
`kubectl get pod -L creation_method,env`，显示pod的指定的label信息  
`kubectl label pod netcorek8s-manual creation_method=manual`，为已存在的pod添加label  
`kubectl label pod netcorek8s-manual-v2 env=debug --overwrite`，为已存在的pod修改label  
`kubectl get pod -l creation_method=manual`，筛选指定的label信息的pod  
`kubectl get pod -l env`，筛选包含label信息的pod，不管label的值  
label筛选pod可以支持`=`，`!=`，`in`，`not in`，`正则表达式`,也支持多个label同时满足，如：`creation_method=manual，env in (prod,debug)`  
`kubectl describe pod netcorek8s-gpu`，会发现调度失败，因为没有node的label是 gpu: "true"  
`kubectl get ns`，获取当前集群的namespaces  
`kubectl get pod -n kube-system`，获取指定namespaces的pod，不指定默认default  
`kubectl create namespace custom-namespace`，创建namespace  
`kubectl delete ns custom-namespace`，删除namespace，会自动删除此namespace下的pod  
`kubectl create -f netcorek8s-manual.yml -n custom-namespace`，在指定的namespace中创建资源  
`kubectl delete pod netcorek8s-gpu`，在指定的namespace中删除pod  
`kubectl delete pod -l creation_method=manual`，在指定的namespace中删除指定label的pod  
`kubectl delete pod --all`，在指定的namespace中删除所有pod  
`kubectl delete all --all`, 在指定的namespace中删除所有pod，service，replicationcontroller等等  






