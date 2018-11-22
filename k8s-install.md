# 安装k8s(3master+etcd+node)

[toc]

## 安装步骤

### 设置主机名
```
hostnamectl set-hostname master1
hostnamectl set-hostname master2
hostnamectl set-hostname master3
hostnamectl set-hostname node1
```

### 配置host和节点，追加host
```
echo "10.45.180.73 master1" >> /etc/hosts 
echo "10.66.93.21 master2" >> /etc/hosts 
echo "10.80.240.214 master3" >> /etc/hosts 
echo "10.81.126.152 node1" >> /etc/hosts 
cat /etc/hosts
```

### master1配置ssh，手动添加key，copy-id到其他机器
```
#一路回车即可
ssh-keygen  

#手动copy public key 到其他机器
ssh-copy-id  master2
ssh-copy-id  master3
ssh-copy-id  node1
```

### 停防火墙、关闭Swap、关闭Selinux、设置内核、K8S的yum源、安装依赖包、配置ntp（配置完后建议重启一次）
```
#停防火墙
systemctl stop firewalld
systemctl disable firewalld

#关闭Swap
swapoff -a 
sed -i 's/.*swap.*/#&/' /etc/fstab

#关闭Selinux
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config  

#配置网桥
modprobe br_netfilter
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
ls /proc/sys/net/bridge

#K8S的yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#安装依赖包
yum install -y epel-release
yum install -y yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools wget vim  ntpdate libseccomp libtool-ltdl 

#配置ntp
systemctl enable ntpdate.service
echo '*/30 * * * * /usr/sbin/ntpdate time7.aliyun.com >/dev/null 2>&1' > /tmp/crontab2.tmp
crontab /tmp/crontab2.tmp
systemctl start ntpdate.service
 
#设置内核
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
```

### 设置cfssl环境（master1）
```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
export PATH=/usr/local/bin:$PATH
```


### 创建etcd

#### 创建证书(master1上执行即可)

##### 创建ca-config.json与ca-csr.json文件，并从 CSR json 产生 CA keys 与 Certificate：

```
mkdir -p /etc/etcd/ssl
cd /etc/etcd/ssl
cat > ca-config.json <<EOF
{
"signing": {
"default": {
  "expiry": "8760h"
},
"profiles": {
  "kubernetes-Soulmate": {
    "usages": [
        "signing",
        "key encipherment",
        "server auth",
        "client auth"
    ],
    "expiry": "8760h"
  }
}
}
}
EOF

cat > ca-csr.json <<EOF
{
"CN": "kubernetes-Soulmate",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
  "C": "CN",
  "ST": "shanghai",
  "L": "shanghai",
  "O": "k8s",
  "OU": "System"
}
]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.45.180.73",
    "10.66.93.21",
    "10.80.240.214"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "shanghai",
      "L": "shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes-Soulmate etcd-csr.json | cfssljson -bare etcd

#完成后删除不必要文件：
rm -rf *.json *.csr

#展示证书文件
ll /etc/etcd/ssl
```

#### 分发证书

```
ssh -n master2 "mkdir -p /etc/etcd/ssl && exit"
ssh -n master3 "mkdir -p /etc/etcd/ssl && exit"
scp -r /etc/etcd/ssl/*.pem master2:/etc/etcd/ssl/
scp -r /etc/etcd/ssl/*.pem master3:/etc/etcd/ssl/
```

#### 安装配置etcd (三主节点）
```
yum install etcd -y
mkdir -p /var/lib/etcd
```

##### master1的etcd.service
```
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name master1 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://10.45.180.73:2380 \
  --listen-peer-urls https://10.45.180.73:2380 \
  --listen-client-urls https://10.45.180.73:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://10.45.180.73:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master1=https://10.45.180.73:2380,master2=https://10.66.93.21:2380,master3=https://10.80.240.214:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

##### master2的etcd.service
```
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name master2 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://10.66.93.21:2380 \
  --listen-peer-urls https://10.66.93.21:2380 \
  --listen-client-urls https://10.66.93.21:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://10.66.93.21:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master1=https://10.45.180.73:2380,master2=https://10.66.93.21:2380,master3=https://10.80.240.214:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

##### master3的etcd.service
```
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name master3 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://10.80.240.214:2380 \
  --listen-peer-urls https://10.80.240.214:2380 \
  --listen-client-urls https://10.80.240.214:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://10.80.240.214:2379 \
  --initial-cluster-token etcd-cluster-0 \
--initial-cluster master1=https://10.45.180.73:2380,master2=https://10.66.93.21:2380,master3=https://10.80.240.214:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
#### 添加自启动(etcd至少2个节点启动才能正常工作)
```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

#### 三个节点分别执行
```
etcdctl --endpoints=https://10.45.180.73:2379,https://10.66.93.21:2379,https://10.80.240.214:2379 \
  --ca-file=/etc/etcd/ssl/ca.pem \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem  cluster-health
```

### 安装docker(所有节点)

```
yum install https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm  -y
yum install https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-17.03.2.ce-1.el7.centos.x86_64.rpm  -y

#设置docker源
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://kn6xpwo2.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl restart docker

```


### 安装kubeadm

#### 配置源，下载kubeadm，docker和相关工具包

```
#设置Kubernetesyum源
#cat <<EOF > /etc/yum.repos.d/kubernetes.repo
#[kubernetes]
#name=Kubernetes
#baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/#kubernetes-el7-x86_64
#enabled=1
#gpgcheck=0
#EOF
#yum -y install epel-release
#yum clean all
#yum makecache

#kubeadm和相关工具包
#master
yum -y install kubelet kubeadm kubectl
#node
yum -y install kubelet kubeadm
```

#### 修改配置
```
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

#修改这一行
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
#添加这一行
Environment="KUBELET_EXTRA_ARGS=--v=2 --fail-swap-on=false --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/k8sth/pause-amd64:3.0"

systemctl daemon-reload
systemctl enable kubelet
```

### 初始化集群(master)

```
cat <<EOF > /etc/kubernetes/config.yaml 
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
etcd:
  endpoints:
  - https://10.45.180.73:2379
  - https://10.66.93.21:2379
  - https://10.80.240.214:2379
  caFile: /etc/etcd/ssl/ca.pem
  certFile: /etc/etcd/ssl/etcd.pem
  keyFile: /etc/etcd/ssl/etcd-key.pem
  dataDir: /var/lib/etcd
networking:
  podSubnet: 10.244.0.0/16
kubernetesVersion: 1.10.0
api:
  advertiseAddress: "10.45.180.73"
token: "b99a00.a144ef80536d4344"
tokenTTL: "0s"
apiServerCertSANs:
- master1
- master2
- master3
- 10.45.180.73
- 10.66.93.21
- 10.80.240.214
- 10.81.126.152
featureGates:
  CoreDNS: true
imageRepository: "registry.cn-hangzhou.aliyuncs.com/k8sth"
EOF
```

```
kubeadm init --config /etc/kubernetes/config.yaml  
```

#### 记录初始化结果(正常情况)

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.45.180.73:6443 --token b99a00.a144ef80536d4344 --discovery-token-ca-cert-hash sha256:51bac855420a83d635d97bc4ab819a1c9d524232c3e735888297c261dad69349
```

#### 初始化失败后处理办法
```
kubeadm reset
#或
rm -rf /etc/kubernetes/*.conf
rm -rf /etc/kubernetes/manifests/*.yaml
docker ps -a |awk '{print $1}' |xargs docker rm -f
systemctl  stop kubelet
```

#### 按照提示执行

```
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### kubeadm生成证书密码文件分发到master2和master3上面去

```
scp -r /etc/kubernetes/pki  master2:/etc/kubernetes/
scp -r /etc/kubernetes/pki  master3:/etc/kubernetes/
```

### 部署flannel网络，只需要在master1执行就行

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
#版本信息：quay.io/coreos/flannel:v0.10.0-amd64

kubectl create -f  /etc/kubernetes/kube-flannel.yml
```

### 部署kubernetes-dashboard

[kubernetes-dashboard](yaml/kubernetes-dashboard.yaml)

```
kubectl create -f /etc/kubernetes/kubernetes-dashboard.yaml
```

#### 获取token,通过令牌登陆

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
#### 通过ip访问
```
https://112.74.128.10:30000/
```

#### 记录令牌
```
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWt4NmpqIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJkODg3ZDNkNi01NTEyLTExZTgtYWUzMy0wMDE2M2UwNmEyOTQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.H21qS_xBTE5sbGXl5TlTFhg6GnDHP9qOh98FgcOZFeqvj9THT3mfyPDk6VOHLRONf7sxjEhx7Ag8jxuMQSYIkVkCTKGc6LkA8wJ_v26aaW_dU8EtaqUrdkrlURckWRfHo77MB5-fvfY4V-8u6qG6Kh70Q-Nj53FVaagTXI_x_vF3PLWYWuEjF8p5oA2tTOvK4297KfGaBhhG1w4rfOlq_tqALU-IcSSbNoeB9s5cDP_rsKt1Kdh5RA7T9XhVgr67dR90ZH_vAD3z_upfzkOubmWZwWJ9TDMe8wF5X1R8_svL1FrgzRIOOuE-BfH249Y1SRUmY78BhLAubkOwMUrTww
```

### 安装heapster，InfluxDB和Grafana

#### 推荐使用

[kube-monitor.yml.conf](yaml/kube-monitor.yml.conf)

```
kubectl create -f /etc/kubernetes/kube-monitor.yml.conf
```

#### 也可以试试这个，作者没装上
[grafana.yaml](yaml/kube-heapster/influxdb/grafana.yaml)
[heapster.yaml](yaml/kube-heapster/influxdb/heapster.yaml)
[influxdb.yaml](yaml/kube-heapster/influxdb/influxdb.yaml)
[heapster-rbac.yaml](yaml/kube-heapster/rbac/heapster-rbac.yaml)

```
kubectl create -f /etc/kubernetes/kube-heapster/influxdb/
kubectl create -f /etc/kubernetes/kube-heapster/rbac/
```

### 在master2和master3分别执行初始化
```
kubeadm init --config config.yaml
#初始化的结果和master1的结果完全一样
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 添加node1到集群
```
kubeadm join 10.45.180.73:6443 --token b99a00.a144ef80536d4344 --discovery-token-ca-cert-hash sha256:18cf256c72fe6e8918d961b6070e4dce2e59c4a0db7a9a492adbad463357dea9
```

### 让master也运行pod（默认master不运行pod）

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

>学习blog https://www.kubernetes.org.cn/3808.html

### 清除命令

```
kubectl delete -f /etc/kubernetes/kube-heapster/rbac/
kubectl delete -f /etc/kubernetes/kube-heapster/influxdb/
kubectl delete -f /etc/kubernetes/etc/kubernetes/kubernetes-dashboard.yaml
kubectl delete -f /etc/kubernetes/etc/kubernetes/kube-flannel.yml


```

