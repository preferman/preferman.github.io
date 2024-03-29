---
title: Kubernete with docker
date: 2023-01-25 15:38:06
---


## Kubernete with docker

### 组件介绍
https://kubernetes.io/zh-cn/docs/concepts/overview/components/#node-components

### 裸机安装

https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/


#### 1.安装环境
安装docker看[查看本博客其它文章](http://preferman.github.io/2021/11/28/docker/)

##### 安装容器运行时
Docker Engine 没有实现 CRI， 而这是容器运行时在 Kubernetes 中工作所需要的。 为此，必须安装一个额外的服务 cri-dockerd。 cri-dockerd 是一个基于传统的内置 Docker 引擎支持的项目， 它在 1.24 版本从 kubelet 中移除。
1. 根据linux发行版下载二进制文件[地址](https://github.com/Mirantis/cri-dockerd/releases)，也可以自行编译
2. 安装cri-dockerd
3. 执行命令`systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket`
> 不同linux发行版有所差异

##### [安装 kubeadm、kubelet 和 kubectl](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

#### 2.配置容器运行时驱动
k8s使用docker作为容器：


查看当前Docker使用的Cgroup Driver：
`docker info | grep "Cgroup Driver"`

修改Cgroup Driver
```
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
systemctl daemon-reload
systemctl restart docker
```

#### 3.初始化控制平面节点

##### 准备

选择一个 Pod 网络插件（此时不安装需要传递参数），并验证是否需要为 kubeadm init 传递参数。 根据你选择的第三方网络插件，你可能需要设置 --pod-network-cidr 的值。 请参阅[安装 Pod 网络附加组件](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)。
本教程使用如下插件：https://github.com/flannel-io/flannel/

`kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock`

成功的标志:
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

//加入集群时使用，需要记下来
  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

##### 遇到的问题
> 最好使用香港的服务器，国内其它地方的可能会出现镜像拉去不到的情况，可能是个bug,指定的镜像仓库未生效。
> 解决办法：docker配置能访问外网的代理，[方法](https://yeasy.gitbook.io/docker_practice/advanced_network/http_https_proxy#wei-dockerd-she-zhi-wang-luo-dai-li)
>`Error syncing pod, skipping" err="failed to \"CreatePodSandbox\" for \"kube-controller-manager-iz8vbia14zbe5y5mt3rka9z_kube-system(786d13527624f9f10ab8c412000ab506)\" with CreatePodSandboxError: \"Failed to create sandbox for pod \\\"kube-controller-manager-iz8vbia14zbe5y5mt3rka9z_kube-system(786d13527624f9f10ab8c412000ab506)\\\": rpc error: code = Unknown desc = failed pulling image \\\"registry.k8s.io/pause:3.6\\\": Error response from daemon: Head \\\"https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/pause/manifests/3.6\\\": dial tcp 64.233.187.82:443: i/o timeout\"" pod="kube-system/kube-controller-manager-iz8vbia14zbe5y5mt3rka9z" podUID=786d13527624f9f10ab8c412000ab506

>/usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:///var/run/cri-dockerd.sock --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9`



#### 4.将工作节点加入集群

在工作节点执行上述init之后生成的命令`kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>`

状态：
```
[root@iZ8vbia14zbe5y5mt3rka9Z ~]# kubectl get nodes
NAME                      STATUS     ROLES           AGE   VERSION
iz8vba21wy1jbsrm04fdrbz   NotReady   <none>          25s   v1.26.1
iz8vbia14zbe5y5mt3rka9z   NotReady   control-plane   51m   v1.26.1
```
[安装 Pod 网络附加组件](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)：

执行命令：`kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml`。

安装 Pod 网络后，你可以通过在 `kubectl get pods --all-namespaces` 输出中检查 CoreDNS Pod 是否 Running 来确认其是否正常运行。 一旦 CoreDNS Pod 启用并运行，你就可以继续加入节点
```
[root@iZ8vbia14zbe5y5mt3rka9Z ~]# kubectl get pods --all-namespaces
NAMESPACE      NAME                                              READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-5z22d                             1/1     Running   0          16s
kube-flannel   kube-flannel-ds-c57lm                             1/1     Running   0          16s
kube-system    coredns-787d4945fb-7mz59                          1/1     Running   0          4m1s
kube-system    coredns-787d4945fb-9h7mm                          1/1     Running   0          4m1s
kube-system    etcd-iz8vbia14zbe5y5mt3rka9z                      1/1     Running   0          4m14s
kube-system    kube-apiserver-iz8vbia14zbe5y5mt3rka9z            1/1     Running   0          4m15s
kube-system    kube-controller-manager-iz8vbia14zbe5y5mt3rka9z   1/1     Running   0          4m14s
kube-system    kube-proxy-tfrb5                                  1/1     Running   0          89s
kube-system    kube-proxy-vzf79                                  1/1     Running   0          4m1s
kube-system    kube-scheduler-iz8vbia14zbe5y5mt3rka9z            1/1     Running   0          4m16s
```


最终状态：
```
[root@iZ8vbia14zbe5y5mt3rka9Z ~]# kubectl get nodes
NAME                      STATUS   ROLES           AGE    VERSION
iz8vba21wy1jbsrm04fdrbz   Ready    <none>          58m    v1.26.1
iz8vbia14zbe5y5mt3rka9z   Ready    control-plane   109m   v1.26.1
```


> 加入集群的节点也需要开启代理（香港服务器轻忽略）,需要拉去镜像



#### 删除节点

https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#remove-the-node

