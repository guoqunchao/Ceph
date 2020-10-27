# ceph
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











































