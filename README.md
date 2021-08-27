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

June 25, 2019 (Kubernetes version v1.15)

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

> To install Docker CE, you need the 64-bit version of one of these Ubuntu versions:
>
> - Cosmic 18.10
> - Bionic 18.04 (LTS)
> - Xenial 16.04 (LTS)

[ansible/roles/docker/tasks/main.yml](ansible/roles/docker/tasks/main.yml)

Packages:

- docker-ce
- docker-ce-cli
- containerd\.io

### Kubeadm

See: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>

> One or more machines running one of:
>
> - Ubuntu 16.04+
> - Debian 9
> - CentOS 7

[ansible/roles/kubeadm/tasks/main.yml](ansible/roles/kubeadm/tasks/main.yml)

Packages:

- kubeadm
- kubelet
- kubectl

## Cluster

Now that machines have been prepared, it is time to install Kubernetes.

You need one master, and one or more nodes (formerly known as minions).

Master:

```shell
kubeadm init [--pod-network-cidr=10.244.0.0/16]
```

<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/>

Minion:

```shell
kubeadm join [--token=... master:6443]
```

<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/>

### Single-node

If you only have room for a single machine, you run pods on the master:

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

In a real production environment, you would have multiple masters (HA).

See: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/>

### Network

A cluster network needs to be deployed as well, two popular choices are:

- Flannel

- Calico

These are normally deployed as resources, by using a `kubectl` command.

See: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network>

#### Flannel

> Flannel is a way to configure a layer 3 network fabric designed for Kubernetes.

<https://github.com/coreos/flannel>

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.14.0/Documentation/kube-flannel.yml
```

### Dashboard

> Kubernetes Dashboard is a general purpose, web-based UI for Kubernetes clusters.

<https://github.com/kubernetes/dashboard>

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

### Metrics Server

> Metrics Server collects resource metrics and exposes them in Kubernetes apiserver.

<https://github.com/kubernetes-sigs/metrics-server>

```shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
```

### Images

As per Kubernetes v1.20.0, here is the list of container images used:

```text
k8s.gcr.io/kube-apiserver:v1.20.0
k8s.gcr.io/kube-controller-manager:v1.20.0
k8s.gcr.io/kube-scheduler:v1.20.0
k8s.gcr.io/kube-proxy:v1.20.0
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0
```

They total a download size of 210 MiB, in their default compressed form.

## References

March 15, 2019 (Kubernetes version v1.13)
<https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/>

Apr 2, 2018 (Kubernetes version v1.9)
<https://medium.com/@lizrice/kubernetes-in-vagrant-with-kubeadm-21979ded6c63>
