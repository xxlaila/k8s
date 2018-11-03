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
  # cat <<EOF > /etc/yum.repos.d/kubernetes.repo
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
## 7、新建一个shell 拉取镜像到本地
```
  #!/bin/bash
  images=(kube-proxy-amd64 kube-scheduler-amd64 kube-controller-manager-amd64 kube-apiserver-amd64 etcd coredns pause-amd64 kubernetes-dashboard-amd64 kubernetes-dashboard-init-amd64 k8s-dns-sidecar-amd64 k8s-dns-kube-dns-amd64 k8s-dns-dnsmasq-nanny-amd64 heapster-amd64)
  for imageName in ${images[@]} ; do
  docker pull xxlaila/$imageName
  docker tag xxlaila/$imageName k8s.gcr.io/$imageName
  docker rmi xxlaila/$imageName
  done
  - docker tag cf80e75b1030 k8s.gcr.io/pause:3.1
```
# 以下操作是在k8s的master进行操作

## 8、初始化相关镜像
```
  # kubeadm init --kubernetes-version=v1.11.0 --pod-network-cidr=10.244.0.0/16
```
  - 记下这句话，后面node节点加入需要
```
  # kubeadm join 172.21.16.244:6443 --token bttbal.356uhebshtqzor6x --discovery-token-ca-cert-hash sha256:0a48f67994da476b646ab8fc15b99d5dd67c3f9bce02f693a927e9bc590976e5
```
## 9、配置kubectl认证信息
```
  # export KUBECONFIG=/etc/kubernetes/admin.conf
```
  - 永久有效
```
  # echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile
```
## 10、安装flannel网络
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
## 建立flannel.yml文件
```
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "type": "flannel",
      "delegate": {
        "isDefaultGateway": true
      }
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.10.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conf
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.10.0-amd64
        command: [ "/opt/bin/flanneld", "--ip-masq", "--kube-subnet-mgr" ]
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
```
  - 执行
```
  # kubectl create -f ./flannel.yml
```
  - 查看
```
  # kubectl get nodes
```
# 以下是每个node节点执行 
```
 # kubeadm join 172.21.16.244:6443 --token bttbal.356uhebshtqzor6x --discovery-token-ca-cert-hash sha256:0a48f67994da476b646ab8fc15b99d5dd67c3f9bce02f693a927e9bc590976e5
```
  - 查看
```
  # kubectl get nodes
```
## 11、Dashboard的配置
  - 下载Dashboard的yaml文件
```
  # https://github.com/xxlaila/k8s.git
  # cd kubernetes-dashboard
  # kubectl  -n kube-system create -f .

```
### 启动容器
```
# kubectl -f ./dashboard-admin.yaml create
```
## 12、访问master节点的
```
http://172.21.16.244:30090/
```
  -  后面所有的节点加30090都可以反问dashboard页面  

