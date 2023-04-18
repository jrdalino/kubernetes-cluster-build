# Building a Practice Kubernetes Cluster

## Prerequisites
- Distribution: Ubuntu 20.04 Focal Fossa LTS & Size: medium
- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Installing kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Creating a cluster with kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

- Set appropriate hostnames for each node
```
sudo hostnamectl set-hostname k8s-control
sudo hostnamectl set-hostname k8s-worker1
sudo hostnamectl set-hostname k8s-worker2
```

## Perfrom on all nodes
- On all nodes, set up the hosts file to enable all the nodes to reach each other using these hostnames
```
sudo vi /etc/hosts
```

- On all nodes, add the following at the end of the file. You will need to supply the actual private IP address for each node
```
<control plane node private IP> k8s-control
<worker node 1 private IP> k8s-worker1
<worker node 2 private IP> k8s-worker2
```

- Log out of all three servers and log back in to see these changes take effect

- On all nodes, set up Docker Engine and containerd. You will need to load some kernel modules and modify some system settings as part of this process
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

- sysctl params required by setup, params persist across reboots
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

- Apply sysctl params without reboot
```
sudo sysctl --system
```

- Set up the Docker Engine repository
```
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https
```

- Add Docker’s official GPG key
```
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

- Set up the repository
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

- Update the apt package index
```
sudo apt-get update
```

- Install Docker Engine, containerd, and Docker Compose
```
VERSION_STRING=5:23.0.1-1~ubuntu.20.04~focal
sudo apt-get install -y docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin
```

- Add your 'cloud_user' to the docker group
```
sudo usermod -aG docker $USER
```

- Log out and log back in so that your group membership is re-evaluated
- Make sure that 'disabled_plugins' is commented out in your config.toml file
```
sudo sed -i 's/disabled_plugins/#disabled_plugins/' /etc/containerd/config.toml
```

- Restart containerd
```
sudo systemctl restart containerd
```

- On all nodes, disable swap.
```
sudo swapoff -a
```

- On all nodes, install kubeadm, kubelet, and kubectl
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update && sudo apt-get install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00

sudo apt-mark hold kubelet kubeadm kubectl
```

## Perform on the control plane node only
- Initialize the cluster and set up kubectl access
```
sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.24.0

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Verify the cluster is working
```
kubectl get nodes
```

## Perform on each of the worker nodes
- Install the Calico network add-on
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

- Get the join command (this command is also printed during kubeadm init . Feel free to simply copy it from there)
```
kubeadm token create --print-join-command
```

- Copy the join command from the control plane node. Run it on each worker node as root (i.e. with sudo )
```
sudo kubeadm join ...
```

- Verify that all nodes are joined and ready
```
$ kubectl get nodes
NAME                    STATUS   ROLES    AGE   VERSION
k8s-control.example.com   	Ready    master   1m   v1.13.4
k8s-worker1.example.com   	Ready    <none>   1m   v1.13.4
k8s-worker2.example.com   	Ready    <none>   1m   v1.13.4
```

## Run end to end tests
- Run a simple nginx deployment:
```
$ kubectl run nginx --image=nginx
```

- View the deployments in your cluster:
```
$ kubectl get deployments
```

- View the pods in the cluster:
```
$ kubectl get pods
```

- Use port forwarding to access a pod directly:
```
$ kubectl port-forward $pod_name 8081:80
```

- Get a response from the nginx pod directly:
```
$ curl --head http://127.0.0.1:8081
```

- View the logs from a pod:
```
$ kubectl logs $pod_name
```

- Run a command directly from the container:
```
$ kubectl exec -it nginx -- nginx -v
```

- Create a service by exposing port 80 of the nginx deployment:
```
$ kubectl expose deployment nginx --port 80 --type NodePort
```

- List the services in your cluster:
```
$ kubectl get services
```

- Get a response from the service:
```
$ curl -I localhost:$node_port
```

- List the nodes' status:
```
$ kubectl get nodes
```

- View detailed information about the nodes:
```
$ kubectl describe nodes
```

- View detailed information about the pods:
```
$ kubectl describe pods
```
