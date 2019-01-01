# heapster

## 1、heapster 介绍
- Heapster是容器集群监控和性能分析工具,支持Kubernetes和CoreOS。Kubernetes有个监控agent—cAdvisor。在每个kubernetes
Node上都会运行cAdvisor,它会收集本机以及容器的监控数据(cpu,memory,filesystem,network,uptime)。heapster需同步部署机器和被采集机器的时间
## 2、部署heapster使用的组件
- 这里使用了heapster、grafana、influxdb
- grafana：做页面展示用的
- influxdb：存储数据
## 3、下载heapster关联镜像（所有节点均操作）
```
 #!/bin/bash
images=(heapster-grafana-amd64:v5.0.4 heapster-amd64:v1.5.4 heapster-influxdb-amd64:v1.5.2)
for imageName in ${images[@]} ; do
docker pull xxlaila/$imageName
docker tag xxlaila/$imageName k8s.gcr.io/$imageName
docker rmi xxlaila/$imageName
done
```
## 4、执行创建
```
# kubectl create -f ./
```
- 查看pod状态(可以看到相关的三个pod均正常)
```
# kubectl get pods --namespace=kube-system
```
