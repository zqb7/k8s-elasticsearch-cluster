# elasticsearch集群管理文档
>参考资料
+ https://blog.csdn.net/chenleiking/article/details/79453460
+ https://github.com/pires/kubernetes-elasticsearch-cluster

## 文件清单
+ elasticsearch-master.yaml
+ elasticsearch-discovery.yaml
+ elasticsearch-data.yaml
+ elasticsearch-svc.yaml

### 重要的环境变量
| 变量名  | 默认值  | 说明  |
| ------------ | ------------ | ------------ |
|  cluster.name |   | 集群名字  |
|  node.master | true  | true代表可被选为master  |
|  discovery.zen.minimum_master_nodes | 1  | 最小master数  |
| node.ingest | true| true代表开启预处理 |
| node.data | true| 是否存数据|
| http.enabled | true | ---|
| ES_JAVA_OPTS| | java环境变量值|
| discovery.zen.ping.unicast.hosts | | es绑定的回环地址，data节点主要通过此配置发现master|


## 各文件介绍
1.  elasticsearch-master
> 定义的是elastic的master节点，只负责集群的管理，不包含查询和存储数据

2. elasticsearch-discovery
> 该文件只是为了创建一个服务，使其elastic可以互相通过9300端口互相发现

3. elasticsearch-data
> 负责数据的查询和存储

4. elasticsearch-svc
> 负责向外提供elastic服务，如果有多个elasticsearch-data副本，即为负载均衡入口

## 使用
1.  快速创建一个master、2个data节点的集群
```
kubectl create -f elasticsearch-master.yaml
kubectl create -f elasticsearch-discovery.yaml
kubectl create -f elasticsearch-data.yaml
kubectl create -f elasticsearch-svc.yaml
```
2. 查看pods
```
kubectl get pods
NAME                                    READY     STATUS    RESTARTS   AGE
elasticsearch-data-5fd4fdd6df-77xnk     1/1       Running   0          19s
elasticsearch-data-5fd4fdd6df-fd8l8     1/1       Running   0          19s
elasticsearch-master-6df9f65bbf-b8ktw   1/1       Running   0          19s
```
3. 查看集群状态
```
curl  10.96.50.52:9200/_cluster/health?pretty
{
  "cluster_name" : "es_cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 2,
...}
```
我们可以看到总节点数是3，data_nodes是2。(因为另一个节点是master)

4.  集群扩容
+ 伸缩data节点
```
kubectl scale --replicas=5 deploy/elasticsearch-data
kubectl get pods
结果:
NAME                                    READY     STATUS    RESTARTS   AGE
elasticsearch-data-5fd4fdd6df-77xnk     1/1       Running   0          14m
elasticsearch-data-5fd4fdd6df-9htr6     1/1       Running   0          5m
elasticsearch-data-5fd4fdd6df-fd8l8     1/1       Running   0          14m
elasticsearch-data-5fd4fdd6df-gj9rt     1/1       Running   0          5m
elasticsearch-data-5fd4fdd6df-jhsts     1/1       Running   0          5m
elasticsearch-master-6df9f65bbf-b8ktw   1/1       Running   0          14m
#查看集群健康
curl  10.96.50.52:9200/_cluster/health?pretty
{
  "cluster_name" : "es_cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 6,
  "number_of_data_nodes" : 5,
... }
```
+ 伸缩master节点
	+ `kubectl scale --replicas=3 deploy/elasticsearch-master`
	+ discovery.zen.minimum_master_node(最小master数)，由master总节点决定num=(master_eligible_nodes / 2）+ 1
	+ 如果可用master是3，则该值为(3/2)+1=2(向下取整)
	+ 使用`kubectl edit deploy/elasticsearch-data`修改该值

