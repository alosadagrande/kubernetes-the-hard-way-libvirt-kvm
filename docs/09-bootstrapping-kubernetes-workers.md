# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet) and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies)


## Prerequisites

Remember that the fully qualified domain name (FQDN) of each member of the cluster must be resolvable by DNS: direct and reverse lookup. Therefore, check and confirm the controllers hostnames just in case: 

```
DOMAIN=k8s-thw.local
for node in worker00 worker01 worker02; do
	kcli ssh $node hostnamectl set-hostname ${node}.${DOMAIN}
	kcli ssh $node "hostname -f"
done
```

The commands in this lab must be run on each worker instance: `worker00`, `worker01`, and `worker02`. Login to each controller instance:

```
kcli ssh worker00
```


### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

> There are other options, the most remarkable in my opinion is [terminator](https://terminator-gtk3.readthedocs.io/en/latest/) which I use a lot.

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```
{
  sudo yum install epel-release -y
  sudo yum install socat conntrack ipset wget jq vim -y
  sudo yum-config-manager --disable epel
}
```

> The socat binary enables support for the `kubectl port-forward` command.

Disable selinux (I know, I know):

```
sudo setenforce 0
sudo sed -i -e 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```


### Download and Install Worker Binaries

On each worker perform the following commands:

```
wget \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.16.1/crictl-v1.16.1-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc9/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.2/cni-plugins-linux-amd64-v0.8.2.tgz \
  https://github.com/containerd/containerd/releases/download/v1.3.0/containerd-1.3.0.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kubelet


```

Create the installation directories:

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kubernetes \
  /var/run/kubernetes \
  /var/lib/kube-proxy

```

Install the worker binaries:

```
{
  mkdir containerd
  tar -xvf crictl-v1.16.1-linux-amd64.tar.gz
  tar -xvf containerd-1.3.0.linux-amd64.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-amd64-v0.8.2.tgz -C /opt/cni/bin/
  sudo mv runc.amd64 runc
  chmod +x crictl kubectl kube-proxy kubelet runc
  sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/
}
```

### Configure CNI Networking

Retrieve the Pod CIDR range for the current compute instance:

```
POD_CIDR=$(cat /home/centos/pod_cidr.txt)
```

> NOTE that when we deploy the [worker nodes](https://github.com/alosadagrande/kubernetes-the-hard-way-libvirt-kvm/blob/master/docs/03-compute-resources.md#kubernetes-workers) we already assigned a subnet of the pod network range to each worker. This information was stored in a filename called pod_cidr.txt into centos username home.

Create the bridge network configuration file:

```
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
```

> Create the loopback network configuration file:

```
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
```

### Configure containerd

Create the `containerd` configuration file:

```
sudo mkdir -p /etc/containerd/
```

```
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

Create the `containerd.service` systemd unit file:

```
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
```

### Configure the Kubelet

```
{
  SHORTNAME=$(hostname -s)
  sudo mv ${SHORTNAME}-key.pem ${SHORTNAME}.pem /var/lib/kubelet/
  sudo mv ${SHORTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

Create the `kubelet-config.yaml` configuration file:

```
POD_CIDR=$(cat /home/centos/pod_cidr.txt)
SHORTNAME=$(hostname -s)

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
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${SHORTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${SHORTNAME}-key.pem"
EOF
```

> The `resolvConf` configuration is used to avoid loops when using CoreDNS for service discovery on systems running `systemd-resolved`.

Create the `kubelet.service` systemd unit file:

```
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
```

## Configure the Kubernetes Proxy

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the kube-proxy-config.yaml configuration file:

```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

Create the kube-proxy.service systemd unit file:

```
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
```

### Start the Worker Services

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy --now
}
```

> Remember to run the above commands on each worker node: `worker00`, `worker01`, and `worker02`.


## Verification

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from any of the master nodes.

List the registered Kubernetes nodes:

```
kcli ssh master00 \
  "kubectl get nodes --kubeconfig admin.kubeconfig -o wide"
```

> output

```
NAME                     STATUS   ROLES    AGE     VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
worker00.k8s-thw.local   Ready    <none>   13m     v1.16.2   192.168.111.198   <none>        CentOS Linux 7 (Core)   3.10.0-1062.4.1.el7.x86_64   containerd://1.3.0
worker01.k8s-thw.local   Ready    <none>   13m     v1.16.2   192.168.111.253   <none>        CentOS Linux 7 (Core)   3.10.0-1062.4.1.el7.x86_64   containerd://1.3.0
worker02.k8s-thw.local   Ready    <none>   13m     v1.16.2   192.168.111.158   <none>        CentOS Linux 7 (Core)   3.10.0-1062.4.1.el7.x86_64   containerd://1.3.0
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
