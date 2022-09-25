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
        最后使用`--ignore-preflight-errors=Swap`


- rerun `sudo kubeadm init` will give error of `FileAvailablexxxxxxx`, 可以用`sudo kubeadm reset` 再重试。
