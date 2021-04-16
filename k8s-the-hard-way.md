# Kubernetes the Hard Way


## Laptop Compute Resources
Now following [Kelsey Hightower's](https://github.com/kelseyhightower/kubernetes-the-hard-way) instructions for installing Kubernetes piece-by-piece. 

Hightower's instructions are written for the Google Cloud.  I am adapting them for hardware.  I have six laptops with Ubuntu setup. Each has the following in /etc/hosts: 

```
192.168.1.50 control1
192.168.1.51 control2
192.168.1.52 control3
192.168.1.53 worker1
192.168.1.54 worker2
192.168.1.55 worker3
```

## Install Client Tools 
Need cfssl, cfssljson, and kubectl.
Rather than downloading cfssl from Hightower's google storage, I installed it as per the [Cloudflare cfssl site](https://github.com/cloudflare/cfssl): 

#### CFSSL
```
sudo apt  install golang-go
# the next statement seemed to get me an old version of cfssl. 
go get -u github.com/cloudflare/cfssl/cmd/cfssl

git clone https://github.com/cloudflare/cfssl.git
cd cfssl
sudo apt install make
make
./cfssl version
./cfssljson version
sudo cp cfssl /usr/bin/cfssl
sudo cp cfssljson /usr/bin/cfssljson
```
This resulted in version 1.5.0 of cfssl and cfssljson. 


#### KUBECTL
Followed the [Kubernetes kubectl docs](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/): 
```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
kubectl version
```

## [Provisioning a CA and Generating TLS Certificates](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md)

Followed the instruction to generate a Certificate Authority. 
```
ca-key.pem
ca.pem
```

Followed the instruction to generate Client and Server certificates. 
```
admin-key.pem
admin.pem
```

Create the Kubelet Client Certificates

Created this script. Couldn't use the one documented since it assumes gcloud. 
```
IP="192.168.1."
LAST=53
for instance in worker1 worker2 worker3; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

LAST=$(($LAST+1))

IP_ADDR=$IP$LAST

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${IP_ADDR} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

Ended up with: 
```
-rw-r--r-- 1 root root 1106 Apr  3 22:00 worker1.csr
-rw-r--r-- 1 root root  243 Apr  3 22:00 worker1-csr.json
-rw------- 1 root root 1675 Apr  3 22:00 worker1-key.pem
-rw-r--r-- 1 root root 1484 Apr  3 22:00 worker1.pem
-rw-r--r-- 1 root root 1106 Apr  3 22:00 worker2.csr
-rw-r--r-- 1 root root  243 Apr  3 22:00 worker2-csr.json
-rw------- 1 root root 1679 Apr  3 22:00 worker2-key.pem
-rw-r--r-- 1 root root 1484 Apr  3 22:00 worker2.pem
-rw-r--r-- 1 root root 1106 Apr  3 22:00 worker3.csr
-rw-r--r-- 1 root root  243 Apr  3 22:00 worker3-csr.json
-rw------- 1 root root 1679 Apr  3 22:00 worker3-key.pem
-rw-r--r-- 1 root root 1484 Apr  3 22:00 worker3.pem
```
Unsure if they are correct.  What's that extra workerX.csr for? 

### Controller Manager Client Certificate
Generated it just like the instructions show. 


### Kube Proxy Client Certificate
Generated it just like the instructions show. 


### Scheduler Client Certificate
Generated it just like the instructions show. 


### Kubernetes API Server Certificate
Modified the script like this to account for my network.  
```
{

KUBERNETES_PUBLIC_ADDRESS=69.71.44.73

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=192.168.1.50,192.168.1.51,192.168.1.52,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

### Service Account Key Pair
Generated it just like the instructions show. 


### Distribute the Client and Server Certificates
```
cd /root/04-certificate-authority
for instance in worker1 worker2 worker3; do scp ca.pem ${instance}-key.pem ${instance}.pem joe@${instance}:~/; done
for instance in control1 control2 control3; do scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem joe@${instance}:~/; done
```


## [Generating Kubernetes Configuration Files for Authentication](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md)

Since I don't have a load balancer like GCP does, I will use the address of the 1st control plane node. Yes, so the two extra control plane nodes won't get used much. 
I could install haproxy and load balance across them, but in my configuration that's a single point of failure also. 

```
KUBERNETES_PUBLIC_ADDRESS="192.168.1.50"
```
and then I ran the scripts as-is, but replaced worker-0 with worker1, etc. 

```
for instance in worker1 worker2 worker3; do scp ${instance}.kubeconfig kube-proxy.kubeconfig joe@${instance}:~/; done
for instance in control1 control2 control3; do scp ca.pem admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig  joe@${instance}:~/; done
```


## [Generating the Data Encryption Config and Key](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/06-data-encryption-keys.md)

```
for instance in control1 control2 control3; do scp encryption-config.yaml   joe@${instance}:~/; done
```


## [Bootstrapping the etcd Cluster](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/07-bootstrapping-etcd.md)

Noticed that there's a newer version of ETCD: 3.4.15. Will use that instead of 3.4.10. 

```
wget "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd/
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
INTERNAL_IP=192.168.1.50
ETCD_NAME=control1
```

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster control1=https://192.168.1.50:2380,control2=https://192.168.1.51:2380,control3=https://192.168.1.52:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

Check that etcd is working. I ran this on each of the control nodes with the localhost address and again with their 192.168.1.50, 51, 52 address. 
Response matches expected from the instruction. 
```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```


## [Bootstrapping the Kubernetes Control Plane](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md)

Tried out `tmux`.  ctrl-b followed by colon gets you to the prompt.  Use `set synchronize-panes` to have the same commands executed in each pane. Created 3 panes and ssh'd to control1, control2, control3 in separate panes. Then, turned on synchronize-panes

```
sudo mkdir -p /etc/kubernetes/config

wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubectl"

{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}

sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Then modified the /etc/systemd/system/kube-apiserver.service to put in the correct IPs for each control node.  
# Almost changed the IP for service-cluster-ip-range, but realized that's a virtual address managed by kube-proxy, 
# so it should probably stay at the 10.32.0.0 that it was set at. 

sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF

# The exercise is planning to use a Google Network Load Balancer. Here I don't have one, but
# I will configure nginx to proxy the https health check service to http anyway. 

sudo apt-get update
sudo apt-get install -y nginx

cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF

{
  sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
}

sudo systemctl restart nginx
sudo systemctl enable nginx

```


Verification

```
$ kubectl get componentstatuses --kubeconfig admin.kubeconfig
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}

$ curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
HTTP/1.1 200 OK...
```

RBAC for Kubelet Authorization

On only one of the control nodes: 
```
$ cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
> apiVersion: rbac.authorization.k8s.io/v1beta1
> kind: ClusterRole
> metadata:
>   annotations:
>     rbac.authorization.kubernetes.io/autoupdate: "true"
>   labels:
>     kubernetes.io/bootstrapping: rbac-defaults
>   name: system:kube-apiserver-to-kubelet
> rules:
>   - apiGroups:
>       - ""
>     resources:
>       - nodes/proxy
>       - nodes/stats
>       - nodes/log
>       - nodes/spec
>       - nodes/metrics
>     verbs:
>       - "*"
> EOF
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created


$ cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
> apiVersion: rbac.authorization.k8s.io/v1beta1
> kind: ClusterRoleBinding
> metadata:
>   name: system:kube-apiserver
>   namespace: ""
> roleRef:
>   apiGroup: rbac.authorization.k8s.io
>   kind: ClusterRole
>   name: system:kube-apiserver-to-kubelet
> subjects:
>   - apiGroup: rbac.authorization.k8s.io
>     kind: User
>     name: kubernetes
> EOF
clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created

```

Frontend Load Balancer
Skipping this.  I could setup HAProxy like I did in the [kubeadm exercise](project-notes.md), but I'll skip it here. 


Verification
Although I don't have a Frontend Load Balancer, I can curl the endpoint and see the version.  This works 
against all three control nodes. 
```
# curl --cacert /var/lib/kubernetes/ca.pem https://192.168.1.50:6443/version
{
  "major": "1",
  "minor": "18",
  "gitVersion": "v1.18.6",
  "gitCommit": "dff82dc0de47299ab66c83c626e08b245ab19037",
  "gitTreeState": "clean",
  "buildDate": "2020-07-15T16:51:04Z",
  "goVersion": "go1.13.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

I was able to copy the admin.kubeconfig file `scp joe@control1:/home/joe/admin.kubeconfig ~/junk/` to a non-control node, 
modify the IP address in it to point to 192.168.1.50 (control1), and connect. 
At this point, the only thing that is visible is a kubernetes service. 
```
$ kubectl --kubeconfig=admin.kubeconfig get all -A
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP   52m
```


## [Bootstrapping the Kubernetes Worker Nodes](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md)

```
{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}

$ sudo swapon --show
NAME      TYPE SIZE USED PRIO
/swap.img file 3.7G   0B   -2
$ sudo swapoff -a
$ sudo swapon --show

# To keep swap permanently off (after reboot), comment out (use #) in front of this line in /etc/fstab
#/swap.img      none    swap    sw      0       0
# A `sudo shutdown -r 0` will reboot the box to test this. 


wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.18.0/crictl-v1.18.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc91/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz \
  https://github.com/containerd/containerd/releases/download/v1.3.6/containerd-1.3.6-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubelet


sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes

{
  mkdir containerd
  tar -xvf crictl-v1.18.0-linux-amd64.tar.gz
  tar -xvf containerd-1.3.6-linux-amd64.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin/
  sudo mv runc.amd64 runc
  chmod +x crictl kubectl kube-proxy kubelet runc 
  sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/
}

# It is asking to Pod CIDR range from Google.  Is that my 192.168.1.0/24? 
export POD_CIDR=192.168.1.0/24

cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF

cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF

sudo mkdir -p /etc/containerd/

cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF

cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF


{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}

cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF


cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig


cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF


cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}

# Verification

$ kubectl --kubeconfig=admin.kubeconfig get nodes
NAME      STATUS   ROLES    AGE   VERSION
worker1   Ready    <none>   58s   v1.18.6
worker2   Ready    <none>   58s   v1.18.6
worker3   Ready    <none>   58s   v1.18.6

```


## [Configuring kubectl for Remote Access](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/10-configuring-kubectl.md)

I had built the certificates on control1 in /root/04-certificate-authority

```
root@control1:~/04-certificate-authority# cd /root/04-certificate-authority/
root@control1:~/04-certificate-authority# export KUBERNETES_PUBLIC_ADDRESS=192.168.1.50
root@control1:~/04-certificate-authority# kubectl config set-cluster kubernetes-the-hard-way \
>     --certificate-authority=ca.pem \
>     --embed-certs=true \
>     --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443
Cluster "kubernetes-the-hard-way" set.
root@control1:~/04-certificate-authority# kubectl config set-credentials admin \
>     --client-certificate=admin.pem \
>     --client-key=admin-key.pem
User "admin" set.
root@control1:~/04-certificate-authority# kubectl config set-context kubernetes-the-hard-way \
>     --cluster=kubernetes-the-hard-way \
>     --user=admin
Context "kubernetes-the-hard-way" created.
root@control1:~/04-certificate-authority# kubectl config use-context kubernetes-the-hard-way
Switched to context "kubernetes-the-hard-way".



root@control1:~# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
root@control1:~# kubectl get nodes -owide
NAME      STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
worker1   Ready    <none>   24m   v1.18.6   192.168.1.53   <none>        Ubuntu 20.10   5.8.0-48-generic   containerd://1.3.6
worker2   Ready    <none>   24m   v1.18.6   192.168.1.54   <none>        Ubuntu 20.10   5.8.0-49-generic   containerd://1.3.6
worker3   Ready    <none>   24m   v1.18.6   192.168.1.55   <none>        Ubuntu 20.10   5.8.0-48-generic   containerd://1.3.6
```

## [Provisioning Pod Network Routes](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/11-pod-network-routes.md)

I don't think I need to do this. 
This appears to create route (permissions) in GCP between the nodes, but since I'm on a local hardware network
and no firewalls are blocking routes... I don't think I need to do anything.  


## [Deploying the DNS Cluster Add-on: CoreDNS](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/12-dns-addon.md)

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.7.0.yaml

# But, I am getting ImagePullBackOff from the above.... 
# kubectl get pods -A
NAMESPACE     NAME                       READY   STATUS             RESTARTS   AGE
kube-system   coredns-5677dc4cdb-nczc7   0/1     ImagePullBackOff   0          57s
kube-system   coredns-5677dc4cdb-nmmdp   0/1     ImagePullBackOff   0          57s

# By using `kubectl -n kube-system describe pod coredns-5677dc4cdb-nczc7` I can see the error
Failed to pull image "coredns/coredns:1.7.0": rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/coredns/coredns:1.7.0": failed to resolve reference "docker.io/coredns/coredns:1.7.0": failed to do request: Head https://registry-1.docker.io/v2/coredns/coredns/manifests/1.7.0: dial tcp: lookup registry-1.docker.io: Temporary failure in name resolution

# I am unable to ping that host: 
$ ping registry-1.docker.io
PING registry-1.docker.io (3.220.36.210) 56(84) bytes of data.
^C
--- registry-1.docker.io ping statistics ---
11 packets transmitted, 0 received, 100% packet loss, time 10224ms

I can `dig registry-1.docker.io` and get an ANSWER section: 
...
;; ANSWER SECTION:
registry-1.docker.io.	26	IN	A	107.23.149.57
...

But, I can't even ping the address it gave me: 
# ping 107.23.149.57
PING 107.23.149.57 (107.23.149.57) 56(84) bytes of data.
^C
--- 107.23.149.57 ping statistics ---
18 packets transmitted, 0 received, 100% packet loss, time 17395ms

But I can ping google: 
ping www.google.com
PING forcesafesearch.google.com (216.239.38.120) 56(84) bytes of data.
64 bytes from any-in-2678.1e100.net (216.239.38.120): icmp_seq=1 ttl=119 time=18.1 ms


So, in the netplan configuration, I only had 192.168.1.1 (my local router) as a nameserver. After I added
8.8.8.8 (Google's), and did a `sudo netplan apply`, then I was able to delete the deployment and re-apply it 
and the pods started happily. 
kubectl get pods -l k8s-app=kube-dns -n kube-system -owide
NAME                       READY   STATUS    RESTARTS   AGE     IP            NODE      NOMINATED NODE   READINESS GATES
coredns-5677dc4cdb-2s2w9   0/1     Running   2          5m14s   192.168.1.2   worker1   <none>           <none>
coredns-5677dc4cdb-bnz6z   0/1     Running   2          5m14s   192.168.1.3   worker3   <none>           <none>

```



