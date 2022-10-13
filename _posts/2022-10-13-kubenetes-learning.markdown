---
layout: post
title:  "Kubenetes Learning"
date:   2022-10-13 21:02:00 +0200
categories: jekyll update
---

打算下个月（hopefully）考一个CKA的证，看的课程貌似都是在用Linux的系统，所以打算在这里写学习笔记，和一些遇到问题的记录，方便以后记起。

- install kubeadm: `sudo apt-get install kubeadm`
- initialize kubenetes: `sudo kubeadm init`
    - 遇到一个小问题类似于 stack overflow (https://stackoverflow.com/questions/72504257/i-encountered-when-executing-kubeadm-init-error-issue). 大概就是在初始的config.toml里面的默认设置是： disabled_plugins = ["cri"]. 于是我就编辑了/etc/containerd/config.toml的这个文件（需要先chmod），删除了“cri”然后运行`systemctl restart containerd` 再重新 `sudo kubeadm init`.
    - Another problem: swap. Kubelet 无法正常运行，提示用`journalctl -xeu kubelet` 查看日志，发现有`command failed" err="failed to run Kubelet: running with swap on is not supported, please disable swap!` 错误，搜索使用了 https://stackoverflow.com/questions/52119985/kubeadm-init-shows-kubelet-isnt-running-or-healthy 里面提到的一个方法
        ```
        cd /etc/systemd/system/kubelet.service.d
        touch 20-allow-swap.conf
        ```
        然后文件内容为
        ```
        [Service] 
        Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false”
        ```
        最后使用`sudo kubeadm init --ignore-preflight-errors=Swap`
    - 完成后的提示：To start using your cluster, you need to run the following as a regular user:
            mkdir -p $HOME/.kube
            sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
            sudo chown $(id -u):$(id -g) $HOME/.kube/config
- 在通过了 Preflight Checks 之后，kubeadm 要为你做的，是生成 Kubernetes 对外提供服务所需的各种证书和对应的目录。Kubernetes 对外提供服务时，除非专门开启“不安全模式”，否则都要通过 HTTPS 才能访问 kube-apiserver。这就需要为 Kubernetes 集群配置好证书文件。kubeadm 为 Kubernetes 项目生成的证书文件都放在 Master 节点的 /etc/kubernetes/pki 目录下。在这个目录下，最主要的证书文件是 ca.crt 和对应的私钥 ca.key。此外，用户使用 kubectl 获取容器日志等 streaming 操作时，需要通过 kube-apiserver 向 kubelet 发起请求，这个连接也必须是安全的。kubeadm 为这一步生成的是 apiserver-kubelet-client.crt 文件，对应的私钥是 apiserver-kubelet-client.key。除此之外，Kubernetes 集群中还有 Aggregate APIServer 等特性，也需要用到专门的证书，这里我就不再一一列举了。需要指出的是，你可以选择不让 kubeadm 为你生成这些证书，而是拷贝现有的证书到如下证书的目录里：/etc/kubernetes/pki/ca.{crt,key}
- 证书生成后，kubeadm 接下来会为其他组件生成访问 kube-apiserver 所需的配置文件。这些文件的路径是：/etc/kubernetes/xxx.conf：ls /etc/kubernetes/admin.conf controller-manager.conf kubelet.conf scheduler.conf这些文件里面记录的是，当前这个 Master 节点的服务器地址、监听端口、证书目录等信息。这样，对应的客户端（比如 scheduler，kubelet 等），可以直接加载相应的文件，使用里面的信息与 kube-apiserver 建立安全连接。
- 在 kubeadm 中，Master 组件的 YAML 文件会被生成在 /etc/kubernetes/manifests 路径下。比如，kube-apiserver.yaml
- 然后，kubeadm 就会为集群生成一个 bootstrap token。在后面，只要持有这个 token，任何一个安装了 kubelet 和 kubadm 的节点，都可以通过 kubeadm join 加入到这个集群当中。

- problem： kubeadm 过于新，1.25。 要更改教程所提的kubeadm.yaml. 参考： https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/
改成了以下可以用：
    ```
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration
    # controllerManager:
    #  extraArgs:
    #    horizontal-pod-autoscaler-use-rest-clients: "true"
    #    horizontal-pod-autoscaler-sync-period: "10s"
    #    node-monitor-grace-period: "10s"
    apiServer:
    extraArgs:
        runtime-config: "api/all=true"
    kubernetesVersion: "v1.25.0"
    ```

- rerun `sudo kubeadm init` will give error of `FileAvailablexxxxxxx`, 可以用`sudo kubeadm reset` 再重试。

- 记录下 kubeadm join 192.168.0.14:6443 --token xxxx 这个提示的命令后 运行 `kubectl get pods` 运行有问题
`Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")`
发现原来是提示的暴露kubeconfig的命令没有认真打。
```
kubectl get nodes
NAME          STATUS     ROLES           AGE     VERSION
yuan-studio   NotReady   control-plane   7m28s   v1.25.1
```
不知道为什么node的名字不是master而是用户名，之后更熟练了之后再来更改。

- 此时node处在NotReady的状态，通过用kubectl describe node xxx得知
`KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized`

- 可以通过 kubectl 检查这个节点上各个系统 Pod 的状态，其中，kube-system 是 Kubernetes 项目预留的系统 Pod 的工作空间（Namepsace，注意它并不是 Linux Namespace，它只是 Kubernetes 划分不同工作空间的单位）

`kubectl get pods -n kube-system`
此时kube-controller-manager-xxxx在CrashLoopBackOff 和教程所说的Pending state 不同
发现其实是之前定义的kubeadm.yaml的问题，需要去掉controller manager的相关设置，新版本已经不支持。

- 安装网络插件
`kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml`
kubeadm join 192.168.0.14:6443 --token rp725d.jzb4ebz39vjzn1f2 \
	--discovery-token-ca-cert-hash sha256:075cfe7fb86dc746e2def383f0e4701cf7a9867a15882685782836100a1616ce

- Kubernetes 的 Worker 节点跟 Master 节点几乎是相同的，它们运行着的都是一个 kubelet 组件。唯一的区别在于，在 kubeadm init 的过程中，kubelet 启动后，Master 节点上还会自动运行 kube-apiserver、kube-scheduler、kube-controller-manger 这三个系统 Pod。

部署 Worker 节点反而是最简单的，只需要两步即可完成。第一步，在所有 Worker 节点上执行“安装 kubeadm 和 Docker”一节的所有步骤。第二步，执行部署 Master 节点时生成的 kubeadm join 指令：$ kubeadm join 10.168.0.2:6443 --token 00bwbx.uvnaa2ewjflwu1ry
明天在另一台电脑试试

- 默认情况下 Master 节点是不允许运行用户 Pod 的。而 Kubernetes 做到这一点，依靠的是 Kubernetes 的 Taint/Toleration 机制。它的原理非常简单：一旦某个节点被加上了一个 Taint，即被“打上了污点”，那么所有 Pod 就都不能在这个节点上运行，因为 Kubernetes 的 Pod 都有“洁癖”。除非，有个别的 Pod 声明自己能“容忍”这个“污点”，即声明了 Toleration，它才可以在这个节点上运行。
```
apiVersion: v1kind: Pod...spec: tolerations: - key: "foo" operator: "Exists" effect: "NoSchedule"
```

`kubectl describe node master`
`Taints:             node-role.kubernetes.io/control-plane:NoSchedule`
默认master是tainted.

当然，如果你就是想要一个单节点的 Kubernetes，删除这个 Taint 才是正确的选择：$ kubectl taint nodes --all node-role.kubernetes.io/master-如上所示，我们在“node-role.kubernetes.io/master”这个键后面加上了一个短横线“-”，这个格式就意味着移除所有以“node-role.kubernetes.io/master”为键的 Taint。
实际上的taint名是 node-role.kubernetes.io/control-plane-
node
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
node/yuan-studio untainted
```
安装dashboard
`https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml`
后生成一个新的namespace `kubernetes-dashboard` 
```
kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-64bcc67c9c-z2ktq   1/1     Running   0          53s
kubernetes-dashboard-5c8bd6b59-6bq84         1/1     Running   0          53s
```


kubeadm join 192.168.0.14:6443 --token rp725d.jzb4ebz39vjzn1f2 \
	--discovery-token-ca-cert-hash sha256:075cfe7fb86dc746e2def383f0e4701cf7a9867a15882685782836100a1616ce

可是，如果你在某一台机器上启动的一个容器，显然无法看到其他机器上的容器在它们的数据卷里写入的文件。这是容器最典型的特征之一：无状态。而容器的持久化存储，就是用来保存容器存储状态的重要手段：存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。这样，无论你在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容。这就是“持久化”的含义。

使用rook 没有成功，最新的common.yaml报错，之后再尝试。
`kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/common.yaml`

- 使用一种 API 对象（Deployment）管理另一种 API 对象（Pod）的方法，在 Kubernetes 中，叫作“控制器”模式（controller pattern）。在我们的例子中，Deployment 扮演的正是 Pod 的控制器的角色。

- 每一个 API 对象都有一个叫作 Metadata 的字段，这个字段就是 API 对象的“标识”，即元数据，它也是我们从 Kubernetes 里找到这个对象的主要依据。这其中最主要使用到的字段是 Labels。顾名思义，Labels 就是一组 key-value 格式的标签。而像 Deployment 这样的控制器对象，就可以通过这个 Labels 字段从 Kubernetes 中过滤出它所关心的被控制对象。

还有一个与 Labels 格式、层级完全相同的字段叫 Annotations，它专门用来携带 key-value 格式的内部信息。所谓内部信息，指的是对这些信息感兴趣的，是 Kubernetes 组件本身，而不是用户。所以大多数 Annotations，都是在 Kubernetes 运行过程中，被自动加在这个 API 对象上。

一个 Kubernetes 的 API 对象的定义，大多可以分为 Metadata 和 Spec 两个部分。前者存放的是这个对象的元数据，对所有 API 对象来说，这一部分的字段和格式基本上是一样的；而后者存放的，则是属于这个对象独有的定义，用来描述它所要表达的功能。

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

then do `kubectl create -f nginx-deployment.yaml`.
在命令行中，所有 key-value 格式的参数，都使用“=”而非“:”表示。
`kubectl get pods -l app=nginx` 

可以使用 kubectl describe 命令，查看一个 API 对象的细节
`kubectl describe pod nginx-deployment-67594d6bf6-9gdvr`

推荐使用 `kubectl apply -f xxxxx.yaml`

```
spec: selector: matchLabels: app: nginx replicas: 2 template: metadata: labels: app: nginx spec: containers: - name: nginx image: nginx:1.8 ports: - containerPort: 80 volumeMounts: - mountPath: "/usr/share/nginx/html" name: nginx-vol volumes: - name: nginx-vol emptyDir: {}
```
`emptyDir`不显式声明宿主机目录的 Volume。所以，Kubernetes 也会在宿主机上创建一个临时目录，这个目录将来就会被绑定挂载到容器所声明的 Volume 目录上。

Kubernetes 也提供了显式的 Volume 定义，它叫作 hostPath。比如下面的这个 YAML 文件： ... volumes: - name: nginx-vol hostPath: path: " /var/data"这样，容器 Volume 挂载的宿主机目录，就变成了 /var/data。

`kubectl exec -it nginx-deployment-5c678cfb6d-lg9lw -- /bin/bash`
`kubectl delete -f nginx-deployment.yaml`