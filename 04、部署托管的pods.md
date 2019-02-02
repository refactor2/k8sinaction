1、liveness probes
--
pod被调度到一个node上后，在node上的Kubelet会运行pod里的container，当pod里的container的主进程挂掉 or 不能访问 or 内存溢出，Kubelet会自动重启container。

Kubernetes保证container一直在运行，是通过node节点Kubelet来处理的，和master节点没有关系。  
手动创建的pod在node节点挂掉后，则不会被自动调度到另外一个node上，通过ReplicationControllers类似的机制创建的pod会被自动调度到另外一个node上。

Kubernetes通过liveness探针判断container是否存活，liveness探针包括3种：
  * http get，执行一个http请求到container里指定的地址，如果探针得到响应，且响应code是`2xx` 0r `3xx`，则认为正常
  * tcp socket，打开一个tcp连接到container指定端口，如果成功连接，则认为正常
  * exec，在container里执行一个命令，如果命令返回的code是`0`，则认为正常
实例，创建一个http get探针
  1. 修改项目代码，添加HealthController，代码如下：
        ```
        [Route("[controller]")]
        [ApiController]
        public class HealthController : Controller
        {
            private static int requestCount = 0;
            
            [HttpGet]
            public ActionResult<string> Get()
            {
                requestCount++;
                if (requestCount>3)
                {
                    return new BadRequestResult();
                }
                return "hello world";
            }
        }
        ```
  2. 制作镜像，推送到dockerhub，参考02、部署.netcore到k8s.md  
        ```
        docker build -t refactor2/netcorek8s:liveness .  
        docker login --username=refactor2  
        docker push refactor2/netcorek8s:liveness  
        ```
  3. 手动创建pod  
        `kubectl create -f netcorek8s-liveness.yml`
  4. 查看container重启原因  
        `kubectl describe pod netcorek8s-liveness`
  5. liveness配置
        通过`kubectl describe`命令可以看到  
        Liveness:http-get http://:43300/health delay=0s timeout=1s period=10s #success=1 #failure=3
        * delay，探针首次执行时间，delay=0s表示探针在container启动后立即执行
        * timeout，探针等待container返回结果的超时时间
        * period，探针每隔多长时间执行一次检测
        * failure，探针在连续几次检查失败后，进行container重启
  6. 指定liveness配置，一定需要指定initialDelaySeconds，给container留够启动时间，避免一直重启
        ```
        livenessProbe:
          httpGet:
            path: /health
            port: 43300
          initialDelaySeconds: 15
        ```

2、 ReplicationController
--
  1. ReplicationController有三个重要部分：
      * label selector，ReplicationController选中那些pod
      * replicas count，保证多少个副本在运行
      * pod template，用来创建新pod的模板  

  2. 在创建ReplicationController时，可以不指定label selector，Kubernetes会从pod template中获取  

  3. `kubectl get rc`，获取ReplicationController的信息，如图：
      * ![kubectl-get-rc](https://images.gitee.com/uploads/images/2019/0202/173537_99c11aed_5849.png "04、getrc.png") 
      * DESIRED：期望pod数量  
      * CURRENT：目前pod数量  
      * READY：准备好的pod数量，指readiness探针检查正常的pod  

  4. `kubectl describe rc netcorek8src`，查看ReplicationController详细信息  

  5. 可以通过改变pod的label信息，使pod脱离ReplicationController的控制
      * `kubectl label pod netcorek8src-f6fkm app=foo --overwrite`，把podnetcorek8src-f6fkm的label标签由netcorek8s改成foo，这样会让ReplicationController重建一个新的pod    
      * `kubectl get pod -L app`，获取pod的app标签value  

  6. ReplicationController的pod template可以随时更改，修改后，对于之前创建的pod无影响，只会影响新创建的pod。如果不主动删除以前的pod，则也不会产生新的pod
      * `kubectl edit rc netcorek8src`，编辑netcorek8src，编辑完保存后，会时时生效  

  7. 水平扩展pod数量，两种方式：  
      * `kubectl scale rc netcorek8src --replicas=10`  
      * `kubectl edit rc netcorek8src`，修改spec.replicas的值  

  8. 当删除一个ReplicationController时，属于这个ReplicationController的pod也会被删除，可以通过指定--cascade=false来保留pod，`kubectl delete rc netcorek8src --cascade=false`  

3、ReplicaSet
--
  1. ReplicaSet的功能类似与ReplicationController，比ReplicationController功能更强大，用于替换ReplicationController，后面ReplicationController会被废弃。一般情况下，我们并不会直接操作ReplicationController或ReplicaSet，而是通过Deployment部署  

  2.  ReplicaSet在pod选择的时候，可以支持更多选择：
      *  in，label的value必须是指定的values中的一个
      * not in
      * exists，lable中包含指定的key，不管value是什么
      * doesnotexist

  3. 如果指定了多个表达式，所有表达式必须都是true才会匹配到pod。如同时指定了`matchLabels` 和`matchExpressions`时，必须全部满足，这样的pod才会被选择  

  4. 删除ReplicaSet时，同ReplicationController一样，也会删除被选择的pod

4、DaemonSet
--
  1. DaemonSet适用于需要在每个node节点上都部署一个pod，即每个node节点有且只有一个pod。通常跑一些系统级别的pod，如日志收集，资源监控  

  2. DaemonSet可以指定只部署在指定label的node节点上，通过node selector  

  3. 目前node节点可以被设置成unschedulable，但是DaemonSet还是会部署到pod到node上，因为DaemonSet部署pod是不通过调度器  

5、Job
--
  1. 运行pod，执行一次性任务，任务完成后，pod停止  

  2. 实战，创建一个一次执行的Job
      * 创建一个Dockerfile，内容如下：  
          ```
          FROM busybox
          ENTRYPOINT echo "$(date) Batch job starting"; sleep 120; echo "$(date) Finished succesfully"
          ```
      * `docker build -t refactor2/batch-job .`  
        `docker login --username=refactor2`    
       `docker push refactor2/batch-job`
      * `kubectl create -f 04、batch-job.yml`，yml文件里没有指定标签选择器，会从pod模版里自动生成。restartPolicy，重启策略默认是Always，Job需指定成OnFailure 或 Never  
      * `kubectl get job`
      * `kubectl get pod -a`

    3. 在Job里的几个配置：
       * completions，完成pod的数量
       * parallelism，同时运行pod的数量
       * activeDeadlineSeconds，指定pod最长可运行的时间，如果超过这个时间，则被认为job执行失败
       * backoffLimit，job运行失败后，最大重试几次
    
  4. Job在运行过程中，还可以通过命令`kubectl scale job batch-job --replicas 3`改变并行度

  5. 创建CronJob，让pod在指定的时间点，或周期性执行， `kubectl create -f 04、cronjob.yaml`




