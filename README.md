
# DVWA in Kubernetes

## Overview

The point of this lab is to teach me Kubernetes. I already had an understanding of container but did not have a deep understanding of what kubernetes looked like or all the features it could do. 

This lab was completed locally using my Cisco UCS which has 2 TB of storage and 128 GB of memory. The control plane contains 3 master nodes running in High Availability and are connected via the control plane load balancer on 192.168.1.75. The data plane consists of 2 worker nodes that expose services via the MetalB application loadbalancer. Calico was chosen as the CNI.

## Components

**Master Nodes**

1. MasterA = 192.168.1.70
2. MasterB = 192.168.1.71
3. Master C = 192.168.1.72

**Worker Nodes**

1. WorkerA = 192.168.1.73
2. WorkerB = 192.168.1.74

**Supporting Nodes**

1. Control Load Balancer = 192.168.1.75
2. NFS Share = 192.168.1.76


## The Lab - DVWA

Kubernetes manages applications via manifest files. These files define the desired state of a cluster of objects where your application may run in. Here is a llst of objects that can be defined in a manifest file but there are many more available

1. namespace =  Creates virtual boundaries inside a cluster to isolate resources, teams, or environments (e.g., `dev`, `staging`, `prod`).
2. node = Represents a physical or virtual worker machine in the cluster (usually managed automatically by the cluster, though modifiable via manifests).
3. peristent volume = A piece of physical or cloud storage provisioned in the cluster by an administrator or dynamically
4. persistent volume claim = `A request for storage made by a user or application that binds to an available PersistentVolume`
5. service = Provides a stable IP address and DNS name to expose a dynamic group of Pods internally (`ClusterIP`) or externally (`NodePort`, `LoadBalancer`)
6. deployment = Manages stateless applications, handling replica counts, scaling, and zero-downtime rolling updates

DVWA has been configured to run as a pod with a replica count of 2 and call on multiple PVs and PVCs for storage. In this confguration DVWA can exist to handle an increase in load, high availability, and self healing by recreating pods when they break. DVWA has its main database in MariaDB and a secondary storage directory when users upload files to the application. 


# Installation

1. Perform baseline package installation against the 3 master and 2 worker nodes

```
ansible-playbook -i inventory.ini setup.yml -k -K
```

2. Configure control node loadbalancer

```
sudo apt install haproxy
vim /etc/haproxy/haproxy.cfg 

#paste the following lines below
frontend kubernetes-frontend
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp 
    balance roundrobin
    option tcp-check
    server mastera 192.168.1.70:6443 check inter 2s fall 2 rise 2  
    server masterb 192.168.1.71:6443 check inter 2s fall 2 rise 2
    server masterc 192.168.1.72:6443 check inter 2s fall 2 rise 2


sudo systemctl restart haproxy
```

3. Initialize the first master node:
   1. The *192.168.1.65* address is the control node loadbalancer which will be treated as the endpoint for master nodes

```
sudo kubeadm init --control-plane-endpoint "192.165.1.65:6443" --upload-certs --pod-network-cidr=10.244.0.0/16

#To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install your CNI
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

4. You should see a line in your terminal with the commands to add both master and worker nodes
5. Configure application loadbalancer

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.2/config/manifests/metallb-native.yaml
kubectl wait -n metallb-system --for=condition=ready pod --selector=app=metallb --timeout=120s

vim metallb-config.yaml

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - homelab-pool



kubectl apply -f metallb-config.yaml
```

7. Configure NFS file share server

```
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/k8s/dvwa-uploads /srv/nfs/k8s/dvwa-db
sudo chown 33:33 /srv/nfs/k8s/dvwa-uploads

sudo vim /etc/exports 
/srv/nfs/k8s 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)

sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
sudo systemctl enable --now nfs-kernel-server
sudo systemctl stop ufw
```

7. Configure NFS clients on master and worker nodes

```
sudo apt install -y nfs-common
```

9. Run DVWA

```
kubectl apply -f dvwa/
```

8. Confirmation commands

```
kubectl get pods -A -o wide = to see what IP each pod got assigned for the default namespace
kubectl get pods -n dvwa -o wide = same command but for the dvwa namespace
kubectl get nodes = to see if your nodes are all there


#Imagine you messed up and need to reset kubernetes
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/etcd ~/.kube
sudo systemctl restart containerd kubelet
#then run the join commands again
```

9. Something I kept messing up:

   1. A common source of frustration was not having the proper settings on your SystemCgroup
   2. This variable controls is systemd will be used to manage containerd and needs to be set to true before joining nodes, if you keep messing up your initial install you may need to go back and ensure this is configured before joining nodes

   ```
   containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
   sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
   grep SystemdCgroup /etc/containerd/config.toml    # should say true
   sudo systemctl restart containerd
   ```
