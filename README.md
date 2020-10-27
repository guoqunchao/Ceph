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

#### 部署ceph和ceph-deploy
```shell
#每个主机安装 ceph
yum install ceph -y

#Admin节点安装
yum install ceph-deploy -y
```

#### 部署监控节点（mon）
```shell
ceph-deploy new ceph-admin ceph-node01 ceph-node02
vim  /root/.cephdeploy.conf  #修改这个文件，添加：  overwrite_conf = true
ceph-deploy  --overwrite-conf  mon create-initial
systemctl status ceph-mon@ceph-node01 #重启命令
```

#### 部署mgr
```shell
ceph-deploy mgr create ceph-admin
```



问题01 ImportError: No module named pkg_resources
```shell
yum -y install python2-pip
```















































