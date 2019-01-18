## 参考下面博客完成大部分组件安装

[Centos7快速搭建Kubernetes 1.11.1单机集群](https://www.datayang.com/article/45)

可以完成到`dashboard`前的搭建

### 问题

1. 安装指定版本的docker

   `yum install docker-ce-17.09.0.ce -y`

2. `coredns`服务不停的`restart`，解决办法：

> kubernetes（k8s）DNS 服务反复重启解决：
>
> k8s.io/dns/pkg/dns/dns.go:150: Failed to list *v1.Service: Get https://10.96.0.1:443/api/v1/services?resourceVersion=0: dial tcp 10.96.0.1:443: getsockopt: no route to host
> 在使用 Minikube 部署 kubernetes 服务时，出现 Kube DNS 服务反复重启现象（错误如上），
>
> 这很可能是 iptables 规则乱了，我通过执行以下命令解决了，在此记录：
> --------------------- 
> ```shell
> systemctl stop kubelet
> systemctl stop docker
> iptables --flush
> iptables -tnat --flush
> systemctl start kubelet
> systemctl start docker
> ```
>
> 参考：https://blog.csdn.net/shida_csdn/article/details/80028905

## Worker节点安装

安装Worker节点步骤如下：就是[Centos7快速搭建Kubernetes 1.11.1单机集群](https://www.datayang.com/article/45)博客中的前面步骤。

1. 用swap分区

```
sudo swapoff -a
```

  永久禁用

```
sudo vi /etc/fstab
```

把/dev/mapper/centos-swap swap这行注释掉

  编写配置

```
vim /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1    
vm.swappiness=0
sysctl --system
```

2. 配置kubernetes yum源

```
vim  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
enable=1
cd /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum clean all
yum repolist
```

##### 安装

安装kubeadm, kubelet and kubectl

```
yum install docker-ce kubelet-1.11.1 kubeadm-1.11.1  kubectl-1.11.1 kubernetes-cni
systemctl enable docker
systemctl enable kubelet.service
systemctl start docker
systemctl start kubelet
```

3. 由于国内网络原因，kubernetes的镜像托管在google云上，无法直接下载，所以直接把把镜像搞下来有个技术大牛把gcr.io的镜像

每天同步到https://github.com/anjia0532/gcr.io_mirror这个站点，因此，如果需要用到gcr.io的镜像，可以执行如下的脚本进行镜像拉取

```
vim pullimages.sh
#!/bin/bash
images=(kube-proxy-amd64:v1.11.1 kube-scheduler-amd64:v1.11.1 kube-controller-manager-amd64:v1.11.1
kube-apiserver-amd64:v1.11.1 etcd-amd64:3.2.18 coredns:1.1.3 pause:3.1 )
for imageName in ${images[@]} ; do
docker pull anjia0532/google-containers.$imageName
docker tag anjia0532/google-containers.$imageName k8s.gcr.io/$imageName
docker rmi anjia0532/google-containers.$imageName
done
sh pullimages.sh
```

5. kubernetes集群不允许开启swap，所以我们需要忽略这个错误

```
vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
```

6. 执行在安装master时最后生成的一段代码，代码格式如下：

   ```shell
   kubeadm join 192.168.0.184:6443 --token zr21ox.if71afbyjpk0khq7 --discovery-token-ca-cert-hash sha256:296c9615eebd158adcfebd6710610b352da55152566ce87bf56559b0ddd596b1
   ```

### 获得`dashboard`文件并安装

1. 下载`dashboard.yaml`文件

   `wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.0/src/deploy/recommended/kubernetes-dashboard.yaml `

2. 修改文件

   ```
   kind: Service
   apiVersion: v1
   metadata:
     labels:
       k8s-app: kubernetes-dashboard
     name: kubernetes-dashboard
     namespace: kube-system
   spec:
     # 添加Service的type为NodePort
     type: NodePort
     ports:
       - port: 443
         targetPort: 8443
         # 添加映射到虚拟机的端口,k8s只支持30000以上的端口
         nodePort: 30001
     selector:
       k8s-app: kubernetes-dashboard
   ```

3. 安装

   ```
   kubectl apply -f kubernetes-dashboard.yaml
   ```

4. 获取token

   这里有一个简单的命令：

   ```
   kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token
   ```

   也可以自己手动查询：

   ```
   # 输入下面命令查询kube-system命名空间下的所有secret
   kubectl get secret -n kube-system
   
   # 然后获取token。只要是type为service-account-token的secret的token都可以使用。
   # 比如我们获取replicaset-controller-token-wsv4v的touken
   kubectl -n kube-system describe replicaset-controller-token-wsv4v
   ```

5. 访问dashboard

   通过node节点的ip，加刚刚我们设置的nodePort就可以访问了。

   ```
   https://<node-ip>:<node-port>
   ```

   **注意是https不是http哦**

   认证有两种方式：

   - 通过我们刚刚获取的token直接认证

   - 通过Kubeconfig文件认证
     只需要在kubeadm生成的admin.conf文件末尾加上刚刚获取的token就好了。

     ```
     - name: kubernetes-admin
       user:
         client-certificate-data: xxxxxxxx
         client-key-data: xxxxxx
         token: "在这里加上token"
     ```

## 创建新的`kubeadm join`令牌

`kubeadm token create --print-join-command`

## `kubectl`命令

1. `kubectl get pods -n kube-system`

2. `kubectl describe pod podName --namespace=kube-system` 其中的podName是你的pod名称

3. `kubectl describe node master` master是节点名称 可以通过`kubectl get nodes`得到。

   ```shell
   [root@nsop-k8s1 ~]# kubectl get nodes
   NAME        STATUS    ROLES     AGE       VERSION
   nsop-k8s1   Ready     master    2h        v1.11.1
   nsop-k8s2   Ready     <none>    2h        v1.11.1
   [root@nsop-k8s1 ~]# 
   ```



## Jenkins + Gitlab + Kubernetes 的自动化持续集成与部署

[Jenkins + Gitlab + Kubernetes 的自动化持续集成与部署](https://huanqiang.wang/2018/03/30/Jenkins-Gitlab-Kubernetes-%E7%9A%84%E8%87%AA%E5%8A%A8%E5%8C%96%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E9%83%A8%E7%BD%B2/)