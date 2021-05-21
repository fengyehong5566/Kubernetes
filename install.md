# 二进制部署k8s集群

# 1 简介
  本文档主要用于讲解使用二进制的方式部署k8s集群


# 2 环境规划

## 2.1 主机规划
master主机部署：apiserver，controllerManager，kube-scheduler， etcd
worker主机部署：kubelet，kube-proxy

|ip|hostname|  
|-|-|
|10.10.10.224| k8s-master|  
|10.10.10.225| k8s-node01|  
|10.10.10.226| k8s-node02|  

## 2.2 软件版本
|软件|版本|
|-|-|
|kube-*|v1.20|
|etcd| v3.4.13|
|calico| v3.14|
|coredns|1.7.0|

# 3 搭建集群

## 3.1 服务器基本配置
>a、修改主机名   

>b、关闭防火墙和selinux
``` 
 > systemctl stop firewalld
 > setenforece 0       //临时关闭selinux  
 > sed -i 's/^SELINUX=.\*/SELINUX=disabled/' /etc/selinux/config   //永久关闭，但需重启服务器  
```
>c、关闭交换分区  
```
 > swapoff -a   //临时关闭  
 注： 修改 /etc/fstab 文件 注释掉swap一行     // 永久关闭  
```

>d、时间同步
```
 > yum install -y chrony  
 > systemctl start chronyd  
 > systemctl enable chronyd  
 > chronyc sources  
```

>e、修改内核参数
```
> cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

> sysctl --system
```
>f、加载ipvs模块
```
> modprobe -- ip_vs
> modprobe -- ip_vs_rr
> modprobe -- ip_vs_wrr
> modprobe -- ip_vs_sh
> modprobe -- nf_conntrack_ipv4     //nf_conntrack_ipv4有可能是其他名字，因系统而定
> lsmod | grep ip_vs
> lsmod | grep nf_conntrack_ipv4
> yum install -y ipvsadm
```

## 3.2 配置工作目录
>每个组件都需要有配置文件、ssl证书文件以及启动文件  
所有服务的bin文件： /usr/local/bin/  
配置文件： /home/kubernetes/cfg/\*.conf  
证书文件： /home/kubernetes/ssl/\*.pem  
日志文件： /home/kubernetes/logs/\<componentName\>/*  
etcd服务： /home/etcd/{conf,logs,data,ssl}  
证书制作目录： /home/makessl
安装包目录： /home/k8s/packages


## 3.3 配置ca证书
```
//下载cfssl证书工具
> mkdir  /home/makessl
> cd /home/makessl

> wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
> wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
> wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
> chmod +x cfssl*
> mv cfssl_linux-amd64  /usr/local/bin/cfssl
> mv cfssljson_linux-amd64  /usr/local/bin/cfssljson
> mv cfssl-certinfo_linux-amd64   /usr/local/bin/cfssl-certinfo

//配置ca请求文件
cat > ca-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "system"
    }
  ],
  "ca": {
          "expiry": "87600h"
  }
}
EOF

> cfssl gencert -initca ca-csr.json  | cfssljson -bare ca
// 将会产生 ca-key.pem(私钥)  ca.pem (公钥)
> ls 
ca.csr  ca-csr.json  ca-key.pem  ca.pem



//配置ca证书策略 [10年有效期]
> cat > ca-config.json << EOF 
{
  "signing": {
      "default": {
          "expiry": "87600h"
        },
      "profiles": {
          "kubernetes": {
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ],
              "expiry": "87600h"
          }
      }
  }
}
EOF

```
>**注**：  
ca-csr.sjon文件：  
　　CN：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；  
　　O：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)
ca-config.json文件：
　　



## 3.4 搭建etcd集群
```
mkdir -p /home/etcd/{conf,logs,data,ssl}

//配置etcd的csr请求文件
cat > etcd-csr.json  << EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.10.10.224",
    "10.10.10.225",
    "10.10.10.226"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "Beijing",
    "L": "Beijing",
    "O": "k8s",
    "OU": "system"
  }]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd.csr.josn | cfssljson -bare etcd
ls etcd*.pem

//安装etcd服务
cd /home/k8s/packages/
wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz
tar etcd-v3.4.13-linux-amd64.tar.gz  
cp -p etcd-v3.4.13-linux-amd64/etcd*  /usr/local/bin

//etcd配置文件
cat > /home/etcd/conf/etcd.conf  << EOF
#[Member]
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/home/etcd/data"
ETCD_LISTEN_PEER_URLS="https://10.10.10.224:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.10.10.224:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.10.10.224:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.10.10.224:2379"
ETCD_INITIAL_CLUSTER="etcd1=https://10.10.10.224:2380,etcd2=https://10.10.10.225:2380,etcd3=https://10.10.10.226:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF


//etcd 服务启动文件
cat >  /usr/lib/systemd/system/etcd.service  << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=-/home/etcd/conf/etcd.conf
WorkingDirectory=/home/etcd/
ExecStart=/usr/local/bin/etcd \
  --cert-file=/home/etcd/ssl/etcd.pem \
  --key-file=/home/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/home/etcd/ssl/ca.pem \
  --peer-cert-file=/home/etcd/ssl/etcd.pem \
  --peer-key-file=/home/etcd/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/home/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF

cd /home/makessl
cp etcd*.pem  /home/etcd/ssl
cp ca*.pem  /home/etcd/ssl

//启动etcd
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd

//查看集群状态

ETCDCTL_API=3  ${ETCD_ROOT}/bin/etcdctl  --write-out=table  \
	--cacert=${ETCD_ROOT}/ssl/ca.pem     \
	--cert=${ETCD_ROOT}/ssl/etcd.pem     \
	--key=${ETCD_ROOT}/ssl/etcd-key.pem  \
	--endpoints=https://10.10.10.224:2379,https://10.10.10.225:2379,https://10.10.10.226:2379   \
	 endpoint health
```
另外两台机器部署，只需把 /home/etcd/ 目录和 etcd.service 复制过去，修改配置文件即可

>**注**：  
ETCD_NAME：节点名称，集群中唯一  
ETCD_DATA_DIR：数据目录  
ETCD_LISTEN_PEER_URLS：集群通信监听地址  
ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址  
ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址  
ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址  
ETCD_INITIAL_CLUSTER：集群节点地址  
ETCD_INITIAL_CLUSTER_TOKEN：集群Token  
ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群  


## 3.5 kubernetes master组件安装  
### 3.5.1 下载安装包
```
> pwd
/home/k8s/packages
> wget https://dl.k8s.io/v1.20.1/kubernetes-server-linux-amd64.tar.gz
> tar -xf kubernetes-server-linux-amd64.tar 
> cd kubernetes/server/bin/
> cp kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/

```

### 3.5.2 install api-server
```
创建csr请求文件
> pwd
/home/makessl
> cat >  kube-apiserver-csr.json  << EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.10.10.224",
    "10.10.10.225",
    "10.10.10.226",
    "10.10.10.227",
    "10.10.10.228",
    "10.10.10.223",
    "10.255.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "system"
    }
  ]
}


```
> 注：  
> 如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表。
由于该证书后续被 kubernetes master 集群使用，需要将master节点的IP都填上，同时还需要填写 service 网络的首个IP。(一般是 kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.254.0.1)  
>hosts 中的内容可以为空，即使按照上面的配置，向集群中增加新节点后也不需要重新生成证书。


```
生成证书和token文件
> cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver

> cat > token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

```

```
创建配置文件：
> cat  > kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=172.10.1.11 \
  --secure-port=6443 \
  --advertise-address=172.10.1.11 \
  --insecure-port=0 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --service-cluster-ip-range=10.255.0.0/16 \
  --token-auth-file=/home/kubernetes/apiserver/conf/token.csv \
  --service-node-port-range=30000-50000 \
  --tls-cert-file=/home/kubernetes/apiserver/ssl/kube-apiserver.pem  \
  --tls-private-key-file=/home/kubernetes/apiserver/ssl/kube-apiserver-key.pem \
  --client-ca-file=/home/kubernetes/apiserver/ssl/ca.pem \
  --kubelet-client-certificate=/home/kubernetes/apiserver/ssl/kube-apiserver.pem \
  --kubelet-client-key=/home/kubernetes/apiserver/ssl/kube-apiserver-key.pem \
  --service-account-key-file=/home/kubernetes/apiserver/ssl/ca-key.pem \
  --service-account-signing-key-file=/home/kubernetes/apiserver/ssl/ca-key.pem  \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --etcd-cafile=/home/etcd/ssl/ca.pem \
  --etcd-certfile=/home/etcd/ssl/etcd.pem \
  --etcd-keyfile=/home/etcd/ssl/etcd-key.pem \
  --etcd-servers=https://172.10.1.11:2379,https://172.10.1.12:2379,https://172.10.1.13:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/home/kubernetes/apiserver/logs/kube-apiserver-audit.log \
  --event-ttl=1h \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/home/kubernetes/apiserver/logs \
  --v=4"
EOF
```
> 注：  
>```
> --service-account-signing-key-fil        # 1.20以上版本必须有此参数  
>   --service-account-issuer    # 1.20以上版本必须有此参数  
> --logtostderr：启用日志  
> --v：日志等级  
> --log-dir：日志目录  
> --etcd-servers：etcd集群地址  
> --bind-address：监听地址
> --secure-port：https安全端口
> --advertise-address：集群通告地址
> --allow-privileged：启用授权
> --service-cluster-ip-range：Service虚拟IP地址段
> --enable-admission-plugins：准入控制模
> --authorization-mode：认证授权，启用RBAC授权和节点自管理
> --enable-bootstrap-token-auth：启用TLS bootstrap机制
> --token-auth-file：bootstrap token文件
> --service-node-port-range：Service nodeport类型默认分配端口范围
> --kubelet-client-xxx：apiserver访问kubelet客户端证书
> --tls-xxx-file：apiserver https证书
> --etcd-xxxfile：连接Etcd集群证书
> --audit-log-xxx：审计日志
> ```
 

```
创建apiserver服务启动文件
> cat > /usr/lib/systemd/system/kube-apiserver.service  << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service

[Service]
EnvironmentFile=-/home/kubernetes/apiserver/conf/kube-apiserver.conf
ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

部署:
> cp ca*.pem  /home/kubernetes/apiserver/ssl
> cp kube-apiserver*.pem   /home/kubernetes/apiserver/ssl
> cp token.csv  /home/kubernetes/apiserver/ssl
> cp kube-apiserver.conf  /home/kubernetes/apiserver/ssl

启动服务：
> systemctl daemon-reload
> systemctl enable kube-apiserver
> systemctl start  kube-apiserver
> systemctl status kube-apiserver

测试：
> curl  --insecure https://10.10.10.224:6443/

有返回说明启动正常

```

### 3.5.3  install  kubectl
```
创建csr请求文件
> cat > admin-csr.json  << EOF 
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "system:masters",             
      "OU": "system"
    }
  ]
}
EOF


```
>注：
> 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限；
O指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；
注：
这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group；
"O": "system:masters", 必须是system:masters，否则后面kubectl create clusterrolebinding报错。

```
生成证书：
> cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
> cp admin*.pem /etc/kubernetes/ssl/

创建kubeconfig配置文件：
    kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书

设置集群参数
> kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://10.10.10.224:6443 --kubeconfig=kube.config

设置客户端认证参数
> kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config

设置上下文参数
> kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
设置默认上下文
> kubectl config use-context kubernetes --kubeconfig=kube.config
> mkdir ~/.kube
> cp kube.config ~/.kube/config

授权kubernetes证书访问kubelet api权限
> kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes

====================
完成上述步骤后，kubectl就可以与kube-apiserver服务通信了
> kubectl cluster-info
> kubectl get componentstatuses
> kubectl get  all --all-namespaces

====
配置kubectl 子命令补全
> yum install  -y  bash-completion
> source /usr/share/bash-completion/bash_completion
> source <(kubectl completion bash)
> kubectl completion bash  > ~/.kube/completion.bash.inc
> source ~/.kube/completion.bash.inc
> source $HOME/.bash_profile
```

### 3.5.4 install kube-controller-manager
```
创建csr请求文件
> cat  kube-controller-manager  << EOF
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "10.10.10.224",
      "10.10.10.225",
      "10.10.10.226"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "Hubei",
        "L": "Wuhan",
        "O": "system:kube-controller-manager",
        "OU": "system"
      }
    ]
}
EOF
```
>注：
hosts 列表包含所有 kube-controller-manager 节点 IP；
CN 为 system:kube-controller-manager、O 为 system:kube-controller-manager，kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限

```
> cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
> ls kube-controller-manager*.pem

========
创建kube-controller-manager的kubeconfig

设置集群参数
> kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://10.10.10.224:6443 --kubeconfig=kube-controller-manager.kubeconfig

设置客户端认证参数
> kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig

设置上下文参数
> kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

设置默认上下文
> kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

===========
创建配置文件
> cat > kube-controller-manager.conf  << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--port=0 \
  --secure-port=10252 \
  --bind-address=127.0.0.1 \
  --kubeconfig=/home/kubernetes/controller-manager/cfg/kube-controller-manager.kubeconfig \
  --service-cluster-ip-range=10.255.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/home/kubernetes/controller-manager/ssl/ca.pem \
  --cluster-signing-key-file=/home/kubernetes/controller-manager/ssl/ca-key.pem \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.0.0.0/16 \
  --experimental-cluster-signing-duration=87600h \
  --root-ca-file=/home/kubernetes/controller-manager/ssl/ca.pem \
  --service-account-private-key-file=/home/kubernetes/controller-manager/ssl/ca-key.pem \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/home/kubernetes/controller-manager/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/home/kubernetes/controller-manager/ssl/kube-controller-manager-key.pem \
  --use-service-account-credentials=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/home/kubernetes/controller-manager/logs/ \
  --v=2"
EOF

============
创建启动文件
> cat > /usr/lib/systemd/system/kube-controller-manager.service  << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/home/kubernetes/controller-manager/cfg/kube-controller-manager.conf
ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

=========
> cp kube-controller-manager*.pem  /home/kubernetes/controller-manager/ssl
> cp kube-controller-manager.kubeconfig  /home/kubernetes/controller-manager/ssl


=====
启动服务
> systemctl daemon-reload 
> systemctl enable kube-controller-manager
> systemctl start kube-controller-manager
> systemctl status kube-controller-manager

```

### 3.5.5 install kube-scheduler
```
创建csr请求文件
> cat > kube-scheduler-csr.json  <<EOF
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "10.10.10.224",
      "10.10.10.225",
      "10.10.10.226"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "Beijing",
        "L": "Beijing",
        "O": "system:kube-scheduler",
        "OU": "system"
      }
    ]
}
EOF
```
>注：  
　　hosts 列表包含所有 kube-scheduler 节点 IP；  
　　CN 为 system:kube-scheduler  
　　O 为 system:kube-scheduler  
　　kubernetes 内置的 ClusterRoleBindings system:kube-scheduler 将赋予 kube-scheduler 工作所需的权限。

```
生成证书
> cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler

=========
创建kube-scheduler的kubeconfig
1、设置集群参数
> kubectl config set-cluster kubernetes -certificate-authority=ca.pem --embed-certs=ture --server=https://10.10.10.224:6443 --kubeconfig=kube-scheduler.kubeconfig

2、设置客户端认证参数
> kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --ambed-certs=true --kubeconfig=kube-scheduler.kubeconfig

3、设置上下文参数
> kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler  --kubeconfig=kube-scheduler.kubeconfig

4、设置默认上下文
> kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig

=======
创建配置文件
> cat  > kube-scheduler.conf  << EOF
KUBE_SCHEDULER_OPTS="--address=127.0.0.1 \
--kubeconfig=/home/kubernetes/kube-scheduler/cfg/kube-scheduler.kubeconfig \
--leader-elect=true \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/home/kubernetes/kube-scheduler/log/ \
--v=2"
EOF

===============
创建启动文件
> cat /usr/lib/systemd/system/kube-scheduler.service  << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/home/kubernetes/kube-scheduler/cfg/kube-scheduler.conf
ExecStart=/usr/local/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
========
复制文件到部署目录
> cp  kube-scheduler*.pem  /home/kubenetes/kube-scheduler/ssl
> cp kube-scheduler.kubeconf  /home/kubenetes/kube-scheduler/cfg/
> cp kube-scheduler.conf  /home/kubenetes/kube-scheduler/cfg/

======
启动服务
> systemctl  daemon-reload
> systemctl enable kube-scheduler
> systemctl start kube-scheduler
> systemctl status kube-scheduler

```

# 3.6 Worker节点组件安装
制作kubelet相关文件【在master上制作，完成后发送到worker节点即可】
## 3.6.1 install docker
```
> wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
> yum install -y  docker-ce
> systemctl enable docker
> systemctl start docker
> docker version

========
修改docker源和 驱动
> cat  > /etc/docker/daemon.json  << EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
        "https://1nj0zren.mirror.aliyuncs.com",
        "https://kfwkfulq.mirror.aliyuncs.com",
        "https://2lqq34jg.mirror.aliyuncs.com",
        "https://pee6w651.mirror.aliyuncs.com",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ]
}
EOF

> systemctl restart docker
> docker info  | grep -i  "cgroup driver"

========
下载依赖镜像
> docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
> docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
修改tag
> docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
> docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
删除就tag
> docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
>  docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
```

## 3.6.2 install kubelet

```
创建kubelet-bootstrap.kubeconfig
> BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' ) token.csv
设置集群参数
> kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://10.10.10.224:6443 --kuberconfig=kubelet-bootstrap.kubeconfig

设置客户端认证参数
> kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRP_TOKEN} --kubeconfig=kubelet-bootstrap.kubeconfig

设置上下文参数
> kubectl config set-context default --cluster=kubernetes -user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.kubeconfig

设置默认上下文
> kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig

创建角色绑定
> kubectl create clusterrolebinding kubelet-bootstrap --clusterroole=system:node-bootstrapper --user=kubelet-bootstrap


=========
创建配置文件
> cat > kubelet.json  << EOF
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/home/kubernetes/kubelet/ssl/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "10.10.10.225",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "cgroupfs",        # 如果docker的驱动为systemd，处修改为systemd。此处设置很重要，否则后面node节点无法加入到集群
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "featureGates": {
    "RotateKubeletClientCertificate": true,
    "RotateKubeletServerCertificate": true
  },
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.255.0.2"]
}

=======
创建启动文件
> cat > /usr/lib/systemd/system/kubelet.service  << EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --bootstrap-kubeconfig=/home/kubernetes/kubelet/cfg/kubelet-bootstrap.kubeconfig \
  --cert-dir=/home/kubernetes/kubelet/ssl \
  --kubeconfig=/home/kubernetes/kubelet/cfg/kubelet.kubeconfig \
  --config=/home/kubernetes/kubelet/cfgkubelet.json \
  --network-plugin=cni \
  --pod-infra-container-image=k8s.gcr.io/pause:3.2 \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/home/kubernetes/kubelet/log/ \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```
>注：  
　　–hostname-override：显示名称，集群中唯一  
　　–network-plugin：启用CNI  
　　–kubeconfig：空路径，会自动生成，后面用于连接apiserver  
　　–bootstrap-kubeconfig：首次启动向apiserver申请证书  
　　–config：配置参数文件  
　　–cert-dir：kubelet证书生成目录  
　　–pod-infra-container-image：管理Pod网络容器的镜像  


```
> cp kubelet*.pem  /home/kubernetes/kubelet/ssl
> cp kubelet.json  /home/kubernetes/kubelet/cfg/
> cp kubelet-bootstrap.kubeconfig  /home/kubernetes/kubelet/cfg/

```
同步/home/kubernetes/kubelet目录到各个worker节点，然后执行一下命令
```
> systemctl daemon-reload
> systemctl enable kubelet
> systemctl start kubelet
> systemctl status kubelet
```
确认kubelet服务启动成功后，接着到master上Approve一下bootstrap请求。执行如下命令可以看到三个worker节点分别发送了三个 CSR 请求：
```
> kubectl  get csr
根据结果中NAME列得到“node-csr-XXX”，且CONDITION列为“Pending”。

执行一下命令
> kubectl certificate approve  node-csr-XXX
...
> kubectl get csr
根据结果可发行 CONDITION列已经由“Pending” 变成 “Approved，Isussed”

> kubectl get  nodes
可以发行已经可以获取到node信息，且node的 STATUS 列为 NotReady，因为还未部署网络组件(如calico、flannel)
```

## 3.6.3
```
创建CSR文件
> cat kube-proxy-csr.json  << EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "system"
    }
  ]
}

生成证书
> cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

=======
创建kubeconfig文件
> kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://172.10.0.20:6443 --kubeconfig=kube-proxy.kubeconfig

> kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --client-key=kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig

> kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig

> kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig


=============
创建kube-proxy配置文件
>  cat  > kube-proxy  << EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 172.10.1.14
clientConnection:
  kubeconfig: /home/kubernetes/kube-proxy/cfg/kube-proxy.kubeconfig
clusterCIDR: 192.168.0.0/16
healthzBindAddress: 172.10.1.14:10256
kind: KubeProxyConfiguration
metricsBindAddress: 172.10.1.14:10249
mode: "ipvs"

EOF

```
> 注：
> clusterCIDR:  此处网段必须与网络组件网段保持一致，否则部署网络组件时会报错

```
创建启动文件
> cat > /usr/lib/systemd/system/kube-proxy.service  << EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --config=/home/kubernetes/kube-proxy/cfg/kube-proxy.yaml \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

============
负责文件到部署目录
> cp kube-proxy*.pem /home/kubernetes/kube-proxy/ssl/
> cp kube-proxy.kubeconfig  /home/kubernetes/kube-proxy/cfg
> cp kube-proxy.yml  /home/kubernetes/kube-proxy/cfg/

```

同步/home/kubernetes/kubelet目录到各个worker节点，然后执行一下命令

```
启动服务
> systemctl daemon-reload
> systemctl enable kube-proxy
> systemctl restart kube-proxy
> systemctl status kube-proxy
```

## 3.6.4 配置网络组件
```
> wget https://docs.projectcalico.org/v3.14/manifests/calico.yaml
> kubectl apply -f calico.yaml 

查看各个节点的STATUS及pods信息
> kubectl get  nodes
> kubectl get pods  -A

```
## 3.6.5 部署 coredns
```
访问 https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed 复制里边的内容，写入 coredns.yaml wenj 

修改yaml文件内容
kubernetes cluster.local in-addr.arpa ip6.arpa
forward . /etc/resolv.conf
clusterIP为：10.255.0.2（kubelet配置文件中的clusterDNS）


==============
> cat coredns.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local  in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. Default is 1.
  # 2. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
         podAntiAffinity:
           preferredDuringSchedulingIgnoredDuringExecution:
           - weight: 100
             podAffinityTerm:
               labelSelector:
                 matchExpressions:
                   - key: k8s-app
                     operator: In
                     values: ["kube-dns"]
               topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: coredns/coredns:1.8.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.255.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP

=====
> kubectl apply  -f coredns.yaml

```

附：  
文本参考
https://blog.51cto.com/13053917/2596613#h26

