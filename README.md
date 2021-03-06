# kubernetes-for-china

## 安装 `Kubernetes`

1. [安装Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

```bash
$ curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

2. [安装 kubeadm, kubelet and kubectl]((https://kubernetes.io/docs/setup/independent/install-kubeadm/))

    - [Alibaba Kubernetes mirror](https://opsx.alibaba.com/mirror)

```bash
# root（sudo -i)

# Debian/Ubuntu
apt-get update && apt-get install -y apt-transport-https
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl

# CentOS/RHEL/Fedora
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
```

3. 预先从阿里的 `gcr.io` 镜像服务拉取必要的 `images`

```bash
$ ./load_images.sh
```

4. 使用 `kubeadm` 创建 `Kubernetes` 集群
```bash

# 确保关闭交换空间(running with swap on is not supported. Please disable swap)
$ sudo swapoff -a
# 永久关闭需要编辑 `/etc/fstab` 注释掉 `swap` 所在行

# 可以用下面的命令列出 kubeadm 需要的 images
$ kubeadm config images list --kubernetes-version=v1.15.0

# 集群初始化（init.yml文件中配置了使用阿里的镜像仓库）
$ sudo kubeadm init --config init.yml
# 或者执行(忽略Docker版本检查)
$ sudo kubeadm init --config init.yml --ignore-preflight-errors=SystemVerification

# 使用 `kube-router` 网络
$ kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml

# Master Isolation (if single-machine Kubernetes cluster )
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

## [Helm](https://github.com/kubernetes/helm)


```bash
# 安装
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.14.1-linux-amd64.tar.gz | tar xzv
$ sudo cp linux-amd64/helm /usr/local/bin
$ rm -rf linux-amd64

# 本地初始化，并将 `Tiller` 安装到 `Kubernetes` 集群
$ kubectl create -f helm-rbac-config.yaml
$ helm init --service-account tiller

# 更新本地 charts repo
$ helm repo update

# 测试安装 mysql chart
$ helm install --name my-mysql stable/mysql

# 删除 mysql
$ helm delete my-mysql

# 删除并释放该部署名以便重用
$ helm delete --purge my-mysql
```

## [Rook](https://github.com/rook/rook)

```bash
$ docker pull rook/ceph:master

# 安装 Rook Operator: https://rook.io/docs/rook/master/helm-operator.html
$ kubectl create -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml

# 创建 Rook cluster
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml

# 列出 rook-ceph 命名空间下的 pods
$ kubectl -n rook-ceph get pod

# 创建 storage pools.
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/pool.yaml
# 创建块存储(block storage)
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/storageclass.yaml

# 将 rook-block 设置为默认的 storageclass 
$ kubectl patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

```bash
# Shared File System

# Create the File System
kubectl create -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/filesystem.yaml

# 启动rook-ceph-tools pod
kubectl create -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/toolbox.yaml
# 进入 pod
kubectl -n rook-ceph exec -it rook-ceph-tools bash

# 获取挂载需要的主机挂载入口IP和用户密钥
mon_endpoints=$(grep mon_host /etc/ceph/ceph.conf | awk '{print $3}')
my_secret=$(grep key /etc/ceph/keyring | awk '{print $3}')

# 如果在普通的docker 容器中挂载需要这样启动容器
docker run -it --rm --privileged -v /lib/modules:/lib/modules ubuntu bash

# 创建挂载目录
mkdir /cephfs
# 挂载文件系统
mount -t ceph -o mds_namespace=myfs,name=admin,secret=$my_secret $mon_endpoints:/ /cephfs
# 查看挂载的文件系统
df -h

# 卸载文件系统
umount /cephfs
```

```bash
# Object Storage

# Create an Object Store
kubectl create -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/object.yaml

# To confirm the object store is configured, wait for the rgw pod to start
kubectl -n rook-ceph get pod -l app=rook-ceph-rgw

# Create a User
kubectl create -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/object-user.yaml

# To confirm the object store user is configured, describe the secret
kubectl -n rook-ceph describe secret rook-ceph-object-user-my-store-my-user

# To directly retrieve the secrets:
kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user -o yaml | grep AccessKey | awk '{print $2}' | base64 --decode
kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user -o yaml | grep SecretKey | awk '{print $2}' | base64 --decode

# Access External to the Cluster
kubectl create -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/rgw-external.yaml

# See both rgw services running and notice what port the external service is running on:
kubectl -n rook-ceph get service rook-ceph-rgw-my-store rook-ceph-rgw-my-store-external

# Consume the Object Storage
export AWS_ENDPOINT=10.104.35.31:80
export AWS_ACCESS_KEY_ID=XEZDB3UJ6X7HVBE7X7MA
export AWS_SECRET_ACCESS_KEY=7yGIZON7EhFORz0I40BFniML36D2rl8CQQ5kXU6l
```


## 升级 Kubernetes 版本
```bash
# 修改 `init.yml` 中 `kubernetesVersion` 版本号， 执行
sudo kubeadm upgrade apply --config init.yml --ignore-preflight-errors=SystemVerification
```
