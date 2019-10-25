# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

Fix the controllers hostnames just in case:

```
[root@smc-master k8s-th]# for node in master00 master01 master02; do kcli ssh $node hostnamectl set-hostname ${node}.${DOMAIN}; kcli ssh $node "hostname -f"; done
```

The commands in this lab must be run on each controller instance: `controller-0`, `controller-1`, and `controller-2`. Login to each controller:

```
ssh -i ~/.ssh/k8s.pem centos@controller-0.${DOMAIN}
```

### Time synchronization

```
[root@smc-master ~]# yum install rhel-system-roles.noarch ansible

```


```
[k8s-thw:children]
masters
workers
lb

[masters]
master00
master01
master02

[workers]
worker00
worker01
worker02

[lb]
loadbalancer

```
> Playbook

```
 - hosts: all
   vars:
     timesync_ntp_provider: chrony
     timesync_ntp_servers:
       - hostname: clock.corp.redhat.com
         iburst: yes
  roles:
    - rhel-system-roles.timesync

```

> Run playbook

```
[root@smc-master ~]# ansible-playbook -i inventory timesync.yml
```

> Verify

```
# ansible -i inventory all -a "date" 
master02 | CHANGED | rc=0 >>
Thu Oct 24 07:53:15 UTC 2019

master00 | CHANGED | rc=0 >>
Thu Oct 24 07:53:15 UTC 2019

master01 | CHANGED | rc=0 >>
Thu Oct 24 07:53:15 UTC 2019

worker00 | CHANGED | rc=0 >>
Thu Oct 24 07:53:15 UTC 2019

worker01 | CHANGED | rc=0 >>
Thu Oct 24 07:53:15 UTC 2019

worker02 | CHANGED | rc=0 >>
Thu Oct 24 07:53:15 UTC 2019

loadbalancer | CHANGED | rc=0 >>
Thu Oct 24 07:53:15 UTC 2019
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:

```
sudo yum install -y wget
wget -q --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.0/etcd-v3.4.0-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` command line utility:

```
{
  tar -xvf etcd-v3.4.0-linux-amd64.tar.gz
  sudo mv etcd-v3.4.0-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server

```
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance:

```
INTERNAL_IP=$(hostname --all-ip-addresses)
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:

```
ETCD_NAME=$(hostname -s)
```

Create the `etcd.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
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
  --initial-cluster master00=https://192.168.111.72:2380,master01=https://192.168.111.173:2380,master02=https://192.168.111.230:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the etcd Server

```
{
sud
  sudo systemctl enable etcd --now
}
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.

## Verification

List the etcd cluster members:

```
sudo ETCDCTL_API=3 /usr/local/bin/etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> output

```
8cc74a63829539dc, started, master02, https://192.168.111.230:2380, https://192.168.111.230:2379, false
b5a40986f14f0229, started, master01, https://192.168.111.173:2380, https://192.168.111.173:2379, false
cda48ac7547eed24, started, master00, https://192.168.111.72:2380, https://192.168.111.72:2379, false

```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
