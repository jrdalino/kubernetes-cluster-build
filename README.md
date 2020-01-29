# Building a Practice Kubernetes Cluster

## Perform the following on all nodes (master and worker)

- Get the Docker gpg key
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

- Add the Docker repository
```
$ sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

- Get the Kubernetes gpg key
```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

- Add the Kubernetes repository 
```
$ cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

- Update your packages
```
$ sudo apt-get update
```

- Install Docker, kubelet, kubeadm, and kubectl
```
$ sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet=1.13.5-00 kubeadm=1.13.5-00 kubectl=1.13.5-00
```

- Hold them at the current version:
```
sudo apt-mark hold docker-ce kubelet kubeadm kubectl
```

- Add the iptables rule to sysctl.conf:
```
$ echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
```

- Enable iptables immediately:
```
$ sudo sysctl -p
```

## Perform on the Kube master server only

- Initialize the cluster (run only on the master):
```
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

- Set up local kubeconfig:
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Apply Flannel CNI network overlay:
```
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## Perform on each of the worker nodes

- Join the node to the cluster:
```
$ sudo kubeadm join $controller_private_ip:6443 --token $token --discovery-token-ca-cert-hash $hash
```

- Verify that all nodes are joined and ready
```
$ kubectl get nodes
NAME                    STATUS   ROLES    AGE   VERSION
node1.example.com   	Ready    master   1m   v1.13.4
node2.example.com   	Ready    <none>   1m   v1.13.4
node3.example.com   	Ready    <none>   1m   v1.13.4
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
