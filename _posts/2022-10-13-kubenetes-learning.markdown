---
layout: post
title:  "Kubenetes Learning"
date:   2022-10-16 21:02:00 +0200
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

- POD
  - 由于容器永远只能管理一个进程。而当有不同进程之间有紧密关系，例如需要共享资源时，是一个难解决的问题。而Kubenetes通过定义最小单位是pod可以很好的解决这个问题，让用户把有强关联的容器部署在同一个pod里，这样很好的解决了资源共享等问题。对于pod中的容器A和B来说：
    - 它们可以直接使用 localhost 进行通信；
    - 它们看到的网络设备跟 Infra 容器看到的完全一样；
    - 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；
    - 当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；
    - Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关。
    其中infra容器永远是在pod中被第一个创建的中间容器，而其他用户的定义容器，则通过join network namespace的方式，与Infra容器关联在一起。
  - 当我们想在一个容器跑多个功能不相关的应用时，应优先考虑是不是应为一个Pod里的多个容器。
    - 例如war包和web服务器，可以用一个Init container来负责copy war包到一个共享的volume里。
    - 例如容器的日志收集也可以放在一个container专门去做。
    - 如上两个例子均为sidecar的设计模式
  - 在 Pod 中，所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。
  - 可以把整个虚拟机想象成为一个 Pod，把这些进程分别做成容器镜像，把有顺序关系的容器，定义为 Init Container。这才是更加合理的、松耦合的容器编排诀窍，也是从传统应用架构，到“微服务架构”最自然的过渡方式。
  - Pod中重要的字段含义与用法
    1. NodeSelector: 将pod与node绑定的字段
      ```
      apiVersion: v1
      kind: Pod
      ...
      spec:
      nodeSelector:
        disktype: ssd #表示这个pod只运行在有disktype:ssd标签（label）的node上
      ```
    2. NodeName: 一旦这个字段被赋值，kubenetes会认为这个pod已经被调度，一般由调度器负责设置，偶尔调试或者测试的时候用户可以设置来‘欺骗’调度器
    3. HostAliases:定义Pod的Hosts文件（e.t. /etc/hosts）
      ```
      spec:  hostAliases:  
      - ip: "10.1.2.3"    
        hostnames:    
        - "foo.remote"    
        - "bar.remote"
      ```
      当这个Pod启动 /etc/hosts 中的内容将为：
      ```
      # Kubernetes-managed hosts file.
      127.0.0.1 localhost
      ...
      10.244.135.10 hostaliases-pod
      10.1.2.3 foo.remote
      10.1.2.3 bar.remote
      ```
      在 Kubernetes 项目中，如果要设置 hosts 文件里的内容，一定要通过这种方法。否则，如果直接修改了 hosts 文件的话，在 Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容。
    凡是跟容器的 Linux Namespace 相关的属性，也一定是 Pod 级别的。这个原因也很容易理解：Pod 的设计，就是要让它里面的容器尽可能多地共享 Linux Namespace，仅保留必要的隔离和限制能力。这样，Pod 模拟出的效果，就跟虚拟机里程序间的关系非常类似了。
    4. shareProcessNamespace=true: 意味着pod里的容器共享PID Namespace.
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: nginx
        spec:
          shareProcessNamespace: true
          containers:
          - name: nginx
            image: nginx
          - name: shell
            image: busybox
            stdin: true #在 Pod 的 YAML 文件里声明开启它们俩，其实等同于设置了 docker run 里的 -it（-i 即 stdin，-t 即 tty）参数。
            tty: true
        ```
        创建这个pod (`kubectl create -f nginx.yaml`) 后，使用`kubectl attach -it nginx -c shell`可以在shell容器里执行ps指令。此时可以看到：

        不仅可以看到它本身的 ps ax 指令，还可以看到 nginx 容器的进程，以及 Infra 容器的 /pause 进程。这就意味着，整个 Pod 里的每个容器的进程，对于所有容器来说都是可见的：它们共享了同一个 PID Namespace。
    5. hostNetwork: true hostIPC: true hostPID: true 这些定义共享宿主机的Network, IPC 和PID Namespace. 意味着，这个Pod里的所有容器，会直接使用宿主机的网络，直接与宿主机进行IPC通信，看到宿主机里正在运行的所有进程。
    6. pod里最重要的字段是 Containers.
      - Init Containers: init container and container 两个字段都属于 Pod 对容器的定义，内容也完全相同，只是 Init Containers 的生命周期，会先于所有的 Containers，并且严格按照定义的顺序执行。
      - Kubernetes 项目中对 Container 的定义，和 Docker 相比并没有什么太大区别 (e.g. Image, comamnd, workingDir, ports, volumnMounts)
      - ImagePullPolicy: 定义镜像拉取策略。默认值always即每次创建Pod都重新拉去一次镜像，另外，当容器的镜像是类似于 nginx 或者 nginx:latest 这样的名字时，ImagePullPolicy 也会被认为 Always。若定义为Never或者IfNotPresent,则意味着Pod永远不会主动拉取这个镜像，或者只在宿主机不存在这个镜像时才拉取。
      - Lifecycle: 
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: lifecycle-demo
        spec:
          containers:
          - name: lifecycle-demo-container
            image: nginx
            lifecycle:
              postStart:
                exec:
                  command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
              preStop:
                exec:
                  command: ["/usr/sbin/nginx","-s","quit"]`
          ```
          - postStart: 在容器启动后立刻执行一个操作，但不严格保证顺序。也就是说，在postStart启动时，Entrypoint有可能还没有结束。如果 postStart 执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态。
    - Pod 生命周期的变化，主要体现在 Pod API 对象的 Status 部分，这是它除了 Metadata 和 Spec 之外的第三个重要字段。其中，pod.status.phase，就是 Pod 的当前状态，它有如下几种可能的情况：
      - Pending。这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
      - Running。这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。
      - Succeeded。这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。
      - Failed。这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
      - Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。
      - 更进一步地，Pod 对象的 Status 字段，还可以再细分出一组 Conditions。这些细分状态的值包括：PodScheduled、Ready、Initialized，以及 Unschedulable。它们主要用于描述造成当前 Status 的具体原因是什么。比如，Pod 当前的 Status 是 Pending，对应的 Condition 是 Unschedulable，这就意味着它的调度出现了问题。而其中，Ready 这个细分状态非常值得我们关注：它意味着 Pod 不仅已经正常启动（Running 状态），而且已经可以对外提供服务了。这两者之间（Running 和 Ready）是有区别的，你不妨仔细思考一下。
    - 全部字段见：https://github.com/kubernetes/api/blob/master/core/v1/types.go

  - Projected Volume. 一种特殊的Volume, 存在的意义不是为了存放数据，也不是用来进行容器和宿主机的数据交换。是为了给容器提供预先定义好的数据。Kubernetes支持一下四种
    1. Secret. 把 Pod 想要访问的加密数据，存放到 Etcd 中。然后，你就可以通过在 Pod 的容器里挂载 Volume 的方式，访问到这些 Secret 里保存的信息了。
      - 例子：
      ```
      apiVersion: v1
      kind: Pod
      metadata:
        name: test-projected-volume 
      spec:
        containers:
        - name: test-secret-volume
          image: busybox
          args:
          - sleep
          - "86400"
          volumeMounts:
          - name: mysql-cred
            mountPath: "/projected-volume"
            readOnly: true
        volumes:
        - name: mysql-cred
          projected:
            sources:
            - secret:
                name: user
            - secret:
                name: pass
        ```
      - 可以通过`kubectl create secret generic secret-key --from-file=./secret-file.txt`来定义secret。也可以通过（更常用）通过定义Secret yaml file 例如：
        ```
        apiVersion: v1
        kind: Secret
        metadata:
          name: mysecret
        type: Opaque
        data:
          user: YWRtaW4=
          pass: MWYyZDFlMmU2N2Rm
        ```
        注意此时创建出来的Secret对象只有一个， 但data字段定义了两份Secret数据，此时挂载的secret的名字，是mysecret。但如果进入pod里查看（路径为 /projected-volume里）,会有user and pass两个密码文件。
      - Secret 对象要求这些数据必须是经过 Base64 转码的，以免出现明文密码的安全隐患。在真正的生产环境中，你需要在 Kubernetes 中开启 Secret 的加密插件，增强数据的安全性
      - 像这样通过挂载方式进入到容器里的 Secret，一旦其对应的 Etcd 里的数据被更新，这些 Volume 里的文件内容，同样也会被更新。其实，这是 kubelet 组件在定时维护这些 Volume。需要注意的是，这个更新可能会有一定的延时。所以在编写应用程序时，在发起数据库连接的代码处写好重试和超时的逻辑，绝对是个好习惯。
    2. ConfigMap. 与Secret相似，区别是其保存的是不需要加密的、应用所需的配置信息。用法与Secret几乎完全相同。可以用`kubectl create configmap`从文件或者目录创建`ConfigMap`,也可以直接编写 yaml 文件。
      例如：
      ```
      # .properties文件的内容
      $ cat example/ui.properties
      color.good=purple
      color.bad=yellow
      allow.textmode=true
      how.nice.to.look=fairlyNice

      # 从.properties文件创建ConfigMap
      $ kubectl create configmap ui-config --from-file=example/ui.properties

      # 查看这个ConfigMap里保存的信息(data)
      $ kubectl get configmaps ui-config -o yaml #kubectl get -o yaml 这样的参数，会将指定的 Pod API 对象以 YAML 的方式展示出来。
      apiVersion: v1
      data:
        ui.properties: |
          color.good=purple
          color.bad=yellow
          allow.textmode=true
          how.nice.to.look=fairlyNice
      kind: ConfigMap
      metadata:
        name: ui-config
        ...
      ```
    3. Downward API. 让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息。
      ```
      volumes:
        - name: podinfo 
          projected: 
            sources: 
            - downwardAPI: 
                items: 
                  - path: "labels"
                    fieldRef: 
                    fieldPath: metadata.labels
      ```
      这里Downward API Volume，声明了要暴露 Pod 的 metadata.labels 信息给容器。
      Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。而如果你想要获取 Pod 容器运行后才会出现的信息，比如，容器进程的 PID，那就肯定不能使用 Downward API 了，而应该考虑在 Pod 里定义一个 sidecar 容器。
    ！！！Secret、ConfigMap，以及 Downward API 这三种 Projected Volume 定义的信息，大多还可以通过环境变量的方式出现在容器里。但是，通过环境变量获取这些信息的方式，不具备自动更新的能力。所以，一般情况下，我都建议你使用 Volume 文件的方式获取这些信息。
    4. ServiceAccountToken
      - Service Account 对象的作用，就是 Kubernetes 系统内置的一种“服务账户”，它是 Kubernetes 进行权限分配的对象。比如，Service Account A，可以只被允许对 Kubernetes API 进行 GET 操作，而 Service Account B，则可以有 Kubernetes API 的所有操作权限。像这样的 Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作 ServiceAccountToken。任何运行在 Kubernetes 集群上的应用，都必须使用这个 ServiceAccountToken 里保存的授权信息，也就是 Token，才可以合法地访问 API Server。
      - 任意一个运行在 Kubernetes 集群里的 Pod，就会发现，每一个 Pod，都已经自动声明一个类型是 Secret、名为 default-token-xxxx 的 Volume (应该是版本更新后是kube-api-access-xxxx)，然后 自动挂载在每个容器的一个固定目录上。
          e.g. describe之后有这一行：`Mounts: /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-tqtxr (ro)`
        - 应用程序只要直接加载这些授权文件，就可以访问并操作 Kubernetes API 了。而且，如果使用的是 Kubernetes 官方的 Client 包（k8s.io/client-go）的话，它还可以自动加载这个目录下的文件，你不需要做任何配置或者编码操作。这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作“InClusterConfig”，也是最推荐的进行 Kubernetes API 编程的授权方式。
      - 自动挂载默认 ServiceAccountToken 的潜在风险，Kubernetes 允许你设置默认不为 Pod 里的容器自动挂载这个 Volume。除了这个默认的 Service Account 外，我们很多时候还需要创建一些我们自己定义的 Service Account，来对应不同的权限设置。这样，我们的 Pod 里的容器就可以通过挂载这些 Service Account 对应的 ServiceAccountToken，来使用这些自定义的授权信息。
  - 容器健康检查和回复机制
    ```
    apiVersion: v1
    kind: Pod
    metadata: 
      labels:
        test: liveness 
        name: test-liveness-exec
    spec: 
      containers: 
      - name: liveness 
        image: busybox 
        args: 
        - /bin/sh 
        - -c 
        - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600 
        livenessProbe: 
          exec: 
            command: 
            - cat 
            - /tmp/healthy 
          initialDelaySeconds: 5 
          periodSeconds: 5
    ```
    此pod启动三十秒后健康检查没有通过，会重启container （也就是重新创建新的container）
    这个功能就是 Kubernetes 里的 Pod 恢复机制，也叫 restartPolicy。它是 Pod 的 Spec 部分的一个标准字段（pod.spec.restartPolicy），默认值是 Always，即：任何时候这个容器发生了异常，它一定会被重新创建。
    但一定要强调的是，Pod 的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去。事实上，一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node 字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。而如果你想让 Pod 出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理 Pod，哪怕你只需要一个 Pod 副本。
    而作为用户，你还可以通过设置 restartPolicy，改变 Pod 的恢复策略。除了 Always，它还有 OnFailure 和 Never 两种情况：Always：在任何情况下，只要容器不在运行状态，就自动重启容器；OnFailure: 只在容器 异常时才自动重启容器；Never: 从来不重启容器。在实际使用时，我们需要根据应用运行的特性，合理设置这三种恢复策略。比如，一个 Pod，它只计算 1+1=2，计算完成输出结果后退出，变成 Succeeded 状态。这时，你如果再用 restartPolicy=Always 强制重启这个 Pod 的容器，就没有任何意义了。而如果你要关心这个容器退出后的上下文环境，比如容器退出后的日志、文件和目录，就需要将 restartPolicy 设置为 Never。因为一旦容器被自动重新创建，这些内容就有可能丢失掉了（被垃圾回收了）。
    ！基本设计原理
      1. 只要 Pod 的 restartPolicy 指定的策略允许重启异常的容器（比如：Always），那么这个 Pod 就会保持 Running 状态，并进行容器重启。否则，Pod 就会进入 Failed 状态 。
      2. 对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数
    livenessProbe更常用的是发起http （pod暴露一个健康检查的URL比如/healthz）或者tcp请求的方式
    ```
    ...
    livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
          httpHeaders:
          - name: X-Custom-Header
            value: Awesome
          initialDelaySeconds: 3
          periodSeconds: 3
    ```
    ```
    ...
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
    ```
    在 Kubernetes 的 Pod 中，还有一个叫 readinessProbe 的字段。虽然它的用法与 livenessProbe 类似，但作用却大不一样。readinessProbe 检查结果的成功与否，决定的这个 Pod 是不是能被通过 Service 的方式访问到，而并不影响 Pod 的生命周期。
  - PodPreset对象. 让kubernetes自动给Pod填充字段。减少开发人员编写Pod Yaml的门槛. 已deprecate可用admission webhooks来在创建时修改Pod.
    ```
- 编排 - 控制器模型
  - 控制器模型即是用一种对象管理另外一种对象
  - 控制循环 = 调谐循环（Reconcile Loop） = 同步循环（Sync Loop）
  - 实现：
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
    Deployment 控制器从 Etcd 中获取到所有携带了“app: nginx”标签的 Pod，然后统计它们的数量，这就是实际状态；
    Deployment 对象的 Replicas 字段的值就是期望状态；
    Deployment 控制器将两个状态做比较，然后根据比较结果，确定是创建 Pod，还是删除已有的 Pod 
  - 类似 Deployment 这样的一个控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象的模板组成的。
  - Deployment -> ReplicaSet -> Pod
  - `kubectl scale deployment nginx-deployment --replicas=4`
    `kubectl get deployments`
    Deployment 的四个状态字段
      1. DESIRED：用户期望的 Pod 副本个数（spec.replicas 的值）；
      2. CURRENT：当前处于 Running 状态的 Pod 的个数；
      3. UP-TO-DATE：当前处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致；
      4. AVAILABLE：当前已经可用的 Pod 的个数，即：既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数。
    `kubectl rollout status deployment/nginx-deployment`
    `kubectl get rs` //replica set
    ReplicaSet 的 DESIRED、CURRENT 和 READY 字段的含义，和 Deployment 中是一致的。所以，相比之下，Deployment 只是在 ReplicaSet 的基础上，添加了 UP-TO-DATE 这个跟版本有关的状态字段。
    `kubectl edit deployment/nginx-deployment`: 把 API 对象的内容下载到了本地文件，让你修改完成后再提交上去。
    `kubectl describe deployment nginx-deployment`
    - 滚动更新：
      首先，当你修改了 Deployment 里的 Pod 定义之后，Deployment Controller 会使用这个修改后的 Pod 模板，创建一个新的 ReplicaSet（hash=1764197365），这个新的 ReplicaSet 的初始 Pod 副本数是：0。然后，在 Age=24 s 的位置，Deployment Controller 开始将这个新的 ReplicaSet 所控制的 Pod 副本数从 0 个变成 1 个，即：“水平扩展”出一个副本。紧接着，在 Age=22 s 的位置，Deployment Controller 又将旧的 ReplicaSet（hash=3167673210）所控制的旧 Pod 副本数减少一个，即：“水平收缩”成两个副本。
      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: nginx-deployment
        labels:
          app: nginx
      spec:
      ...
        strategy:
          type: RollingUpdate
          rollingUpdate:
            maxSurge: 1 #除了 DESIRED 数量之外，在一次“滚动”中，Deployment 控制器还可以创建多少个新 Pod 也可以用百分比表示 e.g. maxSurge: 50%
            maxUnavailable: 1 #在一次“滚动”中，Deployment 控制器可以删除多少个旧 Pod 也可以用百分比表示
      ```
    - `kubectl set image deployment/nginx-deployment nginx=nginx:1.91` 可以直接修改使用镜像
    - 回滚：`kubectl rollout undo deployment/nginx-deployment`
        `kubectl rollout history deployment/nginx-deployment`
        `kubectl rollout history deployment/nginx-deployment --revision=2`
        `kubectl rollout undo deployment/nginx-deployment --to-revision=2`
    - 暂停deployment 同时改变deployment, 这样最后只生成一个新的ReplicaSet对象
      `kubectl rollout pause deployment/nginx-deployment`
      修改后 `kubectl rollout resume deployment/nginx-deployment`.避免多次修改生成多个replicaset.
  - Service 被访问的两种方式
    1. Virtual IP (VIP): 当访问service 的IP地址，service会把请求转发到Service代理的某一个Pod上面
    2. DNS方式，只要访问 my-svc.my-namespace.svc.cluster.local这一条DNS记录，就可以访问到名叫my-svc的Service所代理的某一个pod。
      - 处理方式1：访问这一条dns时，解析到的其实是service的VIP
      - 处理方式2：headless service. 访问这一条记录时，直接解析的是service代理的某一个pod的IP地址。
  - Kubenetes 对“有状态应用”的支持 - StatefulSet
    - 先定义一个headless service:
    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      ports:
      - port: 80
        name: web
      clusterIP: None # 代表这个service 没有VIP，为headless service
      selector:
        app: nginx
    ```
    此headless service 的dns为： <pod-name>.<svc-name>.<namespace>.svc.cluster.local
    再创建一个StatefulSet
    ```
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: web
    spec:
      serviceName: "nginx" #和nginx-deployment 的唯一区别
      replicas: 2
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:1.9.1
            ports:
            - containerPort: 80
              name: web
    ```
    然后：
    `kubectl get service nginx`
    `kubectl get statefulset web`
    `kubectl get pods -w -l app=nginx`
  StatefulSet 给它所管理的所有 Pod 的名字，进行了编号，编号规则是：<statefulset name>-<index>
  也就是会创建web-0 and web-1
  `kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh `
  这个命令可以创建一个一次性的pod (--rm) 但是发现退出时，这个pod暂停运行，进入Error state. 并没有马上被删除
  然后在这个pod里面可以用`nslookup web-0.nginx` 然后发现可以解析正确的pod ip address. 就算删除这两个Pod 新的Pod被创建，ip地址也能够通过这个名字被解析到。

- Stateful Set 的存储状态
  - 定义PVC（persistent volume claim）
  ```
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pv-claim
  spec:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  ```
  - 声明使用PVC
  ```
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: web
  spec:
    serviceName: "nginx"
    replicas: 2
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:1.9.1
          ports:
          - containerPort: 80
            name: web
          volumeMounts:
          - name: www
            mountPath: /usr/share/nginx/html
    volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
    ```
    凡是被这个 StatefulSet 管理的 Pod，都会声明一个对应的 PVC；而这个 PVC 的定义，就来自于 volumeClaimTemplates 这个模板字段。更重要的是，这个 PVC 的名字，会被分配一个与这个 Pod 完全一致的编号。

    PVC 其实就是一种特殊的 Volume。只不过一个 PVC 具体是什么类型的 Volume，要在跟某个 PV 绑定之后才知道。关于 PV、PVC 更详细的知识，我会在容器存储部分做进一步解读。当然，PVC 与 PV 的绑定得以实现的前提是，运维人员已经在系统里创建好了符合条件的 PV（比如，我们在前面用到的 pv-volume）；或者，你的 Kubernetes 集群运行在公有云上，这样 Kubernetes 就会通过 Dynamic Provisioning 的方式，自动为你创建与 PVC 匹配的 PV。

    `for i in 0 1; do kubectl exec web-$i -- sh -c 'echo hello $(hostname) > /usr/share/nginx/html/index.html'; done` 在Pod的volume目录里写入一个文件 - 此处无法测试，本地没有PV

    *  如果删除这两个pod， 新的pod创建后，会自动对应到这两个volume
    在这个新的 Pod 对象的定义里，它声明使用的 PVC 的名字，还是叫作：www-web-0。这个 PVC 的定义，还是来自于 PVC 模板（volumeClaimTemplates），这是 StatefulSet 创建 Pod 的标准流程。所以，在这个新的 web-0 Pod 被创建出来之后，Kubernetes 为它查找名叫 www-web-0 的 PVC 时，就会直接找到旧 Pod 遗留下来的同名的 PVC，进而找到跟这个 PVC 绑定在一起的 PV。这样，新的 Pod 就可以挂载到旧 Pod 对应的那个 Volume，并且获取到保存在 Volume 里的数据。
  - 一个虽然复杂但是很好用来理解stateful set的实例：https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/#statefulset


    - patch rolling update: `kubectl patch statefulset mysql -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}` 表示当pod魔板发生变化时，只有序号大于或者等于2的pod会被更新。
  
- DaemonSet 
  - 主要作用，是让你在 Kubernetes 集群里，运行一个 Daemon Pod。 所以，这个 Pod 有如下三个特征：这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；每个节点上只有一个这样的 Pod 实例；当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。
  - 使用例子： 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。
  - 更重要的是，跟其他编排对象不一样，DaemonSet 开始运行的时机，很多时候比整个 Kubernetes 集群出现的时机都要早。
  - ```

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
  ```
  DaemonSet 跟 Deployment 其实非常相似，只不过是没有 replicas 字段；它也使用 selector 选择管理所有携带了 name=fluentd-elasticsearch 标签的 Pod。
  - 在指定Node上创建新的Pod
    可以用NodeSelector,但多用一个更新，功能更完善的字段nodeAffinity来代替。e.g.
    ```

    apiVersion: v1
    kind: Pod
    metadata:
      name: with-node-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: #这个 nodeAffinity 必须在每次调度的时候予以考虑。同时，这也意味着你可以设置在某些情况下不考虑这个 nodeAffinity；
            nodeSelectorTerms:
            - matchExpressions:
              - key: metadata.name
                operator: In
                values:
                - node-geektime
    ```
    你应该注意到 nodeAffinity 的定义，可以支持更加丰富的语法，比如 operator: In（即：部分匹配；如果你定义 operator: Equal，就是完全匹配），这也正是 nodeAffinity 会取代 nodeSelector 的原因之一。
  
  我们的 DaemonSet Controller 会在创建 Pod 的时候，自动在这个 Pod 的 API 对象里，加上这样一个 nodeAffinity 定义。其中，需要绑定的节点名字，正是当前正在遍历的这个 Node。

  DaemonSet 还会给这个 Pod 自动加上另外一个与调度相关的字段，叫作 tolerations。这个字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint）。而 DaemonSet 自动加上的 tolerations 字段，格式如下所示：
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: with-toleration
  spec:
    tolerations:
    - key: node.kubernetes.io/unschedulable
      operator: Exists
      effect: NoSchedule
  ```
  这个 Toleration 的含义是：“容忍”所有被标记为 unschedulable“污点”的 Node；“容忍”的效果是允许调度。
  而在正常情况下，被标记了 unschedulable“污点”的 Node，是不会有任何 Pod 被调度上去的（effect: NoSchedule）。可是，DaemonSet 自动地给被管理的 Pod 加上了这个特殊的 Toleration，就使得这些 Pod 可以忽略这个限制，继而保证每个节点上都会被调度一个 Pod。当然，如果这个节点有故障的话，这个 Pod 可能会启动失败，而 DaemonSet 则会始终尝试下去，直到 Pod 启动成功。
  假如当前 DaemonSet 管理的，是一个网络插件的 Agent Pod，那么你就必须在这个 DaemonSet 的 YAML 文件里，给它的 Pod 模板加上一个能够“容忍”node.kubernetes.io/network-unavailable“污点”的 Toleration。
  ```
  ...
  template:
      metadata:
        labels:
          name: network-plugin-agent
      spec:
        tolerations:
        - key: node.kubernetes.io/network-unavailable
          operator: Exists
          effect: NoSchedule
  ```
  DaemonSet 其实是一个非常简单的控制器。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理 Pod 的情况，来决定是否要创建或者删除一个 Pod。只不过，在创建每个 Pod 的时候，DaemonSet 会自动给这个 Pod 加上一个 nodeAffinity，从而保证这个 Pod 只会在指定节点上启动。同时，它还会自动给这个 Pod 加上一个 Toleration，从而忽略节点的 unschedulable“污点”。当然，你也可以在 Pod 模板里加上更多种类的 Toleration，从而利用 DaemonSet 达到自己的目的。
  - 在 DaemonSet 上，我们一般都应该加上 resources 字段，来限制它的 CPU 和内存使用，防止它占用过多的宿主机资源。
  - `kubectl get ds -n kube-system fluentd-elasticsearch`查看集群里DaemonSet对象
  - (Kubernetes 里比较长的 API 对象都有短名字，比如 DaemonSet 对应的是 ds，Deployment 对应的是 deploy。)
  - `kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record -n=kube-system` 改变容器版本 (由于这一次我在升级命令后面加上了–record 参数，所以这次升级使用到的指令就会自动出现在 DaemonSet 的 rollout history 里面)
  - `kubectl rollout history daemonset fluentd-elasticsearch -n kube-system` 查看rollout history
  - `kubectl rollout status ds/fluentd-elasticsearch -n kube-system` 查看rollout status
  - Kubernetes v1.7 之后添加了一个 API 对象，名叫 ControllerRevision，专门用来记录某种 Controller 对象的版本。比如，你可以通过如下命令查看 fluentd-elasticsearch 对应的 ControllerRevision： `kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch` 
  - `kubectl describe controllerrevision fluentd-elasticsearch-64dc6799c9 -n kube-system`
  - `kubectl rollout undo daemonset fluentd-elasticsearch --to-revision=1 -n kube-systemdaemonset.extensions/fluentd-elasticsearch rolled back`
  - 在 Kubernetes 项目里，ControllerRevision 其实是一个通用的版本管理对象。这样，Kubernetes 项目就巧妙地避免了每种控制器都要维护一套冗余的代码和逻辑的问题。
- Job and CronJob
  - Job.
  ```
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: pi
  spec:
    template:
      spec:
        containers:
        - name: pi
          image: resouer/ubuntu-bc 
          command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
        restartPolicy: Never
    backoffLimit: 4
  ```
    - `kubectl describe jobs/pi`
    - 这个 Job 对象在创建后，它的 Pod 模板，被自动加上了一个 controller-uid=< 一个随机字符串 > 这样的 Label。而这个 Job 对象本身，则被自动加上了这个 Label 对应的 Selector，从而 保证了 Job 与它所管理的 Pod 之间的匹配关系。
    - 我们需要在 Pod 模板中定义 restartPolicy=Never 的原因：离线计算的 Pod 永远都不应该被重启，否则它们会再重新计算一遍。 restartPolicy 在 Job 对象里只允许被设置为 Never 和 OnFailure；而在 Deployment 对象里，restartPolicy 则只允许被设置为 Always。
    - 如果这个离线作业失败了要怎么办？比如，我们在这个例子中定义了 restartPolicy=Never，那么离线作业失败后 Job Controller 就会不断地尝试创建一个新 Pod. 我们就在 Job 对象的 spec.backoffLimit 字段里定义了重试次数为 4（即，backoffLimit=4），而这个字段的默认值是 6。
    - 需要注意的是，Job Controller 重新创建 Pod 的间隔是呈指数增加的，即下一次重新创建 Pod 的动作会分别发生在 10 s、20 s、40 s …后。
    - 如果你定义的 restartPolicy=OnFailure，那么离线作业失败后，Job Controller 就不会去尝试创建新的 Pod。但是，它会不断地尝试重启 Pod 里的容器。
    - Job 的 API 对象里，有一个 spec.activeDeadlineSeconds 字段可以设置最长运行时间. 一旦运行超过了 100 s，这个 Job 的所有 Pod 都会被终止。并且，你可以在 Pod 的状态里看到终止的原因是 reason: DeadlineExceeded。
    - Job 对象中，负责并行控制的参数有两个：spec.parallelism，它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行；spec.completions，它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成数。
    - 三种常用使用Job对象的方法
      1. 外部管理器+job模板
      ```
      apiVersion: batch/v1
      kind: Job
      metadata:
        name: process-item-$ITEM
        labels:
          jobgroup: jobexample
      spec:
        template:
          metadata:
            name: jobexample
            labels:
              jobgroup: jobexample
          spec:
            containers:
            - name: c
              image: busybox
              command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
            restartPolicy: Never
      ```
      创建 Job 时，替换掉 $ITEM 这样的变量；所有来自于同一个模板的 Job，都有一个 jobgroup: jobexample 标签，也就是说这一组 Job 使用这样一个相同的标识。
      2. 拥有固定任务书目的并行Job
      ```
      apiVersion: batch/v1
      kind: Job
      metadata:
        name: job-wq-1
      spec:
        completions: 8
        parallelism: 2
        template:
          metadata:
            name: job-wq-1
          spec:
            containers:
            - name: c
              image: myrepo/job-wq-1
              env:
              - name: BROKER_URL
                value: amqp://guest:guest@rabbitmq-service:5672
              - name: QUEUE
                value: job1
            restartPolicy: OnFailure
      ```
      总共会有 8 个任务会被逐一放入工作队列里（你可以运行一个外部小程序作为生产者，来提交任务）。在这个实例中，我选择充当工作队列的是一个运行在 Kubernetes 里的 RabbitMQ。所以，我们需要在 Pod 模板里定义 BROKER_URL，来作为消费者。所以，一旦你用 kubectl create 创建了这个 Job，它就会以并发度为 2 的方式，每两个 Pod 一组，创建出 8 个 Pod。每个 Pod 都会去连接 BROKER_URL，从 RabbitMQ 里读取任务，然后各自进行处理。
      3. 指定并行度（parallelism）, 但不设置固定的completions的值
   - Cronjob 为定时任务
      ```
      apiVersion: batch/v1beta1
      kind: CronJob
      metadata:
        name: hello
      spec:
        schedule: "*/1 * * * *"
        jobTemplate:
          spec:
            template:
              spec:
                containers:
                - name: hello
                  image: busybox
                  args:
                  - /bin/sh
                  - -c
                  - date; echo Hello from the Kubernetes cluster
                restartPolicy: OnFailure
      ```
      CronJob 是一个 Job 对象的控制器（Controller）
      这个 Cron 表达式里 */1 中的 * 表示从 0 开始，/ 表示“每”，1 表示偏移量。所以，它的意思就是：从 0 开始，每 1 个时间单位执行一次。
      Cron 表达式中的五个部分分别代表：分钟、小时、日、月、星期。
      上面这句 Cron 表达式的意思是：从当前开始，每分钟执行一次。
      由于定时任务的特殊性，很可能某个 Job 还没有执行完，另外一个新 Job 就产生了。这时候，你可以通过 spec.concurrencyPolicy 字段来定义具体的处理策略。比如：
      1. concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在
      2. concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过；
      3. concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job。
      而如果某一次 Job 创建失败，这次创建就会被标记为“miss”。当在指定的时间窗口内，miss 的数目达到 100 时，那么 CronJob 会停止再创建这个 Job。这个时间窗口，可以由 spec.startingDeadlineSeconds 字段指定。比如 startingDeadlineSeconds=200，意味着在过去 200 s 里，如果 miss 的数目达到了 100 次，那么这个 Job 就不会被创建执行了。