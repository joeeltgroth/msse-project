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



