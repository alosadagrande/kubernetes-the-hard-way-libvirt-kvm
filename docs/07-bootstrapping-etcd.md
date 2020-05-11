# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). etcd is a strongly consistent, distributed key-value store that provides a reliable way to store data that needs to be accessed by a distributed system or cluster of machines

In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

Remember that the fully qualified domain name (FQDN) of each member of the cluster must be resolvable by DNS: direct and reverse lookup. Therefore, check and confirm the controllers hostnames just in case: 

```
DOMAIN=k8s-thw.local
for node in master00 master01 master02; do
	kcli ssh $node hostnamectl set-hostname ${node}.${DOMAIN}
	kcli ssh $node "hostname -f"
done
```

### Time synchronization

etcd as any distributed system requires its components to be in sync. However, as a good practice we suggest to configure time synchronization along all the VMs in the infrastructure. To do so, we will leverage Ansible and system roles that comes as a package in CentOS.

> [Ansible](https://github.com/ansible/ansible) is a configuration management tool that automates the configuration of multiple servers by the use of Ansible playbooks. It handles configuration management, application deployment, cloud provisioning, ad-hoc task execution, network automation, and multi-node orchestration. Ansible makes complex changes like zero-downtime rolling updates with load balancers easy

First, install rhel-system-roles package on the baremetal server, which will be the Ansible control node:
```
yum install rhel-system-roles.noarch ansible
```

Create a basic inventory called `inventory` which actually contains all the servers that are part of the infrastructure.

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

Create a playbook that consists on calling the system role called timesync. A description of this role and all possible options and configurations are installed with the rhel-system-role rpm and located at /usr/share/ansible/roles/linux-system-roles.timesync/README.md. In this case we are using chrony as the synchronization tool instead of ntp, however it can be easily change taking a look at the documentation.

```
 - hosts: all
   vars:
     timesync_ntp_provider: chrony
     timesync_ntp_servers:
       - hostname: pool.ntp.org
         iburst: yes
  roles:
    - rhel-system-roles.timesync

```

> An Ansible playbook is an organized unit of scripts that defines work for a server configuration managed by the automation tool Ansible

Then, run playbook:

```
ansible-playbook -i inventory timesync.yml
```

Verify that the clocks are in sync. This task can be easily done with Ansible ad-hoc commands:

```
# ansible -i inventory all -a "date" 
```

Output:

```
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

### Running commands in parallel with tmux or terminator

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

> There are other options, the most remarkable in my opinion is [terminator](https://terminator-gtk3.readthedocs.io/en/latest/) which I use a lot.


## Bootstrapping an etcd Cluster Member

The commands to bootstrap etcd must be run **on each controller instance**: `master00`, `master01`, and `master02`. Login to each controller can be easily accomplished by:

```
kcli ssh centos@master00.${DOMAIN}
```

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:

```
sudo yum install -y wget bind-utils
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
INTERNAL_IP=$(hostname --ip-address)
for node in master00 master01 master02 ; do export IP_${node}=$(dig +short ${node}) ;done
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
  --initial-cluster master00=https://${IP_master00}:2380,master01=https://${IP_master01}:2380,master02=https://${IP_master02}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

> NOTE that `initial-cluster` value is composed by all three masters or controllers since this is a distributed system.


### Start the etcd Server

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd --now
}
```

> Remember to run the above commands on each controller node: `master00`, `master01`, and `master02`.

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
