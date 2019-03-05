# 使用statefulset部署es集群

## 使用ceph做持续化存储
### 配置ceph-secret
+ 获取base64加密的key:`grep key /etc/ceph/ceph.client.admin.keyring |awk '{printf "%s", $NF}'|base64`
+ 替换ceph-secret.yaml文件中的key值为该值
+ 执行`kubectl create -f ceph-secret.yaml`

### 创建Persistent Volumes
+ pv-data.yaml是elastic的data节点的pv配置
+ pv-master.yaml是elastic的master节点的pv配置
+ 具体配置见pv-data.yaml
+ 需要修改monitors项的地址为ceph的服务地址
+ 需要根据修改pool值，以及image、storage、name值
+ 确保在ceph节点上存在该pool池对应的image
	- 列如:
	- 创建pool:`ceph osd pool create {pool-name} {pg-num}` 
	- 创建image:`rbd create {pool-name}/{image-name} --size 1G --image-feature layering`
	- 其中--image-feature为开启的特征
	- 如果在ceph节点上使用`rbd map {pool-name}/{image-name}`无法挂载成功，则很可能是不支持该特征.
+ 执行`kubectl create -f pv-data.yaml`
+ 创建elastic的master节点的pv：`kubectl create -f pv-master.yaml`
### 创建具有1个master和1个data节点的elastic集群
+ 创建configmap：`kubectl create -f elastic-cluster-config.yaml`
+ 创建服务发现: `kubectl create -f elasticsearch-discovery.yaml`
+ 创建master: `kubectl create -f elasticsearch-master.yaml`
+ 创建data: `kubectl create -f elasticsearch-data.yaml`
+ 创建对外服务: `kubectl create -f elasticsearch-svc.yaml`
+ 确保存在对应的pv
+ 确保elasticsearch-master.yaml和elasticsearch-data.yaml文件中的最后一行的storage值的大小与对应的pv相同

### 扩展data节点为2个
+ 复制pv-data.yaml 为pv-data1.yaml
+ 修改pv-data1.yaml文件中的name、image的值
+ 创建pv `kubectl create -f pv-data1.yaml`
+ 扩展: `kubectl scale --replicas=2 statefulset/elasticsearch-data`
+ 应确保对应的pv数量与要创建的副本数一致

### 扩展master节点为3个
+ 为了防止脑裂，需要master数为奇数,以及discovery.zen.minimum_master_nodes（total_master/2+1）的值
+ 删除存在的configmap: `kubectl delete -f elastic-cluster-config.yaml`
+ 修改elastic-cluster-config.yaml中的discovery.zen.minimum_master_nodes值为3
+ 复制pv-master.yaml 为pv-master1.yaml 和pv-master2.yaml
+ 记得在ceph节点上创建对应的image
+ 修改pv-master1.yaml和pv-master2.yaml中的name、image值
+ 创建pv:`kubectl create -f pv-master1.yaml;kubectl create -f pv-master2.yaml`
+ 中止当前的master:`kubectl scale --replicas=0 statefulset/elasticsearch-master`
+ 重新创建configmap:`kubectl create -f elastic-cluster-config.yaml`
+ 扩展:`kubectl scale --replicas=3 statefulset/elasticsearch-master`
