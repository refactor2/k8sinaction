StatefulSet和ReplicaSet部署pod的区别：
  * StatefulSets刚开始叫PetSets。小故事，类似于pet和cattle对人的影响，宠物我们一般会起个名字并且精心照料；但是牛我们不会太关心，可以随时替换一个新的，就像ReplicaSet可以替换一个不健康的pod，人们不会太关心。如果一个pet死了，我们会知道，并且期望新的pod需要和上一个pod一样，具有相同的状态和标识
  * ReplicaSet管理的pod类似与cattle，是无状态的，可以在任何时间替换成一个新的pod。有状态的pod被替换成新的pod时，这个pod和以前的pod需要有相同的名称，网络标识，状态
  * StatefulSet会为管理pod分配从0开始递增序号，被用做pod的名称，hostname，存储
  * 无状态的pod可以通过service随便选一个pod提供服务，因为它们基本无差别。但是有状态的pod，客户端可能需要链接特定的pod，因为每个pod都是不一样的。所以StatefulSet的service需要是headless Service，如可以通过a-0.foo.default.svc.cluster.local. 来访问a-0pod，其中a-0为pod名称，foo为headless Service名称，default为命名空间。另外可以通过DNS的SRV记录来查找所有的pod名称
  * StatefulSet在Scale时只会一个一个pod进行的扩容或缩容，如果pod是不健康的，则不允许进行缩容操作
  * StatefulSet可以通过VOLUME CLAIM TEMPLATES为每个pod自动声明一个PVC，并进行引用，即每个pod都会有自己的存储。当进行缩容时，pod会被删除，但是pod引用的PVC不会被删除，因为如果PVC也被删除，则PV里的内容会删除或回收，不能为扩容再次使用

使用StatefulSet
  * 修改netcorek8s源码，实现每个pod可以在自己的存储卷上读写文件，然后通过一个nodeport服务获取所有pod存储的值
      * 在`Startup.cs`中，添加服务 `services.AddHttpClient();`，可以在Controller发送http请求
      * 在Controller里修改post方法，向指定的文件写入传入的数据
        ```
        [HttpPost]
        public string Post(string value)
        {
            CreateDirectory();
            System.IO.File.WriteAllText(filename, value);//文件名称为filename = "data/netcorek8s.txt";
            return $"Data stored on pod {System.Environment.MachineName}";
        }
        ```
      * 在Controller里添加Data方法，从文件里读取写入的数据
        ```
        [HttpGet("data")]
        public string Data()
        {
            var msg = "no post data";
            if (System.IO.File.Exists(filename))
            {
                msg = System.IO.File.ReadAllText(filename);
                if (string.IsNullOrEmpty(msg))
                {
                    msg = "no post data";
                }
            }
            return msg;
        }
        ```
      * 修改Controller里的Get方法，通过headless service找到所有的SRV记录，然后向每个SRV记录发送请求，需在Nuget中引用DnsClient包
        ```
        [HttpGet("{id}")]
        public async Task<string> Get(int id)
        {
            var msg = $"you are hit {System.Environment.MachineName}\n";
            try
            {
                var client = _httpClientFactory.CreateClient();
                var serviceName = "netcorek8s-headless.default.svc.cluster.local";
                var lookup = new LookupClient();
                var result = await lookup.QueryAsync(serviceName, QueryType.SRV);
                if (result.HasError)
                {
                    msg += "查找SRV记录失败";
                    return msg;
                }

                var srvRecords = result.Answers.SrvRecords();
                
                foreach (var item in srvRecords)
                {
                    var response = await client.GetAsync($"http://{item.Target.Value}:43300/api/values/data");

                    msg += $"{item.Target.Value}--{await response.Content.ReadAsStringAsync()}\n";
                }

                return msg;
            }
            catch (Exception ex)
            {
                return ex.ToString();
            }
        }
        ```
  * 构建镜像refactor2/netcorek8s:statefulset，上传至dockerhub
  * `kubectl create -f netcorek8s-headless.yml`，创建名为netcorek8s-headless的service，注意名称要和代码里的对应
  * `kubectl create -f netcorek8s-statefulset.yml`，创建名为netcorek8s的statefulset，副本为2，镜像为refactor2/netcorek8s:statefulset，存储卷由`standard` storageClass自动创建
  * `kubectl get pod`，发现pod名称为netcorek8s-0，netcorek8s-1，且是一个pod创建好后，才会进行下一个pod的创建，如图：  
    ![podlist.png](https://images.gitee.com/uploads/images/2019/0301/150147_01e3def7_5849.png "podlist.png")"pod-yaml.png")
  * `kubectl get pod netcorek8s-0 -o yaml`，查看pod的定义，可以看出由storageClass自动创建了data-netcorek8s-0 pvc，如图：  
    ![pod-yaml.png](https://images.gitee.com/uploads/images/2019/0301/150155_85ecc3da_5849.png "pod-yaml.png")
  * `kubectl get pvc`，如图：  
    ![pvc.png](https://images.gitee.com/uploads/images/2019/0301/150214_ddee0f14_5849.png "pvc.png")
  * `kubectl proxy`，本地开启代理
  * `curl localhost:8001/api/v1/namespaces/default/pods/netcorek8s-0/proxy/api/values`，通过API的方式可以访问应用，如图：  
    ![proxy.png](https://images.gitee.com/uploads/images/2019/0301/150204_57a06137_5849.png "proxy.png")
  * `curl -X POST "localhost:8001/api/v1/namespaces/default/pods/netcorek8s-0/proxy/api/values?           value=hello%20world%20to%20netcorek8s-0."`，向netcorek8s-0 post 数据hello world to netcorek8s-0.
  * `curl localhost:8001/api/v1/namespaces/default/pods/netcorek8s-0/proxy/api/values/data`，访问文件存储的数据
  * `kubectl delete pod netcorek8s-0`，删除netcorek8s-0，由statefulset重建
  * `curl localhost:8001/api/v1/namespaces/default/pods/netcorek8s-0/proxy/api/values/data`，访问文件存储的数据，数据并没有消失
  * `kubectl create -f netcorek8s-nodeport.yml`，创建一个名为netcorek8s-nodeport服务，端口为32002
  * `curl localhost:8001/api/v1/namespaces/default/services/netcorek8s-nodeport/proxy/api/values/data`，访问服务
  * `kubectl run -it srvlookup --image=tutum/dnsutils --rm --restart=Never -- dig SRV netcorek8s-headless.default.svc.cluster.local`，声明一个一次性pod，用完之后删掉，查询headless的SRV记录，如图： ![dig.png](https://images.gitee.com/uploads/images/2019/0301/150138_c541654f_5849.png "dig.png")
  * `curl localhost:8001/api/v1/namespaces/default/services/netcorek8s-nodeport/proxy/api/values/1`，访问服务，如图：  ![srv.png](https://images.gitee.com/uploads/images/2019/0301/150912_da679ac3_5849.png "srv.png")