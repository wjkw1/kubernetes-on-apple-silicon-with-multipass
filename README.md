## 1. Installing VM software

```zsh
brew install --cask multipass
```

## 2. Creating VMs
This step creates the VMs that will form the Kubernetes cluster using multipass.

To launch the VMs, run the following commands from your host machine:

```zsh
multipass launch --name k8s-control --cpus 2 --memory 2G --disk 10G --cloud-init cloud-init-k8s-node-setup.yaml
multipass launch --name k8s-worker --cpus 2 --memory 2G --disk 10G --cloud-init cloud-init-k8s-node-setup.yaml

multipass list
``` 

Read the `cloud-init-*.yaml` files to see how the VMs are configured.
Note, we are using k8s version 1.34 so there are multiple versions of kubeadm, kubelet, and kubectl installed for upgrade learning purposes.



## 3. Accessing VMs
This step shows how to access the VMs created in the previous step.

```zsh
# shell into each VM
multipass shell k8s-control
multipass shell k8s-worker

# executing commands on each VM without shelling in
multipass exec k8s-control -- <command>
multipass exec k8s-worker -- <command>
```

Some useful commands to verify the setup:

```zsh
# Check cloud-init status
multipass exec k8s-control -- sudo cloud-init status -l
multipass exec k8s-worker -- sudo cloud-init status -l

# Check custom setup script logs
multipass exec k8s-control -- sudo cat /var/log/setup-node.log
multipass exec k8s-worker -- sudo cat /var/log/setup-node.log

# Check cloud-init logs after VM is launched
multipass exec k8s-control -- sudo cat /var/log/cloud-init.log
multipass exec k8s-worker -- sudo cat /var/log/cloud-init.log
```

## 4. Initializing the Kubernetes Cluster
This step sets up the control plane node. Run the following command inside the control plane VM:
```zsh
multipass shell k8s-control
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Or execte it from your host machine:
```zsh
multipass exec k8s-control -- hostname -I | awk '{print $1}'  # get the IP address of the control plane node
multipass exec k8s-control -- sudo kubeadm init --pod-network-cidr=10.0.0.0/16 --apiserver-advertise-address=192.168.64.45
```

## 5. Setting up kubeconfig for the cluster on your host machine
This allows you to use `kubectl` from your host machine to interact with the cluster.
```zsh
multipass exec k8s-control -- sudo cat /etc/kubernetes/admin.conf > ~/.kube/admin.config
export KUBECONFIG=~/.kube/admin.config
kubectl get nodes
kubectl get pods -n kube-system
```

At this point, the control plane node will be in NotReady status and the pods will be in Pending status because the Pod Network Add-on (CNI plugin) has not been installed yet:
```zsh
(.venv) ➜  kubeadm git:(main) ✗ k get nodes
NAME                STATUS     ROLES           AGE     VERSION
k8s-control   NotReady   control-plane   7m57s   v1.35.0

(.venv) ➜  kubeadm git:(main) ✗ k get pods -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
coredns-7d764666f9-h5xhq                    0/1     Pending   0          4m29s
coredns-7d764666f9-vwfr8                    0/1     Pending   0          4m29s
...
```

## 6. Installing a Pod Network Add-on (CNI Plugin)
To enable communication between the pods in the cluster, you need to install a Pod Network Add-on. 
There are several options available, such as Flannel, Calico, Weave Net, etc.

### 6a. Installing Calico CNI Plugin (Production grade Alternative)
This gives you network policy capabilities in addition to pod networking. Which is useful for training for the CKA exam.

To install Calico CNI plugin run:
```zsh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# Verify that the nodes are in Ready status and the pods are running:
kubectl get nodes
kubectl get pods -n kube-system
```

### 6b. Installing Flannel CNI Plugin for Lab setup
Install Flannel CNI plugin:
```zsh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# Verify that the nodes are in Ready status and the pods are running:
kubectl get nodes
kubectl get pods -n kube-system
```

After installing the CNI plugin, give it a few moments to initialize. Here is an example of the intermediate state:
```
(.venv) ➜  kubeadm git:(main) ✗ k get nodes              
NAME                STATUS     ROLES           AGE   VERSION
k8s-control   NotReady   control-plane   12m   v1.35.0

(.venv) ➜  kubeadm git:(main) ✗ k get pods -A
NAMESPACE      NAME                                        READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-fjw76                       1/1     Running   0          24s
kube-system    coredns-7d764666f9-h5xhq                    0/1     Pending   0          12m
kube-system    coredns-7d764666f9-vwfr8                    0/1     Pending   0          12m

(.venv) ➜  kubeadm git:(main) ✗ k get pods -A
NAMESPACE      NAME                                        READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-fjw76                       1/1     Running   0          28s
kube-system    coredns-7d764666f9-h5xhq                    1/1     Running   0          12m
```
Eventually, the control plane node should be in Ready status and all pods should be running:
```zsh
(.venv) ➜  kubeadm git:(main) ✗ k get nodes
NAME                STATUS   ROLES           AGE    VERSION
k8s-control   Ready    control-plane   10m    v1.35.0
(.venv) ➜  kubeadm git:(main) ✗ k get pods -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
kube-flannel-ds-fjw76                       1/1     Running   0          28s
coredns-7d764666f9-h5xhq                    1/1     Running   0          12m
coredns-7d764666f9-vwfr8                    1/1     Running   0          12m
...
```

## 7. Joining Worker Nodes to the Cluster

To join worker nodes to the cluster, run the `kubeadm join` command provided in the output of the `kubeadm init` command on each worker node. For example:
```zsh
multipass exec k8s-worker -- sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
Replace `<control-plane-ip>`, `<token>`, and `<hash>` with the actual values from your `kubeadm init` output.

If like me, you missed the output, you can retrieve the join command from the control plane node by running:
```zsh
multipass exec k8s-control -- sudo kubeadm token create --print-join-command
```

## 8. Verifying the Cluster
After joining the worker nodes, verify that all nodes are in Ready status:
```zsh
kubectl get nodes
```

If the node still shows as NotReady, give it a minute for the node to transition to Ready status. It's probably still initializing the flannel CNI plugin (which gives connectiviy).

## 9. TODO Updating or upgrading the Cluster
To update or upgrade the Kubernetes cluster, you can follow these general steps:
1. **Backup your cluster**: Before making any changes, ensure you have a backup of your cluster configuration and data.
2. **Update kubeadm**: On each node, update the kubeadm package to the desired version.
3. **Drain the node**: Before upgrading a node, drain it to safely evict all pods.
4. **Upgrade the control plane node**: Run the `kubeadm upgrade` command on the control plane node to upgrade the Kubernetes control plane components.
5. **Upgrade worker nodes**: After upgrading the control plane, upgrade each worker node by


## 9. Cleaning Up
To delete the VMs and clean up resources, run the following commands from your host machine:
```zsh
multipass delete k8s-control
multipass delete k8s-worker
...
multipass purge
```


