# 环境：一个master节点，四个node节点
## master节点ip
  - 172.21.16.244
## node节点ip：
  - 172.21.16.240
  - 172.21.16.231
  - 172.21.16.202
  - 172.21.16.55

# 以下是每一个节点上均进行操作

## 1、服务器添加阿里云yum源
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
  http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
## 2、重新建立yum缓存
```
  # yum -y install epel-release &&yum clean all &&yum makecache
```
  - 记得同步系统的时间
## 3、关闭swap
```
  # sudo swapoff -a
```
## 4、配置转发请求
```
  # cat <<EOF > /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  vm.swappiness=0
  EOF
  # sysctl --system
```
## 5、安装docker
```
  # yum -y install docker
  # systemctl enable docker && systemctl start docker
```
## 6、安装k8s 需要的插件
```
  # yum -y install kubelet kubeadm kubectl kubernetes-cni
  # systemctl enable kubelet && systemctl start kubelet
```
  - 修改为 kubelet 为Cgroup模式
```
# vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
systemctl daemon-reload && systemctl restart kubelet
```
## 7、新建一个shell 拉取镜像到本地(所有节点均操作)
```
 #!/bin/bash
images=(kube-proxy:v1.13.0 kube-scheduler:v1.13.0 kube-controller-manager:v1.13.0 kube-apiserver:v1.13.0 etcd:3.2.24 coredns:1.2.6 pause:3.1 kubernetes-dashboard-amd64:v1.10.0 kubernetes-dashboard-init-amd64:v1.0.1  k8s-dns-sidecar-amd64:1.14.9 k8s-dns-kube-dns-amd64:1.14.9 k8s-dns-dnsmasq-nanny:1.15.0 heapster:v1.5.2 kubernetes-dashboard-arm:v1.10.0)
for imageName in ${images[@]} ; do
docker pull xxlaila/$imageName
docker tag xxlaila/$imageName k8s.gcr.io/$imageName
docker rmi xxlaila/$imageName
done
```
# 以下操作是在k8s的master进行操作

## 8、初始化相关镜像
```
# kubeadm init --kubernetes-version=v1.13.0 --pod-network-cidr=10.244.0.0/16
```
  - 记下这句话，后面node节点加入需要
```
kubeadm join 172.21.17.4:6443 --token 0mdk7x.du3cn19qm1jl2b0e --discovery-token-ca-cert-hash sha256:19bf79b41a931735b1f2f5138e1daa436ab26a4f19781ccf2015cff749ddb4b9
```
## 9 执行创建目录（后面在生成证书的时候需要）
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## 10、查看验证
```
# kubectl get componentstatus
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
```
## 11、安装flannel网络(配置文件和目录每个node都要建立)
```
# mkdir -p /etc/cni/net.d/
# cat <<EOF> /etc/cni/net.d/10-flannel.conf
{
“name”: “cbr0”,
“type”: “flannel”,
“delegate”: {
“isDefaultGateway”: true
}
}
EOF
# mkdir /usr/share/oci-umount/oci-umount.d -p
# mkdir /run/flannel/
# cat <<EOF> /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.1.0/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
EOF
```
###11.1、建立flannel网络
```
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```
## 12、查看命名空间
```
# kubectl get ns
NAME          STATUS   AGE
default       Active   27m
kube-public   Active   27m
kube-system   Active   27m
```
##13、 查看system的pod
```
# kubectl get pods -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-4gfvd                     1/1     Running   0          27m
coredns-86c58d9df4-cxtz5                     1/1     Running   0          27m
etcd-k8s-zxc-test-3.kxl                      1/1     Running   0          26m
kube-apiserver-k8s-zxc-test-3.kxl            1/1     Running   0          26m
kube-controller-manager-k8s-zxc-test-3.kxl   1/1     Running   0          26m
kube-flannel-ds-nh95x                        1/1     Running   0          15m
kube-proxy-kvlng                             1/1     Running   0          27m
kube-scheduler-k8s-zxc-test-3.kxl            1/1     Running   0          27m
```
## 14、以下是每个node节点执行
```
# kubeadm join 172.21.17.4:6443 --token 0mdk7x.du3cn19qm1jl2b0e --discovery-token-ca-cert-hash sha256:19bf79b41a931735b1f2f5138e1daa436ab26a4f19781ccf2015cff749ddb4b9
```
  - 查看节点是否加入
```
  # kubectl get nodes
```
  - 使用kubectl get pods命令来查看部署状态
```
# kubectl get pods --all-namespaces
```
## 15、安装kubernetes dashboard
  - 下载官网的dashboard文件
  - 修改kubernetes-dashboard.yaml文件,用修改之后的kubernetes-dashboard.yaml
```
# git clone https://github.com/xxlaila/kubernetes-yaml.git
```
  - 执行创建dashboard
```
# kubectl apply -f kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```
## 16、查看dashboard部署是否成功
```
# kubectl get pods -n kube-system
```
## 17、查看dashboard info
```
# kubectl describe svc kubernetes-dashboard -n kube-system
```
## 18、查看dashboard部署在那个节点
```
# kubectl  get pods -n kube-system -o wide
```
## 19、查看service 节点端口
```
# kubectl get service -n kube-system -o wide
```
## 20、创建dashboard admin 账户
```
# kubectl apply -f admin-user.yaml
```
  - 获取tokens
```
# kubectl describe serviceaccount admin -n kube-system
Name:                admin
Namespace:           kube-system
Labels:              k8s-app=kubernetes-dashboard
Annotations:         kubectl.kubernetes.io/last-applied-configuration:
                       {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"admin","namesp...
Image pull secrets:  <none>
Mountable secrets:   admin-token-kxs6k
Tokens:              admin-token-kxs6k
Events:              <none>
```
  - 查看token 信息
```
# kubectl describe secret admin-token-kxs6k -n kube-system
```
## 21、利用节点ip+30001 端口进行访问
```
访问之前需要在master节点生成证书，把证书(kubecfg.p12)下载到本地，进行导入到浏览器，这里使用火狐浏览器，google浏览器导入不成功，生产证书
之前记得第9步已操作
```
  - 生产证书
```
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```
  - kubernetes dashboard v1.10.0使用的是双因子登陆，默认token失效的时间是900秒，15分钟，每15分钟就要进行一次认证。
  - 我们可以功过修改token-ttl参数来设置，主要是修改dashboard.yaml文件，并重新建立即可
## 22、在配置文件里面添加
```
ports:
- containerPort: 8443
  protocol: TCP
args:
  - --auto-generate-certificates
  - --token-ttl=43200
```
  - 重建dashboard

  - 通过利用http添加端口30001，然后利用tonken进行验证登陆

  - 安装失败清理环境
```
# kubeadm reset
```
  - 查看加入集群token
```
# kubeadm token list
```


