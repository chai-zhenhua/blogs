### 准备工作
```
master节点      k8s-node-1  10.206.0.2
node节点        k8s-node-2  10.206.0.4
node节点        k8s-node-3  10.206.0.7
```

### 规划pod网络
```
pod cidr : 192.168.0.0/16
tips:
    1. 考虑业务发展趋势预设集群规模
    2. 考虑网段是否和现有其他业务网段冲突
```

### 配置yum源
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 安装
```
master节点: yum -y install kubelet kubeadm kubectl docker
node节点: yum -y install kubelet kubeadm docker
```

### 初始化master节点
```
kubeadm  init --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers --pod-network-cidr 192.168.0.0/16
```

```
[root@k8s-node-1 ~]# kubectl  get pod -A
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-65c54cc984-7mld2             0/1     Pending   0          2m6s        # 可以describe 看下原因,是因为网络未就绪
kube-system   coredns-65c54cc984-ld6f6             0/1     Pending   0          2m6s
kube-system   etcd-k8s-node-1                      1/1     Running   0          2m20s
kube-system   kube-apiserver-k8s-node-1            1/1     Running   0          2m22s
kube-system   kube-controller-manager-k8s-node-1   1/1     Running   0          2m20s
kube-system   kube-proxy-7vwt5                     1/1     Running   0          2m6s
kube-system   kube-scheduler-k8s-node-1            1/1     Running   0          2m20s
[root@k8s-node-1 ~]#

```

### 安装网络插件flanel
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    # flannel网络保持和上面规划的pod-cidr一致
    net-conf.json: |
      {
        "Network": "192.168.0.0/16",
        "Backend": {
          "Type": "vxlan"
        }
      }

# pod ready
[root@k8s-node-1 ~]# kubectl get pod -A
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-65c54cc984-7mld2             1/1     Running   0          4m47s
kube-system   coredns-65c54cc984-ld6f6             1/1     Running   0          4m47s
kube-system   etcd-k8s-node-1                      1/1     Running   0          5m1s
kube-system   kube-apiserver-k8s-node-1            1/1     Running   0          5m3s
kube-system   kube-controller-manager-k8s-node-1   1/1     Running   0          5m1s
kube-system   kube-flannel-ds-w6x5w                1/1     Running   0          41s
kube-system   kube-proxy-7vwt5                     1/1     Running   0          4m47s
kube-system   kube-scheduler-k8s-node-1            1/1     Running   0          5m1s
```


### 加入node
```
kubeadm join 10.206.0.2:6443 --token w26thy.hr1rxlmna58viyme --discovery-token-ca-cert-hash sha256:530f0096881332c5dc20fc4c1d753d806814f826c90e1b406b6152bc7eeea8bd


[root@k8s-node-1 ~]# kubectl  get node
NAME         STATUS   ROLES                  AGE     VERSION
k8s-node-1   Ready    control-plane,master   13m     v1.23.5
k8s-node-2   Ready    <none>                 2m17s   v1.23.5
k8s-node-3   Ready    <none>                 54s     v1.23.5
```


### 安装ingress-control
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.3/deploy/static/provider/cloud/deploy.yaml

[root@k8s-node-1 ~]# kubectl  get pod -n ingress-nginx
NAME                                        READY   STATUS              RESTARTS   AGE
ingress-nginx-admission-create-nsq8z        0/1     ErrImagePull        0          69s
ingress-nginx-admission-patch-x9xww         0/1     ErrImagePull        0          69s
ingress-nginx-controller-69fbbf9f9c-r4tv8   0/1     ContainerCreating   0          69s

kubectl describe pod ingress-nginx-admission-create-nsq8z -n ingress-nginx
    Events:
      Type     Reason     Age                From               Message
      ----     ------     ----               ----               -------
      Normal   Scheduled  91s                default-scheduler  Successfully assigned ingress-nginx/ingress-nginx-admission-create-nsq8z to     k8s-node-2
      Normal   BackOff    33s (x2 over 75s)  kubelet            Back-off pulling image "k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1. 1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660"
      Warning  Failed     33s (x2 over 75s)  kubelet            Error: ImagePullBackOff
      Normal   Pulling    21s (x3 over 90s)  kubelet            Pulling image "k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.  1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660"
      Warning  Failed     6s (x3 over 75s)   kubelet            Failed to pull image "k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.   1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660": rpc error: code = Unknown desc = Get https://k8s.gcr.io/v2/:   net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
      Warning  Failed     6s (x3 over 75s)   kubelet            Error: ErrImagePull

tips:
    由于国内网络原因,官方的镜像无法拉取, 但是还是有很多其他方法可以用， 比如自定义， 用其他人拉取的镜像...
```