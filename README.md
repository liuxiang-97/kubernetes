# kubernetes 1.14 setup on centos7.5+
## 单master多节点
### 1.初始化系统
#### 1.1安装系统工具
echo "将最大文件打开数修改成100001, 关闭selinux"
echo "*                -       nofile    100001"  >> /etc/security/limits.conf
echo "*                -       nproc     100001"  >> /etc/security/limits.conf
sed -i "s/SELINUXTYPE=targeted/#SELINUXTYPE=targeted/" /etc/selinux/config
sed -i "s/SELINUX=enforcing/SELINUX=disabled/" /etc/selinux/config

echo "安装netstat,wget,curl,git,unzip等工具 "
yum install wget iptables-services telnet net-tools git curl unzip sysstat lsof ntpdate lrzsz vim  -y
#### 1.2安装ntp并同步时间
echo "安装ntp,并设置同步时间"
yum install ntp -y
systemctl start ntpd
systemctl enable ntpd
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp yes
timedatectl
#### 1.3关闭防火墙
echo "关闭firwalld，并备份iptables"
systemctl stop firewalld.service
systemctl disable firewalld.service
mv /etc/sysconfig/iptables  /etc/sysconfig/iptables.bak

systemctl disable iptables.service
echo "停止iptables"
systemctl stop iptables.service

echo "优化sshd登陆"
echo "UseDNS no" >> /etc/ssh/sshd_config
echo "UseDNS no" >> /etc/ssh/sshd_config
systemctl restart sshd
### 2. 启用IPVS和安装相关工具
#### 2.1启用IPVS
参考IPVS https://www.cnblogs.com/hongdada/p/9758939.html
在所有的Kubernetes节点执行以下脚本（若内核大于4.19替换nf_conntrack_ipv4为nf_conntrack）:
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
#### 2.2安装工具
#安装相关管理工具
yum install ipset ipvsadm -y
#查看超时设置
ipvsadm -l --timeout 
#查看链接
ipvsadm -ln
1. 查看lvs的连接状态命令： ipvsadm  -l  --stats

说明：

1. Conns    (connections scheduled)  已经转发过的连接数
2. InPkts   (incoming packets)       入包个数
3. OutPkts  (outgoing packets)       出包个数
4. InBytes  (incoming bytes)         入流量（字节）  
5. OutBytes (outgoing bytes)         出流量（字节）

2. 查看lvs速率  ：ipvsadm   -l  --rate

说明：

1. CPS      (current connection rate)   每秒连接数
2. InPPS    (current in packet rate)    每秒的入包个数
3. OutPPS   (current out packet rate)   每秒的出包个数
4. InBPS    (current in byte rate)      每秒入流量（字节）
5. OutBPS   (current out byte rate)      每秒入流量（字节）
### 3. 系统内核调整
cat <<EOF > /etc/sysctl.d/k8s.conf
# https://github.com/moby/moby/issues/31208 
# ipvsadm -l --timout
# 修复ipvs模式下长连接timeout问题 小于900即可
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10

#调整系统系统令牌数量 
net.ipv4.tcp_max_tw_buckets = 100000
#调整k8s要求的相关参数 

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF

#重载内核参数
sysctl --system
### 4. 安装docker
cd /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#安装docker存储相关工具
yum install -y yum-utils device-mapper-persistent-data lvm2

#指定版本k8s要求是18.06
#yum list docker-ce --showduplicates -y
yum install docker-ce*18.06*  -y

# 配置 daemon.阿里云镜像加速和systemd进程管理
#参考: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
mkdir -pv /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": ["https://gfexkhvj.mirror.aliyuncs.com"]
}
EOF

#设置开机启动并启动docker
systemctl daemon-reload
systemctl restart docker
systemctl enable docker && systemctl restart docker
docker ps -a
### 5. 安装kubeadmin
cd /etc/yum.repos.d/
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#yum search kubelet --showduplicates -y
#安装指定的1.14.3版本
yum install -y kubeadm-1.14.3 kubelet-1.14.3 kubectl-1.14.3
systemctl enable kubelet
### 6. 安装haproxy(做tcp代理，公有云可用SLB替代)
yum install haproxy -y 
mv /etc/haproxy/haproxy.cfg  /etc/haproxy/haproxy.cfg.bak

echo """
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    option                  redispatch
    retries                 3
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout check           10s
    maxconn                 3000

listen stats
    mode   http
    bind :10086
    stats   enable
    stats   uri     /admin?stats
    stats   auth    admin:admin
    stats   admin   if TRUE
    
frontend  k8s_https *:8443
    mode      tcp
    maxconn      2000
    default_backend     https_sri
    
backend https_sri
    balance      roundrobin
    server master1-api 10.0.0.4:6443  check inter 10000 fall 2 rise 2 weight 1
    server master2-api 10.0.0.5:6443  check inter 10000 fall 2 rise 2 weight 1
    server master3-api 10.0.0.6:6443  check inter 10000 fall 2 rise 2 weight 1
""" > /etc/haproxy/haproxy.cfg

#启动haproxy 监控8443端口 代理到6443端口
#10.0.0.4 10.0.0.5 10.0.0.6 改成自已集群IP
systemctl enable haproxy && systemctl start haproxy 
### 7.初始化k8s集群
#谷歌镜像pull不下来需更换为国内阿里云的镜像仓库
#生成yaml配置
kubeadm config print init-defaults > kubeadmin-config.yaml

#### 7.1配置yaml文件
apiVersion: kubeadm.k8s.io/v1beta1
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.0.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: uat-csp-k8s-master1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "10.0.0.4:8443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.14.3
networking:
  dnsDomain: cluster.local
  podSubnet: "10.244.0.0/16"  
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs

#advertiseAddress:改成执行kubeadmin的IP
#controlPlaneEndpoint:改成haproxy的地址
#imageRepository：改成阿里云的镜像仓库
#kubernetesVersion:k8s版本
#podSubnet:改为pod子网网段
#### 7.2 初始化k8s
配置hosts
echo """
10.0.0.4 master1
10.0.0.5 node1
10.0.0.6 node2
""" >> /etc/hosts
#1.13版本无此参数--experimental-upload-certs
#执行日志存一份在kubeadmin-init.log
kubeadm init --config=kubeadmin-config.yaml --experimental-upload-certs | tee kubeadmin-init.log

#上面执行成功后
cat << EOF >> ~/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF
source ~/.bashrc

#查看状态，如果coreDNS没有running，是因为网络组件没有安装
kubectl get cs
kubectl get nodes
kubectl -n kube-system get pods
根据日志输出提示在master节点或node节点执行加入集群的命令


