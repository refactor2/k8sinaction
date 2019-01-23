`kubectl get pod netcorek8s-bdznl -o yaml`，获取一个pod的完整的yaml  
`kubectl explain pods`，当创建一个pod时，可以使用explain辅助查找字段  
`kubectl explain pods.spec`，当创建一个pod时，可以使用explain辅助查找字段，指定子级  
`kubectl create -f netcorek8s-manual.yml`，通过yaml文件创建资源  
`kubectl logs netcorek8s-manual`，查看pod的日志，注意Container的日志会每天自动滚动记录，或者达到10M时滚动记录，kubectl logs只显示最后一次的日志  
`kubectl port-forward netcorek8s-manual 43301:43300`，在本地暴露一个端口，让pod可以访问，用于快速调试，可以通过http://localhost:43301/api/values或http://127.0.0.1:43301/api/values访问  

