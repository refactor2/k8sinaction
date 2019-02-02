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
      * ![kubectl-get-rc](https://images.gitee.com/uploads/images/2019/0123/075009_00be8a3d_5849.png "02、进入VM验证应用.png")  
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






