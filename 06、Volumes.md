如同Docker的Volume一样，Kubernetes提供类似的功能。保证容器在重启后可以读到上个容器写入到Volume的数据。  
Kubernetes volumes是定义在pod上，如同container一样。 pod里的所有container都可以使用pod里定义的volume，但是每个container都需要挂载volume，才能读写volume数据  
Kubernetes目前支持的volume类型有：
  * emptyDir，一个空目录，用于存储临时数据