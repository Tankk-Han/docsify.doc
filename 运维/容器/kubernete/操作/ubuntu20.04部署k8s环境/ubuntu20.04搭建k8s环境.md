# Ubuntu20.04搭建k8s环境

## 前置操作 （所有节点都要操作）
- 关闭swap

    >swap，这个当内存不足时，linux会自动使用swap，将部分内存数据存放到磁盘中，这个这样会使性能下降，为了性能考虑推荐关掉

    - 检查交换分区是否存在：

    ```
    cat /proc/swaps
    ```
    - 如果存在，运行以下命令

    ```
    sudo swapoff -a
    ```
    - 注释`swapfile`
    ```
    sudo vim /etc/fstab
    # 注释掉 swapfile 所在行
    #/swapfile  none  swap  sw  0  0
    ```
- 安装docker
    - 安装
    ```
    sudo apt update && sudo apt install -y docker.io
    ```
    - 配置docker守护程序，换源
    
        [官网文档](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#docker)

    ```
    cat <<EOF | sudo tee /etc/docker/daemon.json
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "registry-mirrors": ["https://puop2od6.mirror.aliyuncs.com"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
    EOF    
    ```
    - 去掉sudo权限

      - step1: 创建docker用户组
        ```
        sudo groupadd docker
        ```
      - step2: 当前用户组增加到docker用户组中
        ```
        sudo usermod -aG docker $USER
        ```
      - step3: 注销并重新登录，以便重新评估组成员关系
        ```
        newgrp docker
        ```
    - 重启

    ```
    sudo systemctl restart docker
    ```

- 安装k8s组件

    参考资料:

    [官方文档: 安装 kubeadm](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm)

    [k8s 国内源安装](https://gist.github.com/islishude/231659cec0305ace090b933ce851994a)

```
# 安装 software-properties-common
sudo apt install -y software-properties-common

# 添加并信任APT证书
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

# 添加源地址
sudo add-apt-repository "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"

# 更新源并安装最新版 kubenetes
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
```
- **证书问题**的特殊解决（本地操作）

  `kubernetes`证书的默认有效期是一年、在实际生产环境中，当证书过期时会造成宕机等风险，因此使用了非正当手法将`kubeadm` 的证书过期时间做了修改

  在此操作之前请先安装`go` ，稳定版本即可

  1. 在git上下载`kubernetes`源码

  ``` bash
  git clone https://github.com/kubernetes/kubernetes.git
  ```

  2. 切换到所需版本分支（以1.22为例）

  ```bash
  git checkout release-1.22
  ```

  3. 修改`cmd/kubeadm/app/constants/constants.go`

  ```go
  // 源码
  CertificateValidity = time.Hour * 24 * 365
  // 修改后
  // 我们将证书有效时间修改为100年
  CertificateValidity = time.Hour * 24 * 365 * 100
  ```

  4. 接下来是编译环节

     - linux系统 

     1. 执行

     ```bash
     make WHAT=cmd/kubeadm GOFLAGS=-v 
     ```

     2. 在`_output`下找到`kubeadm` 文件即为我们所需文件

     - mac系统

     1. 切换目录

     ```bash
     cd cmd/kubeadm 
     ```

     2. 执行交叉编译（因为服务器是linux/amd64 本人电脑是arm架构）

     ```go
     GOOS=linux GOARCH=amd64 go build
     ```

     3. 将在当前文件夹下生成一个`kubeadm`二进制文件

  5. 我们将之前用`apt`下载的`kubeadm` 用此文件替换即可

  ​		

  ```bash
  # 本机
  scp kubeadm root@xxx.xxx.xxx.xxx:[路径]
  # 服务器主机
  chmod -x kubeadm
  cp kubeadm /usr/bin/kubeadm
  ```

  

- 验证是否安装成功

```bash
kubeadm version
kubectl version
kubelet --version
```

## 初始化控制平面节点（主节点操作）

- 使用 `kubeadm init <arg>`  命令：

```bash
sudo kubeadm init --apiserver-advertise-address 172.22.108.36 \
    --pod-network-cidr 10.244.0.0/16 \
    --image-repository gotok8s
```
这里的 `172.22.108.36` 为 **master** 的本地 IP。

> `--pod-network-cidr 10.244.0.0/16` 参数与后续 CNI 插件有关，这里以 `flannel` 为例，若后续部署其他类型的网络插件请更改此参数。[参考连接](https://yeasy.gitbook.io/docker_practice/setup/kubeadm)
>
> 必须部署一个基于 Pod 网络插件的 [容器网络接口](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#cni) (CNI)，以便 Pod 可以相互通信。 在安装网络之前，集群 DNS (CoreDNS) 将不会启动。
>
> 详见：[官方文档](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)

成功后显示以下信息：

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.22.108.36:6443 --token tokenstring... \
	--discovery-token-ca-cert-hash sha256:... 
```

- 根据提示，root 用户运行以下命令：

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

- 部署 CNI

运行以下命令，安装 flannel：

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

> 详见：[flannel](https://github.com/flannel-io/flannel)

如果安装不上可以修改镜像源地址，新建flannel.yaml文件

<details>
<summary>flannel.yaml</summary>
<br>
   <pre><code>
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath
  allowedHostPaths:
  - pathPrefix: "/etc/cni/net.d"
  - pathPrefix: "/etc/kube-flannel"
  - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN', 'NET_RAW']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['psp.flannel.unprivileged']
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: flannelcni/flannel:v0.15.0-rc1
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: flannelcni/flannel:v0.15.0-rc1
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
   </code></pre>
</details>

安装

```shell
kubectl apply -f flannel.yaml
```

- 检查状态

```bash
kubectl get nodes# Status 应为 Readykubectl get pods --all-namespaces# Status 应为 running
```

## 节点操作（主节点之外的节点）

- 加入节点

在 **node** 节点使用以下命令：

> [官方文档：加入节点](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)

```bash
kubeadm join 172.22.108.36:6443 --token <token string> \    --discovery-token-ca-cert-hash sha256:<sha256 string>
```

成功后显示如下信息：

```bash
This node has joined the cluster:* Certificate signing request was sent to apiserver and a response was received.* The Kubelet was informed of the new secure connection details.Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

此时在 **master** 节点使用`kubectl get nodes`可以获取到 node 节点状态。

### token 

**以下命令在控制节点（master）上使用。**

- 列出 token：

```bash
kubeadm token list
```

- 默认情况下，令牌会在24小时后过期。创建新令牌：

```bash
kubeadm token create
```

- 生成`--discovery-token-ca-cert-hash` 所需的值：

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \   openssl dgst -sha256 -hex | sed 's/^.* //'
```

### 删除节点

> [官方文档](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#%E5%88%A0%E9%99%A4%E8%8A%82%E7%82%B9)

**以下命令在 node 节点使用**。

在删除节点之前，重置 `kubeadm` 安装的状态：

```bash
kubeadm reset
```

删除节点：

```bash
kubectl delete node <node name>
```

重置过程不会重置或清除 iptables 规则或 IPVS 表。手动操作：

```bash
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -Xipvsadm -C
```

# 测试集群

**master** 节点执行以下命令：

```bash
kubectl create deployment nginx --image=nginxkubectl expose deployment nginx --port=80 --type=NodePortkubectl get pod,svc
```

显示信息如下：
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGEservice/kubernetes
```bash
service/nginx        NodePort    10.106.10.30   <none>        80:32579/TCP   18s
```

此时访问：`<node 节点 ip>:32579` 即可访问到 NGINX 服务。

# 证书轮换

https://kubernetes.io/docs/tasks/tls/certificate-rotation/

https://kubernetes.feisky.xyz/practice/certificate-rotation

https://kubernetes.io/zh/docs/tasks/tls/manual-rotation-of-ca-certificates/