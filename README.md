# ceph
### 准备环境
```shell
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
hostnamectl set-hostname ceph-admin
hostnamectl set-hostname ceph-node01
hostnamectl set-hostname ceph-node01

cat /etc/hosts
192.168.12.242 ceph-admin
192.168.12.173 ceph-node01
192.168.12.146 ceph-node02

ssh-keygen
ssh-copy-id -i .ssh/id_rsa ceph-node01
ssh-copy-id -i .ssh/id_rsa ceph-node02

vi /etc/yum.repos.d/ceph.repo
[ceph-nautilus] 
name=ceph-nautilus
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/x86_64/
enabled=1
gpgcheck=0

[ceph-nautilus-noarch]
name=ceph-nautilus-noarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch/
enabled=1
gpgcheck=0 
yum  clean all
yum makecache
```

### NTP安装
```shell
#安装
yum install ntp
#执行 最好在局域网内，建立自己的时间同步源。其中ntpdate 是客户端命令， 连接时间同步服务器，修改自己的时间。 一定要同步时间，
ntpdate -s time-b.nist.gov  

#修改 /etc/ntp.conf文件
#注释下面4行
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst

#服务器节点修改如下
server 127.127.1.0
fudge  127.127.1.0 stratum 10

#验证下
ntpq –p –-查看ceph-admin的ntp服务器是否为自身
remote           refid      st t when poll reach   delay  offset  jitter
==========================================================================
*LOCAL(0)        .LOCL.          10 l  28   64  377   0.000    0.000   0.000
这样的输出证明已经配置成功了。

#配置其他节点
#其他服务器都连接admin节点，其中段: server 210.72.145.44   iburst
Server 192.168.12.242 -----------–为ceph-admin的IP
fudge  127.127.1.0 stratum 11

#重启后检查
systemctl restart ntpd 
systemctl enable ntpd 
#查看是否为ceph-admin
ntpq –p  
#结果如下
remote           refid      st t when poll reach   delay  offset  jitter
===========================================================================
*ceph-admin      LOCAL(0)        11 u  13   64    1   1.236   -0.700   0.000
Remote ：*ceph-admin
Refid  ：LOCAL(0)
如果是其他的，如：refid：init就为配置不成功，按此步骤在其他节点进行配置。

```

### 部署ceph和ceph-deploy
```shell
#每个主机安装 ceph
yum install ceph -y

#Admin节点安装
yum install ceph-deploy -y
```

### 部署监控节点（mon）
```shell
ceph-deploy new ceph-admin ceph-node01 ceph-node02
vim  /root/.cephdeploy.conf  #修改这个文件，添加：  overwrite_conf = true
ceph-deploy  --overwrite-conf  mon create-initial
systemctl status ceph-mon@ceph-node01 #重启命令
```

### 部署mgr
```shell
ceph-deploy mgr create ceph-admin
systemctl status ceph-mgr@ceph-admin

[root@ceph-admin ~]# ps -ef | grep ceph
root       44009       1  0 15:04 ?        00:00:00 /usr/bin/python2.7 /usr/bin/ceph-crash
ceph       55206       1  0 15:25 ?        00:00:00 /usr/bin/ceph-mon -f --cluster ceph --id ceph-admin --setuser ceph --setgroup ceph
ceph       55568       1  2 15:35 ?        00:00:01 /usr/bin/ceph-mgr -f --cluster ceph --id ceph-admin --setuser ceph --setgroup ceph
root       55692    1476  0 15:36 pts/0    00:00:00 grep --color=auto ceph

[root@ceph-node01 ~]# ps -ef | grep ceph
root       44650       1  0 15:09 ?        00:00:00 /usr/bin/python2.7 /usr/bin/ceph-crash
ceph       45026       1  0 15:25 ?        00:00:00 /usr/bin/ceph-mon -f --cluster ceph --id ceph-node01 --setuser ceph --setgroup ceph
root       45119    1449  0 15:36 pts/0    00:00:00 grep --color=auto ceph

[root@ceph-node02 ~]# ps -ef | grep ceph
root       44134       1  0 15:09 ?        00:00:00 /usr/bin/python2.7 /usr/bin/ceph-crash
ceph       54684       1  0 15:25 ?        00:00:00 /usr/bin/ceph-mon -f --cluster ceph --id ceph-node02 --setuser ceph --setgroup ceph
root       54775    1449  0 15:36 pts/0    00:00:00 grep --color=auto ceph
```

### 部署OSD
```shell
先要有挂载磁盘，用系统盘也可以，但是并不安全，这里有两个方案
　　1.找几台新机器，OSD挂载的目录反正随便定，新机器上的数据都无所谓
　　2.额外挂载磁盘，可以对挂载磁盘做虚拟化，LVM，可以部署更多OSD
可以执行lsblk命令查看，如果磁盘已经被挂载，则会报错哦

ceph-deploy --overwrite-conf osd create --data /dev/sdb $HOSTNAME
```


### 部署dashboard
```shell
yum install -y ceph-mgr-dashboard
ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert  
ceph dashboard set-login-credentials  admin 111111
https://192.168.12.242:8443
```



### 问题01 ImportError: No module named pkg_resources
```shell
yum -y install python2-pip
```

### 问题02 admin_socket: exception getting command descriptions: [Errno 2] No such file or directory
```shell
iptables -F
getenforce
setenforce 0
# 删除之前版本ceph残留的文件
rm -rf /etc/ceph/*
rm -rf /var/lib/ceph/*/*
rm -rf /var/log/ceph/*
rm -rf /var/run/ceph/*
```

### 问题03 [errno 2] error connecting to the cluster
```shell
find / -name ceph.client.admin.keyring
/root/ceph.client.admin.keyring
cp -a  /root/ceph.client.admin.keyring  /etc/ceph/
chmod +r ceph.client.admin.keyring
ceph -s
ceph health
```

### 问题04 CEPH集群无法初始化OSD问题
```shell
# 安装ceph的osd时.运行清空磁盘命令
ceph-deploy disk zap ceph-node01 /dev/sdb
[node3-ceph][WARNIN]  stderr: wipefs: error: /dev/sdb: probing initialization failed: Device or resource busy
[node3-ceph][WARNIN] --> failed to wipefs device, will try again to workaround probable race condition

ceph-deploy osd create --data /dev/sdb ceph-admin
[ceph-admin][ERROR ] RuntimeError: command returned non-zero exit status: 1
[ceph_deploy.osd][ERROR ] Failed to execute command: /usr/sbin/ceph-volume --cluster ceph lvm create --bluestore --data /dev/sdb
[ceph_deploy][ERROR ] GenericError: Failed to create 1 OSDs


# 遇到这种报错时，只能上这台机器，手动进行dd命令清空磁盘并重启
dd if=/dev/zero of=/dev/sdb bs=512K count=1
reboot
```


### 问题05 ceph:health_warn clock skew detected on mon的解决办法
```shell
# 造成集群状态health_warn：clock skew detected on mon节点的原因有两个，一个是mon节点上ntp服务器未启动，另一个是ceph设置的mon的时间偏差阈值比较小。
1、确认ntp服务是否正常工作
systemctl status ntpd

2、修改ceph配置中的时间偏差阈值
vim ~/ceph.conf
在global字段下添加：
mon clock drift allowed = 2
mon clock drift warn backoff = 30

# 向需要同步的mon节点推送配置文件，命令如下：
ceph-deploy --overwrite-conf config push node{1,2,3}
systemctl restart ceph-mon.target

[root@ceph-admin ~]# ceph -s
  cluster:
    id:     a0b96b48-aeec-4da3-8609-f41630b92367
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-node02,ceph-node01,ceph-admin (age 2m)
    mgr: ceph-admin(active, since 6m)
    osd: 3 osds: 3 up (since 6m), 3 in (since 28m)
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 57 GiB / 60 GiB avail
    pgs:   
```








































