


### First attempt at networking computers on a subnet via a separate switch and bridging via WiFi
Worked on setting up local network.  Following this: 
https://wiki.dd-wrt.com/wiki/index.php/Linking_Subnets_with_Static_Routes
Had a big struggle with this.  Though I was able to setup a 10.0.0.0 network, I wasn't able to get routing (or bridging?) to work.  I tried having my stand-alone 10.0.0.0 network, but tried to make one of the boxes WiFi wireless interfaces into a bridge, but got errors back perhaps indicating that wasn't allowed... not sure why.  Since my project is more about Kubernetes than networking, and the networking I care about is AWS, GCP, Azure, vSphere, etc, not home-style Linksys hardware, I decided to leave that behind... 


### Second attempt at networking using base 192.168.1.0 network on existing home network
Just allocate 192.168.1.x static addresses to the boxes by connecting an unmanaged swith into the main router and get on with the show of installing kubernetes. 


### The resulting network
![Physical network](/images/network.png)

Have been using this doc to understand how to configure networks on Ubuntu: 
https://ubuntu.com/server/docs/network-configuration


### My process for setting up a Latitude E5540 with Ubuntu linux

First, I downloaded [Ubuntu Server 20.10](https://ubuntu.com/download/server).  I got an .iso file.  
I used commands like these to burn a CVD disk becuase I couldn't get a flash drive to boot on the old hardware I have. 
```bash
hdiutil convert -format UDRW -o ubuntu-20.10-live-server-amd64.img ubuntu-20.10-live-server-amd64.iso 
diskutil list
diskutil umount /dev/disk2s1
sudo dd if=./ubuntu-20.10-live-server-amd64.img of=/dev/rdisk2 bs=1m
```
The above commands resulted in a bootable DVD. 

The reason I choose Ubuntu Server is that I wanted a lightweight server with no graphical user interface.  I looked into 
Photon OS Minimal, but got stuck and given that I use Ubuntu more often at work it seemed like the logical choice. 

To install Ubuntu, I started the Latitude E5540 and pressed F12 until the boot menu came up, and I choose
CD/DVD as the boot medium. I installed the Ubuntu server fully on the disk (these computers were prior wiped clean with dban.org's boot/nuke product). 

On most of the boxes, I let them come up using DHCP, then later changed their IP address to 
a static. 

I also installed an ssh server on each box. This allows me to connect from my MacBook, the computer
I normally use, and not have to go to the command line on each computer's individual keyboard. 

[Pictures of the installation process](installing-ubuntu.md)


### Setting IP Addresses
A tool called "netplan" is used on Ubuntu Linux (probably other distributions also), to configure 
networking on linux. 

The important file is /etc/netplan/00-installer-config.yaml. (note that the name can vary a bit, but it is the only file in that directory).  Below I am configuring with 
dhcp turned off, and 192.168.1.50 as the address. The gateway is the address of my 
Linksys router/switch. 

Below I am showing the nameservers, but I couldn't configure that section until I'd setup
a bind9 DNS server (which was then listening to DNS queries on 192.168.1.50)
```
network:
  ethernets:
    enp10s0:
      dhcp4: no
      addresses: [192.168.1.50/24]
      gateway4: 192.168.1.1
      nameservers:
        search: [joesmsse.com]
        addresses: [192.168.1.50, 192.168.1.1]
  version: 2
```
Useful Commands when setting the IP:  

`ip a` - show the network interfaces

`sudo lshw -class network` - get detailed network interface details. 

`sudo ethtool eth4` - changes ethernet card settings (auto-negotiation, port speed... ) 

`vi /etc/netplan/00-installer-config.yaml` - edit the netplan configuration to set static ip, gateway, and nameservers. 

`sudo netplan apply` - apply changes to the netplan

`cat /etc/os-release` - what version of Linux you are running


### Installing a DNS Server. 

The instructions I use later for Kubernetes don't require a DNS server, but I wanted to set one up to see how it is done and learn more about how DNS is configured. 

I setup the box I called f6thw as with bind9 as a DNS server while referencing the [Ubuntu DNS documentation](https://ubuntu.com/server/docs/service-domain-name-service-dns)

1. Install bind9: `sudo apt install bind9`
2. Upgrade all the packages (not strictly necessary, but good practice): `sudo apt dist-upgrade -y` (See [How to List Available Updates...](https://www.poftut.com/how-to-list-available-updates-and-updateable-packages-with-apt-apt-get-aptitude-commands/) 
for apt command explanations. 
3. Edit the bind9 configuration file `vi /etc/bind/named.conf.options`. 
5. Restat the bind9 service: `sudo systemctl restart bind9.service`. 
6. dig www.htch.com - note the query time is significantly less after the 1st time. 

When doing a dig, remember to look for the answer section.  For example, in the query below, the question section is just echoing back what I asked for, but the ANSWER section shows the address I want.  If the answer wasn't found, there won't be an answer section. 
```
# dig haproxy.joesmsse.com

; <<>> DiG 9.16.6-Ubuntu <<>> haproxy.joesmsse.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53797
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;haproxy.joesmsse.com.		IN	A

;; ANSWER SECTION:
haproxy.joesmsse.com.	7145	IN	CNAME	ns.joesmsse.com.
ns.joesmsse.com.	7145	IN	A	192.168.1.50

;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Mon Mar 15 20:05:49 UTC 2021
;; MSG SIZE  rcvd: 82
```

It was interesting for me to see the named.conf.local file points to two other "db" files: one that
specifies the mapping from name to address, and the other from address to name. 

The bind9 configuration. Key thing here is the zone and reverse zone definition and the path to the files that have the mappings. 
```
joe@f6thw:~$ cat /etc/bind/named.conf.local

zone "joesmsse.com" {
    type master;
    file "/etc/bind/db.joesmsse.com";
};

zone "1.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/db.192";
};
```

The mapping from name to IP. 
I think this is saying it serves everyting in joesmsse.com, and that the nameserver "ns" can be found on 192.168.1.50.  And, I wanted a name "haproxy" (cause this box is also where I plan to install that), so that is set to the same address (CNAME) that ns is.  Pretty sure this is where we'll set all the mappings for all the static k8s cluster boxes in the network. 
```
joe@f6thw:~$ cat /etc/bind/db.joesmsse.com
;
; BIND data file for joesmsse
;
$TTL	604800
@	IN	SOA	joesmsse.com. root.joesmsse.com. (
			      6		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	joesmsse.com.
@	IN	A	192.168.1.50
@	IN	AAAA	::1
ns      IN      A       192.168.1.50
haproxy IN      CNAME   ns
f6thw   IN      CNAME   ns
ctrl1   IN      A       192.168.1.51
ctrl2   IN      A       192.168.1.52
frhn    IN      CNAME   ctrl1
worker1 IN      A       192.168.1.53
worker2 IN      A       192.168.1.54
worker3 IN      A       192.168.1.55
```

The mapping from ip to name.  Not 100% sure if this is necessary, but does allow a reverse nslookup as shown. 
```
joe@f6thw:~$ cat /etc/bind/db.192
;
; BIND reverse data file for local loopback interface
;
$TTL	604800
@	IN	SOA	ns.joesmsse.com. root.joesmsse.com. (
			      5		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	ns.
50	IN	PTR	ns.joesmsse.com.
51      IN      PTR     ctrl1.joesmsse.com.
52      IN      PTR     ctrl2.joesmsse.com.
53      IN      PTR     worker1.joesmsse.com.
54      IN      PTR     worker2.joesmsse.com.
55      IN      PTR     worker3.joesmsse.com.

# nslookup 192.168.1.50
50.1.168.192.in-addr.arpa	name = ns.joesmsse.com.

```

Useful commands when installing DNS: 

`vi /etc/bind/named.conf.local` - edit the bind9 configuration

`named-checkzone joesmsse.com /etc/bind/db.joesmsse.com` validate your zone file

`named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192` validate your reverse zone file. 

`sudo systemctl restart bind9.service` - restart the bind9 DNS server 

`systemd-resolve --status` show you what DNS server you are resolving against. 

`vi /etc/netplan/00-installer-config.yaml` edit your static ip, gateway and resolver

`dig haproxy.joesmsse.com` query the DNS server 


### Configuring HAProxy as a Load Balancer

I did [Step 1: Configuring HAProxy as a Load Balancer](https://www.inap.com/blog/deploying-kubernetes-on-bare-metal/) but the `nc -v $KBLB_ADDR 6443` says "... Connection refused". So, I've either mis-configured HAProxy, or I need to have the Kubernetes control plane running on the ctl1 and ctlr2 boxes before haproxy can serve any traffic to them. 

I re-installed HAProxy as per the above instructions and now 
the test of it succeeded: 
```
# nc -vz 192.168.1.50 6443
Connection to 192.168.1.50 6443 port [tcp/*] succeeded!
```

### Install Docker on Kubernetes Nodes

It seems the instructions on [Step 2: Install Docker and Related Packages on All Kubernetes Nodes](https://www.inap.com/blog/deploying-kubernetes-on-bare-metal/#anchor-2)
are out of date, so jumped to the [official Docker docs](https://docs.docker.com/engine/install/ubuntu/) 

```bash
sudo apt update && sudo apt install apt-transport-https  ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io

sudo docker run hello-world
```
Docker version 5:20.10.5~3-0~ubuntu-groovy got installed by the above. 
Repeated on all the control and worker nodes. 

SIDE NOTE: 
I noticed that sometimes instructions use `apt` and other times `apt-get`.  Curious, I found this [APT vs APT-GET: What's the Difference?](https://phoenixnap.com/kb/apt-vs-apt-get). Looks like `apt` is a newer improvement that combines apt-get with apt-cache, provides better output, and is faster.  So, try to use `apt` when available. It isn't completely backwards compatible. 

### Prepare Kubernetes on all nodes
Following [Step 3: Prepare Kubernetes on All Nodes](https://www.inap.com/blog/deploying-kubernetes-on-bare-metal/#anchor-3). 

But, found that the Kubernetes official docs also cover this [Installing kubeadm, kubelet and kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

```bash
sudo su -
apt-get update && apt-get install -y apt-transport-https ca-certificates curl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-get install -y kubelet kubeadm kubectl && apt-mark hold kubelet kubeadm kubectl
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
kubeadm version
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload && systemctl restart docker
```
Did this on all the control and worker nodes. 

### Install ETCD on the 1st Control node

[Step 4](https://www.inap.com/blog/deploying-kubernetes-on-bare-metal/#anchor-4)

```
cd /root

cat > kubeadm-config.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
    certSANs:
      - "192.168.1.50"
controlPlaneEndpoint: "192.168.1.50:6443"
etcd:
     local:
       endpoints:
         - "https://192.168.1.51:2379"
         - "https://192.168.1.52:2379"
       caFile: /etc/kubernetes/pki/etcd/ca.crt
       certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
       keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
EOF
```


```
# kubeadm init --config=kubeadm-config.yaml --upload-certs
W0324 02:38:28.946536    8787 common.go:77] your configuration file uses a deprecated API spec: "kubeadm.k8s.io/v1beta1". Please use 'kubeadm config migrate --old-config old.yaml --new-config new.yaml', which will write the new, similar spec using a newer API version.
W0324 02:38:28.946901    8787 strict.go:54] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta1", Kind:"ClusterConfiguration"}: error unmarshaling JSON: while decoding JSON: json: cannot unmarshal string into Go struct field APIServer.apiServer.certSANs of type []string
v1beta1.ClusterConfiguration.APIServer: v1beta1.APIServer.CertSANs: []string: decode slice: expect [ or n, but found ", error found in #10 byte of ...|ertSANs":"– \"192.|..., bigger context ...|{"apiServer":{"certSANs":"– \"192.168.1.50\""},"apiVersion":"kubeadm.k8s.i|...
To see the stack trace of this error execute with --v=5 or higher
root@frhn:~# kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
W0324 02:40:20.974612    8886 strict.go:54] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta1", Kind:"ClusterConfiguration"}: error unmarshaling JSON: while decoding JSON: json: cannot unmarshal string into Go struct field APIServer.apiServer.certSANs of type []string
v1beta1.ClusterConfiguration.APIServer: v1beta1.APIServer.CertSANs: []string: decode slice: expect [ or n, but found ", error found in #10 byte of ...|ertSANs":"– \"192.|..., bigger context ...|{"apiServer":{"certSANs":"– \"192.168.1.50\""},"apiVersion":"kubeadm.k8s.i|...
To see the stack trace of this error execute with --v=5 or higher
root@frhn:~# vi kubeadm-config.yaml 
root@frhn:~# kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
W0324 02:42:50.829878    9020 strict.go:54] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta1", Kind:"ClusterConfiguration"}: error unmarshaling JSON: while decoding JSON: json: cannot unmarshal string into Go struct field APIServer.apiServer.certSANs of type []string
v1beta1.ClusterConfiguration.APIServer: v1beta1.APIServer.CertSANs: []string: decode slice: expect [ or n, but found ", error found in #10 byte of ...|ertSANs":"– \"192.|..., bigger context ...|{"apiServer":{"certSANs":"– \"192.168.1.50\""},"apiVersion":"kubeadm.k8s.i|...
To see the stack trace of this error execute with --v=5 or higher
root@frhn:~# vi kubeadm-config.yaml 
root@frhn:~# kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
W0324 02:43:56.040299    9077 strict.go:54] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta1", Kind:"ClusterConfiguration"}: error unmarshaling JSON: while decoding JSON: json: unknown field "caFile"
root@frhn:~# vi kubeadm-config.yaml 
root@frhn:~# kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
W0324 02:44:57.718960    9170 strict.go:54] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta1", Kind:"ClusterConfiguration"}: error unmarshaling JSON: while decoding JSON: json: unknown field "caFile"
root@frhn:~# vi kubeadm-config.yaml 
root@frhn:~# kubeadm init --config=kubeadm-config.yaml --upload-certs
W0324 02:46:31.348079    9289 common.go:77] your configuration file uses a deprecated API spec: "kubeadm.k8s.io/v1beta1". Please use 'kubeadm config migrate --old-config old.yaml --new-config new.yaml', which will write the new, similar spec using a newer API version.
W0324 02:46:31.348482    9289 strict.go:54] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta1", Kind:"ClusterConfiguration"}: error unmarshaling JSON: while decoding JSON: json: unknown field "caFile"
[init] Using Kubernetes version: v1.20.5
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.5. Latest validated version: 19.03
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [frhn kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.1.51 192.168.1.50]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [frhn localhost] and IPs [192.168.1.51 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [frhn localhost] and IPs [192.168.1.51 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 85.524579 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
f58dfe7489c3d0653eee6e3d96b861ca45f3d052b0533b373f9bbb521d83f08e
[mark-control-plane] Marking the node frhn as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
[mark-control-plane] Marking the node frhn as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: scof1a.8yq25brso95dywm2
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

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

You can now join any number of the control-plane node running the following command on each as root: 

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.50:6443 --token scof1a.8yq25brso95dywm2 \
    --discovery-token-ca-cert-hash sha256:e256108e1e0096867b6d5c8b99273deac4b0b30239a85b42d6cd5dde63776b56
```

Did this for the pod network.  It installs weave. 
```
$ sudo kubectl --kubeconfig /etc/kubernetes/admin.conf apply -f  "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
[sudo] password for joe: 
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```

Joined the 2nd control node to the first
```
$ sudo kubeadm join 192.168.1.50:6443 --token scof1a.8yq25brso95dywm2     --discovery-token-ca-cert-hash sha256:e256108e1e0096867b6d5c8b99273deac4b0b30239a85b42d6cd5dde63776b56     --control-plane --certificate-key f58dfe7489c3d0653eee6e3d96b861ca45f3d052b0533b373f9bbb521d83f08e
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.5. Latest validated version: 19.03
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [ctbz localhost] and IPs [192.168.1.52 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [ctbz localhost] and IPs [192.168.1.52 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ctbz kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.1.52 192.168.1.50]
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[check-etcd] Checking that the etcd cluster is healthy
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[etcd] Announced new etcd member joining to the existing etcd cluster
[etcd] Creating static Pod manifest for "etcd"
[etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
[kubelet-check] Initial timeout of 40s passed.
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node ctbz as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
[mark-control-plane] Marking the node ctbz as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

Joined the worker nodes to the cluster like this: 
```
joe@worker1:~$ sudo kubeadm join 192.168.1.50:6443 --token scof1a.8yq25brso95dywm2 \
>     --discovery-token-ca-cert-hash sha256:e256108e1e0096867b6d5c8b99273deac4b0b30239a85b42d6cd5dde63776b56
[sudo] password for joe: 
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.5. Latest validated version: 19.03
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

To see that it is working, use `kubectl get nodes` or `kubectl get pods -A` from one of the control plane nodes. 

I also copied the ~/.kube/config file to my Macbook (on my network, but not part of the k8s cluster): 
```
sftp joe@192.168.1.51:/home/joe/.kube/config k8s-config
$ kubectl --kubeconfig k8s-config get pods -A -o wide
NAMESPACE     NAME                           READY   STATUS    RESTARTS   AGE     IP             NODE      NOMINATED NODE   READINESS GATES
kube-system   coredns-74ff55c5b-dhkqk        1/1     Running   0          51m     10.32.0.3      frhn      <none>           <none>
kube-system   coredns-74ff55c5b-ptb24        1/1     Running   0          51m     10.32.0.2      frhn      <none>           <none>
kube-system   etcd-ctbz                      1/1     Running   0          19m     192.168.1.52   ctbz      <none>           <none>
kube-system   etcd-frhn                      1/1     Running   0          51m     192.168.1.51   frhn      <none>           <none>
kube-system   kube-apiserver-ctbz            1/1     Running   0          19m     192.168.1.52   ctbz      <none>           <none>
kube-system   kube-apiserver-frhn            1/1     Running   0          51m     192.168.1.51   frhn      <none>           <none>
kube-system   kube-controller-manager-ctbz   1/1     Running   0          19m     192.168.1.52   ctbz      <none>           <none>
kube-system   kube-controller-manager-frhn   1/1     Running   1          51m     192.168.1.51   frhn      <none>           <none>
kube-system   kube-proxy-5l8pm               1/1     Running   0          12m     192.168.1.53   worker1   <none>           <none>
kube-system   kube-proxy-5vr8v               1/1     Running   0          21m     192.168.1.52   ctbz      <none>           <none>
kube-system   kube-proxy-mmtd9               1/1     Running   0          51m     192.168.1.51   frhn      <none>           <none>
kube-system   kube-proxy-rqw28               1/1     Running   0          9m23s   192.168.1.54   worker2   <none>           <none>
kube-system   kube-proxy-z242m               1/1     Running   0          8m39s   192.168.1.55   worker3   <none>           <none>
kube-system   kube-scheduler-ctbz            1/1     Running   0          19m     192.168.1.52   ctbz      <none>           <none>
kube-system   kube-scheduler-frhn            1/1     Running   1          51m     192.168.1.51   frhn      <none>           <none>
kube-system   weave-net-27p2s                2/2     Running   1          12m     192.168.1.53   worker1   <none>           <none>
kube-system   weave-net-brcvv                2/2     Running   1          35m     192.168.1.51   frhn      <none>           <none>
kube-system   weave-net-jmgsb                2/2     Running   0          9m23s   192.168.1.54   worker2   <none>           <none>
kube-system   weave-net-jwv76                2/2     Running   0          21m     192.168.1.52   ctbz      <none>           <none>
kube-system   weave-net-nfdwh                2/2     Running   0          8m39s   192.168.1.55   worker3   <none>           <none>
```
Note that there are pods running on all the different nodes. (frhn == ctrl1 and ctbz == ctrl2)

Here is the list of nodes: 
```
$ kubectl --kubeconfig k8s-config get nodes -o wide
NAME      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
ctbz      Ready    control-plane,master   22m     v1.20.5   192.168.1.52   <none>        Ubuntu 20.10   5.8.0-45-generic   docker://20.10.5
frhn      Ready    control-plane,master   53m     v1.20.5   192.168.1.51   <none>        Ubuntu 20.10   5.8.0-45-generic   docker://20.10.5
worker1   Ready    <none>                 13m     v1.20.5   192.168.1.53   <none>        Ubuntu 20.10   5.8.0-45-generic   docker://20.10.5
worker2   Ready    <none>                 10m     v1.20.5   192.168.1.54   <none>        Ubuntu 20.10   5.8.0-45-generic   docker://20.10.5
worker3   Ready    <none>                 9m58s   v1.20.5   192.168.1.55   <none>        Ubuntu 20.10   5.8.0-45-generic   docker://20.10.5
```
I wonder why the worker nodes don't have a ROLE... 


I tried to install MySql via Helm `$ helm --kubeconfig k8s-config install joes-mysql bitnami/mysql`
But, the pods have the following error event: 
`0/5 nodes are available: 5 pod has unbound immediate PersistentVolumeClaims`

Next... remember how to create PVCs. 


