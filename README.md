## 1. Installing VM software

```zsh
brew install --cask multipass
```

## 2. Creating VMs

Read the `cloud-init-*` files to see how the VMs are configured.

Launching the VMs for the cluster:

```zsh
multipass launch --name k8s-control-plane --cpus 2 --memory 2G --disk 10G --cloud-init cloud-init-control-plane.yaml
multipass launch --name k8s-worker1 --cpus 2 --memory 2G --disk 10G --cloud-init cloud-init-worker.yaml
multipass launch --name k8s-worker2 --cpus 2 --memory 2G --disk 10G --cloud-init cloud-init-worker.yaml

multipass list
``` 

Some useful tips when verifying cloud-init files:

```zsh
# Validate YAML syntax
yamllint cloud-init-control-plane.yaml
yamllint cloud-init-worker.yaml

# Check cloud-init status
multipass exec k8s-control-plane -- sudo cloud-init status -l
multipass exec k8s-worker1 -- sudo cloud-init status -l
multipass exec k8s-worker2 -- sudo cloud-init status -l

# Check cloud-init logs after VM is launched
multipass exec k8s-control-plane -- sudo cat /var/log/cloud-init.log
multipass exec k8s-worker1 -- sudo cat /var/log/cloud-init.log
multipass exec k8s-worker2 -- sudo cat /var/log/cloud-init.log

```

## 3. Accessing VMs

```zsh
# shell into each VM
multipass shell k8s-control-plane
multipass shell k8s-worker1
multipass shell k8s-worker2

# executing commands on each VM without shelling in
multipass exec k8s-control-plane -- <command>
multipass exec k8s-worker1 -- <command>
multipass exec k8s-worker2 -- <command>
```

## TODO 4. Initializing the Kubernetes Cluster
```zsh
sudo kubeadm init --pod-network-cidr=<TBD>
```

## TODO 5. Setting up kubeconfig for the master node
