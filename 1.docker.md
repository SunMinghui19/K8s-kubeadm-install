													      kubeadm部署安装k8s

# 一、集群规划
```
mater
	主机名：k8s-master1
	IP:192.168.133.60
worker1
	主机名：k8s-node1
	IP:192.168.133.61
worker2
	主机名：k8s-node02
	IP:192.168.133.62

k8s版本：1.16
安装方式：kubeadm
操作系统：CentOS7
```
# 二、初始化服务器(所有节点都要做)
## 准备工作
```
更换镜像的源

```

## 2.1、配置主机名称
  ```
# hostnamectl set-hostname [主机名]
# hostname 查看主机名
  ```
## 2.2、配置名称解析
```
vim /etc/hosts 
添加
	192.168.133.60   k8s-master1
	192.168.133.61   k8s-node1
	192.168.133.62   k8s-node2
```
## 2.3、安装依赖包
```
# yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat libseccomp wgetvimnet-tools git	
```
## 2.4、设置防火墙为 Iptables 并设置空规则
```
关闭防火墙、禁用防火墙
# systemctl stop firewalld && systemctl disable firewalld

安装iptables服务，开启iptables服务，开机自启iptables，清空iptables规则，保存iptables的设置
# yum -y install iptables-services && systemctl start iptables &&  systemctl enable iptables && iptables -F && service iptables save
```
## 2.5、关闭交换分区和selinux
  ```
关闭交换分区
# swapoff -a && sed -i '/  swap  /  s/^\(.*\)$/#\1/g'  /etc/fstab

关闭selinux
# setenforce  0  &&  sed  -i  's/^SELINUX=.*/SELINUX=disabled/'  /etc/selinux/config
  ```
	
## 2.6、调整内核参数，对于 K8S
```
#cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

# mv kubernetes.conf  /etc/sysctl.d/kubernetes.conf
# sysctl -p /etc/sysctl.d/kubernetes.conf	

```
  
## 2.7、调整系统时区

```
# 设置系统时区为中国/上海
timedatectl set-timezone Asia/Shanghai
# 将当前的 UTC 时间写入硬件时钟
timedatectl set-local-rtc 0
# 重启依赖于系统时间的服务
systemctl restart rsyslog
systemctl restart crond
		
```
## 2.8、关闭系统不需要服务
```
# systemctl stop postfix && systemctl disable postfix
```
	
## 2.9、设置 rsyslogd 和 systemd journald
```
# mkdir /var/log/journal  # 持久化保存日志的目录
# mkdir /etc/systemd/journald.conf.d

cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent

# 压缩历史日志Compress=yes

SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000

# 最大占用空间 10G
SystemMaxUse=10G

# 单日志文件最大 200M
SystemMaxFileSize=200M

# 日志保存时间 2 周
MaxRetentionSec=2week

# 不将日志转发到 syslog
ForwardToSyslog=no
EOF

# systemctl restart systemd-journald
```

## 2.10、升级系统内核为 4.44
```
centos7.x系统自带的3.10.x内核存在一些BUG，导致运行的Docker、Kubernetes不稳定。

# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# 安装完成后检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置，如果没有，再安装一次！
# yum --enablerepo=elrepo-kernel install -y kernel-lt

# 设置开机从新内核启动
# grub2-set-default 'CentOS Linux (4.4.189-1.el7.elrepo.x86_64) 7 (Core)'

# 重启查看新内核
# reboot
# uname -r
```
## 2.10、kube-proxy开启ipvs的前置条件(all)
```
# modprobe br_netfilter
# cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

# chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules &&lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

# 三、安装docker软件(所有节点都要装)
```
# yum install -y yum-utils device-mapper-persistent-data lvm2

# yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# yum update -y && yum install -y docker-ce

##启动docker并设置为开机自启
# systemctl start docker && systemctl enable docker

## 创建 /etc/docker 目录
# mkdir /etc/docker

# 配置 daemon.json
# cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"  
}
}
EOF

# mkdir -p /etc/systemd/system/docker.service.d

# 重启docker服务
systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```


# 四、安装kubeadm(主从配置)(所有节点都配置)

## 4.1、添加kubernetes YUM软件源
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
## 4.2、安装kubeadm、kubectl、kubelet
```
# yum -y  install  kubeadm-1.15.1 kubectl-1.15.1 kubelet-1.15.1
# systemctl enable kubelet.service  #开机自启
```

# 五、导入镜像到docker(all)
```
1、把kubeadm-basic.images.tar.gz包导入到/root/目录下并解压
	tar -zxvf kubeadm-basic.images.tar.gz

2、编写脚本文件load-images.sh
# vim load-images.sh
写如如下内容
	#!/bin/bash
	ls /root/kubeadm-basic.images > /tmp/image-list.txt
	cd /root/kubeadm-basic.images
	for i in $( cat /tmp/image-list.txt )
	do
			docker load -i $i
	done
	rm -rf /tmp/image-list.txt
	
##赋予执行权限
# chmod a+x load-images.sh
3、运行脚本把/root/kubeadm-basic.images下的所有镜像导入到docker中去
# ./load-images.sh
4、将镜像和shell文件拷贝到其它节点
# scp -r kubeadm-basic.images  load-images.sh root@k8s-node2:/root/
5、在其它节点行执行shell脚本
```

# 六、初始化主节点(master)

## 6.1、显示init默认的初始化文件，并打印出来到kubeadm-config.yaml文件中
```
# kubeadm config print init-defaults > /root/kubeadm-config.yaml
```
## 6.2、修改kubeadm-config.yaml配置文件
```
# vim /root/kubeadm-config.yaml

	apiVersion: kubeadm.k8s.io/v1beta2
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
	  advertiseAddress: 192.168.88.10   #修改为当前服务器地址
	  bindPort: 6443
	nodeRegistration:
	  criSocket: /var/run/dockershim.sock
	  name: k8s-master01
	  taints:
	  - effect: NoSchedule
		key: node-role.kubernetes.io/master
	---
	apiServer:
	  timeoutForControlPlane: 4m0s
	apiVersion: kubeadm.k8s.io/v1beta2
	certificatesDir: /etc/kubernetes/pki
	clusterName: kubernetes
	controllerManager: {}
	dns:
	  type: CoreDNS
	etcd:
	  local:
		dataDir: /var/lib/etcd
	imageRepository: registry.aliyuncs.com/google_containers  #修改镜像仓库
	kind: ClusterConfiguration
	kubernetesVersion: v1.15.1  #修改为当前kubeadm版本号
	networking:
	  dnsDomain: cluster.local
	  podSubnet: "10.244.0.0/16"  #添加此行
	  serviceSubnet: 10.96.0.0/12
	scheduler: {} #添加下面字段，改默认调度模式为ipvs调度
	---
	apiVersion: kubeproxy.config.k8s.io/v1alpha1
	kind: KubeProxyConfiguration
	featureGates:
	  SupportIPVSProxyMode: true
	mode: ipvs
```
## 6.3、开始初始化：指定从哪个yaml文件进行初始化安装，自动颁发证书，并将所有信息写入到kubeadm-init.log	
```
kubeadm init --config=/root/kubeadm-config.yaml --experimental-upload-certs | tee /root/kubeadm-init.log
查看/root/kubeadm-init.log显示下面表明初始化成功
Your Kubernetes control-plane has initialized successfully!

##根据日志提示操作
# mkdir -p $HOME/.kube
#sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

## 6.4、查看节点
```
[root@k8s-master01 ~]# kubectl get node
NAME           STATUS     ROLES    AGE   VERSION
k8s-master01   NotReady   master   21m   v1.15.1
```

Tip：配置文件在/etc/kubernetes/下
```
[root@k8s-master1 kubernetes]# ls /etc/kubernetes/
admin.conf  controller-manager.conf  kubelet.conf  manifests  pki  scheduler.conf
```
# 七、部署网络（master）
## 7.1、整理目录
```
[root@k8s-master1 ~]# mkdir -p /root/install-k8s/
[root@k8s-master1 ~]# mv kubeadm-config.yaml kubeadm-init.log /root/install-k8s/
[root@k8s-master1 ~]# cd install-k8s/
[root@k8s-master1 install-k8s]# mkdir core
[root@k8s-master1 install-k8s]# mv kubeadm-config.yaml kubeadm-init.log core/
[root@k8s-master1 install-k8s]# mkdir plugin
[root@k8s-master1 install-k8s]# cd plugin/
[root@k8s-master1 plugin]# mkdir flannel
[root@k8s-master1 plugin]# cd flannel/
```
## 7.2、下载kube-flannel.yml
```
# vim /etc/hosts  
添加一条 199.232.68.133 raw.githubusercontent.com 
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
## 7.3、执行完后，/root目录下文件结构如下
```
--root/
--install-k8s/
	--core/
		--kubeadm-config.yaml  
		--kubeadm-init.log
	--plugin/
		--flannel/
			kube-flannel.yml
```
## 7.4、再执行yml文件
```
# kubectl apply -f kube-flannel.yml
```
## 7.5、重启kubelet并查看
```
[root@k8s-master1 flannel]# kubectl get pods -A
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
kube-system   coredns-5c98db65d4-s4rvh              1/1     Running   0          29m
kube-system   coredns-5c98db65d4-v555f              1/1     Running   0          29m
kube-system   etcd-k8s-master1                      1/1     Running   0          28m
kube-system   kube-apiserver-k8s-master1            1/1     Running   0          28m
kube-system   kube-controller-manager-k8s-master1   1/1     Running   0          28m
kube-system   kube-flannel-ds-amd64-l8725           1/1     Running   0          55s
kube-system   kube-proxy-pmjml                      1/1     Running   0          29m
kube-system   kube-scheduler-k8s-master1            1/1     Running   0          29m
```

# 八、node节点加入集群

## 8.1、查看master节点上的kubeadm-init.log
```
#查看对应秘钥
# cat /root/install-k8s/core/kubeadm-init.log
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.133.60:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:7ea56270ff231268914af8a5158349e369d2048a9ed93ddf69014f3d2ebca5af 
```
## 8.2、在node节点上执行即可
```
[root@k8s-node1 ~]# kubeadm join 192.168.133.60:6443 --token abcdef.0123456789abcdef \
>     --discovery-token-ca-cert-hash sha256:7ea56270ff231268914af8a5158349e369d2048a9ed93ddf69014f3d2ebca5af

[root@k8s-node2 ~]# kubeadm join 192.168.133.60:6443 --token abcdef.0123456789abcdef \
>     --discovery-token-ca-cert-hash sha256:7ea56270ff231268914af8a5158349e369d2048a9ed93ddf69014f3d2ebca5af
```
## 8.3、查看节点是否加入
```
[root@k8s-master01 core]# kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    master   95m   v1.15.1
k8s-node01     Ready    <none>   42s   v1.15.1
k8s-node02     Ready    <none>   42s   v1.15.1
```	






docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.3
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.3
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.3
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.3
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.3 k8s.gcr.io/kube-apiserver:v1.17.3
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.3 k8s.gcr.io/kube-controller-manager:v1.17.3
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.3 k8s.gcr.io/kube-scheduler:v1.17.3
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.3 k8s.gcr.io/kube-proxy:v1.17.3
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5 k8s.gcr.io/coredns:1.6.5



kubeadm init --kubernetes-version=v1.17.3 --pod-network-cidr 10.244.0.0/16


kubeadm join 192.168.133.30:6443 --token 61x67u.2ioif503ity5ure8 \
    --discovery-token-ca-cert-hash sha256:389623d9868e292912eff6c5eba6cea566aa26dbd6f78dfdc3e892e960865ca3


https://www.cnblogs.com/oscarli/p/12737409.html

# 问题
1、Failed to start Docker Application Container Engine（master上的docker启动不了）
原因是/etc/docker/daemon.json中出现了错误导致启动不起来，改正文件错误后systemctl daemon-reload就好了




	
