# Vanilla Kubernetes

Installing a multi-node Kubernetes cluster, by using:

- Vagrant
  - Ubuntu (operating system)
  - VirtualBox (hypervisor)

- Ansible
  - Docker (container runtime)
  - Kubeadm (installer)

There are alternatives, for each of the four components...

But this setup is a very common and tested combination.

Alternative setups:

- Minikube (uses machine)
- Microk8s (uses snap)

Those are more all-in-one solutions,and slightly opaque...

This method is transparent, although resource-intensive:

```console
vagrant up
```

Written by Anders Björklund @afbjorklund

Dec 16, 2022 (Kubernetes version v1.25)

Official method:

- [Kubespray](https://kubernetes.io/docs/setup/production-environment/tools/kubespray/) (HA cluster)

## Vagrant

You will need to install the `vagrant` tool, in order to run the `Vagrantfile`:

```ruby
Vagrant.configure("2") do |config|
  ...
end
```

<https://vagrantup.com>

### Ubuntu

The virtual machine is packed into a .box, for each provider (such as VirtualBox):

```ruby
  config.vm.box = "ubuntu/xenial64"
  config.vm.box_check_update = true
```

### VirtualBox

Kubernetes wants more resources from VirtualBox, than the default VM allocation:

> - 2 GB or more of RAM per machine (any less will leave little room for your apps)
> - 2 CPUs or more

```ruby
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end
```

Ansible provisioning requires ssh access with sudo, and that python is installed:

```ruby
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y python python-apt
  SHELL
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/playbook.yml"
    ansible.verbose = false
  end
```

## Ansible

You will need to install `ansible-playbook`, in order to run the `playbook.yaml`:

```yaml
- hosts: all
  become: yes
  roles:
    - docker
    - kubeadm
```

<https://ansible.com>

### Docker

See: <https://docs.docker.com/install/linux/docker-ce/ubuntu/>

[ansible/roles/docker/tasks/main.yml](ansible/roles/docker/tasks/main.yml)

Packages:

- docker-ce
- docker-ce-cli
- containerd\.io

### Requirements

Since Kubernetes version 1.24, it is now **required** to install and configure CRI and CNI:

- cri-tools (<https://github.com/kubernetes-sigs/cri-tools>)
  `/etc/crictl.yaml`
- kubernetes-cni (<https://github.com/containernetworking/plugins>)
  `/etc/cni/net.d`

The default container runtime is containerd, to use docker additional setup is needed:

- cri-dockerd (<https://github.com/Mirantis/cri-dockerd>)
  `cri-docker.service`

### Kubeadm

See: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>

[ansible/roles/kubeadm/tasks/main.yml](ansible/roles/kubeadm/tasks/main.yml)

Packages:

- kubeadm
- kubelet
- kubectl

### Config

Using kubeadm now requires a YAML file, instead of the previous flags:

```console
$ vagrant ssh
...
vagrant@kubernetes:~$ sudo kubeadm --config kubeadm-config.yaml init
```

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock
```

## Cluster

Now that machines have been prepared, it is time to install Kubernetes.

You need one master, and one or more nodes (formerly known as minions).

Master:

```shell
kubeadm init [--pod-network-cidr=10.244.0.0/16]
```

<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/>

Note: the change of subnet CIDR is only needed for "flannel" (see below)

```yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
networking:
  podSubnet: "10.244.0.0/16"
```

Minion:

```shell
kubeadm join [--token=... master:6443]
```

<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/>

### Single-node

If you only have room for a single machine, you run pods on the master:

```shell
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

In a real production environment, you would have multiple masters (HA).

See: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/>

### Network

A cluster network needs to be deployed as well, two popular choices are:

- Flannel

- Calico

These are normally deployed as resources, by using a `kubectl` command.

See: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network>

### Images

As per Kubernetes v1.25.0, here is the list of container images used:

```shell
kubeadm config images list
```

```text
registry.k8s.io/kube-apiserver:v1.25.0
registry.k8s.io/kube-controller-manager:v1.25.0
registry.k8s.io/kube-scheduler:v1.25.0
registry.k8s.io/kube-proxy:v1.25.0
registry.k8s.io/pause:3.8
registry.k8s.io/etcd:3.5.4-0
registry.k8s.io/coredns/coredns:v1.9.3
```

They total a download size of 200 MiB, in their default compressed form.

### Flannel

> Flannel is a way to configure a layer 3 network fabric designed for Kubernetes.

<https://github.com/flannel-io/flannel>

```shell
rm -f /etc/cni/net.d/*.conf*  # remove any existing CNI configuration, "there can be only one" </highlander>
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml
```

```text
docker.io/flannelcni/flannel:v0.20.2
docker.io/flannelcni/flannel-cni-plugin:v1.1.0
```

### Dashboard

> Kubernetes Dashboard is a general purpose, web-based UI for Kubernetes clusters.

<https://github.com/kubernetes/dashboard>

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

```text
docker.io/kubernetesui/dashboard:v2.7.0
docker.io/kubernetesui/metrics-scraper:v1.0.8
```

Note: to be able to _access_ the dashboard you need to create a user, token and proxy:

`kubectl proxy`

<http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/>

<https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md>

### Metrics Server

> Metrics Server collects resource metrics and exposes them in Kubernetes apiserver.

<https://github.com/kubernetes-sigs/metrics-server>

```shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.2/components.yaml
```

```text
k8s.gcr.io/metrics-server/metrics-server:v0.6.2
```

Note: there is a bug in that the server name is not included in the https certificate.

The easiest workaround is editing the metrics-server Deployment, to allow the access:

`kubectl edit deployment -n kube-system metrics-server`

```diff
@@ -129,6 +129,7 @@
     spec:
       containers:
       - args:
+        - --kubelet-insecure-tls
         - --cert-dir=/tmp
         - --secure-port=443
         - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
```

## References

Sep 14, 2020 (Kubernetes version v1.19)
<https://itnext.io/kubernetes-production-cluster-with-vagrant-and-virtualbox-on-mac-8718586f179f>

March 15, 2019 (Kubernetes version v1.13)
<https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/>

Apr 2, 2018 (Kubernetes version v1.9)
<https://medium.com/@lizrice/kubernetes-in-vagrant-with-kubeadm-21979ded6c63>
