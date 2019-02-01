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
2. ReplicationController