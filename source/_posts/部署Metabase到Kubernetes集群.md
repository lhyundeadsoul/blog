---
title: 部署Metabase到Kubernetes集群
date: 2020-04-14 19:52:35
tags: Kubernetes
---

### 部署Kubernetes集群
<!-- more -->

1. 初始化集群master

  ``` shell
  kubeadm reset
  kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.11.0 ##记录下最后出现的kubeadm join xxxx
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

2. 添加集群网络
   ``` shell
   kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```
   
3. 添加slave结点

   ``` shell
   kubeadm join xxxx ##取自kubeadm init的结果，也可以使用命令生成： kubeadm token create --print-join-command
   ```


4. 检查集群部署情况

   ``` shell
   kubectl cluster-info  
   kubectl get nodes
   ```



### 部署helm

1. 部署Tiller（helm服务端）

   ``` shell
   helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.12.1 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts ##注意tiller和helm client版本要一致
   ```

2. 创建 Tiller 的ServiceAccount并绑定cluster-admin角色

   ``` shell
   kubectl create serviceaccount --namespace kube-system tiller     
   kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
   ```

3. 给 Tiller 的 deployments 添加刚才创建的 ServiceAccount

   ``` shell
   kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
   ```

4. 查看 Tiller deployments 资源是否绑定 ServiceAccount以及Tiller 是否安装成功

   ``` shell
   kubectl get deploy -n kube-system tiller-deploy -o yaml | grep serviceAccount
   helm version ##注意tiller和helm client版本要一致
   ```



### 部署metabase（VPC环境）

1. 在VPC环境外获取metabase Chart文件

   ``` shell
   git clone https://github.com/helm/charts.git
   helm package charts/stable/metabase -d xxxx#本地某目录
   ```

2. 在VPC环境外获取metabase image文件

   ```shell
    docker pull metabase/metabase
    docker save metabase/metabase -o metabase.img
   ```

3. 手动传入Chart文件和image文件到VPC环境

   ```shell
   #传入跳板机
   scp  -P 28265 metabase.img {user}@{ip}:/home/{user}/metabase
   scp  -P 28265 metabase-0.10.2.tgz {user}@{ip}:/home/{user}/metabase
   #传入VPC内（忽略）
   ```

4. 进入VPC环境，处理image文件

   ``` shell
   docker load -i metabase.img
   docker tag metabase/metabase:latest {registry}/metabase/metabase:v0.34.0#这里的{registry}一定要是环境内可通的docker registry，可kubectl describe其它已部署pod查找
   docker push {registry}/metabase/metabase:v0.34.0
   ```
   
5. 修改Chart配置：values.yaml的各项配置

   ``` shell
   image:
     repository: {registry}/metabase/metabase#这里要和docker tag保持一致
     tag: v0.34.0
     pullPolicy: IfNotPresent
   ```
   ``` shell
   service:
     name: metabase
     type: NodePort#需要外网访问metabase时，要用NodePort
     externalPort: 80
     internalPort: 3000
     nodePort: 30303#增加NodePort
     annotations:
   ```
   ``` shell
   database:
      type: #postgres或mysql
      host: xxxx
      port: xxxx
      dbname: xxxx
      username: xxxx
      password: "xxxxx"#注意这里有引号
   ```


5. 安装metabase


  ``` shell
  helm install --name my-bi-release --values=values.yaml ./metabase#chart所在目录
  ```



### 部署中可能会遇到的问题

1. 集群初始化问题：--pod-network-cidr=10.244.0.0/16 使用kube-flannel网络时，此参数必须要有

2. 在k8s反复增删node时，需要在对应node执行`kubeadm reset`，直接删除node会有网络没及时清理的问题。可尝试：
   ```shell
   #重置kubernetes服务，重置网络。删除网络配置，link
   kubeadm reset
   systemctl stop kubelet
   systemctl stop docker
   rm -rf /var/lib/cni/
   rm -rf /var/lib/kubelet/*
   rm -rf /etc/cni/
   ifconfig cni0 down
   ifconfig flannel.1 down
   ifconfig docker0 down
   ip link delete cni0
   ip link delete flannel.1
   systemctl start docker
   ```
   
3. k8s账号权限问题：每个服务要有对应的ServiceAccount并赋予权限

4. Helm pull image error：VPC环境下要改helm values.yaml中的镜像地址为环境内的docker registry，类似`{registry}/metabase/metabase:v0.34.0`
