
## Deploy Kubernetes Cluster Using Kubeadm
we will be building a k8s cluster with one control-plane(CP) / k8s-master node and a worker(W) node on Ubuntu 22.04 or later




## prerequisites

1- Two computers, which could be physical machines, virtual machines, cloud compute instances, or containers.

2- Ubuntu 22.04 or a later version installed on both computers

3-These two computers should be capable of communicating via a network, and firewalls should allow the necessary ports and protocols.

 *For testing purposes within a closed network, you can disable firewalls on both computers*

4-Both computers should have unique hostnames. You can use ‘hostnamectl’ to set up the hostnames

5-Both computers require internet access

6-Superuser or root access is needed on both computers

To verify system requirements, refer to the official Kubernetes documentation.




## Keys to This Demo

Short terms used in this article

CP — apply on control-plane node only

W —apply on worker node only

CP-W — apply on both control-plane and worker node


## System preparation & Configuration


1- **Check the port availability:**

`sudo netstat -tulpn | grep <port-to-check> `

*If the port is free, this command should not display any processes associated with the relevant port. If you observe any process associated with any of the below ports, you’ll need to free them up first. (We’re not discussing customizing ports in this article. We’ll be using the default set of Kubernetes ports)*

**command for control-plane / CP**

`sudo netstat -tulpn | grep "6443\|2379\|2380\|10250\|10259\|10257"`

**command for worker node / W**

` sudo netstat -tulpn | grep "10250" `

**ports — Extra information,**

#### Control Plane Node

kube-controller manager : 10257

kubernets API server : 6443

kubelet API : 10250

kube-scheduler : 10259

etcd API : 2379 / 2380

#### Worker Node

kubelet API : 10250

NodePort port range : 30000–32767


2-**Disable Ubuntu Firewall / CP-W**

`sudo ufw disable`

3- **Update and upgrade the system / CP-W**

```
sudo apt update

sudo apt upgrade -y

```

4- **Enable time-sync with an NTP server / CP-W**

```
sudo apt install systemd-timesyncd

sudo timedatectl set-ntp true

sudo timedatectl status

```

5- **Turn of the swap / CP-W**

```
sudo swapoff -a

sudo sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

free -m
```

*Check the fstab file as well. otherwise swap will be turned on automatically on reboot and you won’t be able to use k8s services properly* 

` cat /etc/fstab | grep swap `

6- **Configure required kernel modules / CP-W**

`sudo vim /etc/modules-load.d/k8s.conf `

*Add below content, save and close the file*

```
overlay
br_netfilter
```

7- **Load above modules to the current session**

```
sudo modprobe overlay 
sudo modprobe br_netfilter
```

*Check the status* 

`lsmod | grep "overlay\|br_netfilter"`


8- **Configure network parameters / CP-W**

Create k8s.conf file in /etc/sysctl.d

`sudo nano /etc/sysctl.d/k8s.conf `

Add below content, save and close the file

```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1

```

Then

`sudo sysctl --system`

9- **Install necessary software tools to continue / CP-W**

`sudo apt-get install -y apt-transport-https ca-certificates curl \
  gpg gnupg2 software-properties-common`


10- **Install Kubernetes Tools & containerd Runtime**

*Install Kubernetes Tools*

`sudo mkdir -m 755 /etc/apt/keyrings`

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

```



```
sudo apt update

sudo apt-get install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl

```


*Install containerd runtime*

```
sudo apt-get update

sudo apt-get install ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update


sudo apt install containerd.io

```


*Configuring ContainerD*

```
sudo mkdir -p /etc/containerd

sudo containerd config default | sudo tee /etc/containerd/config.toml
```


`sudo nano  /etc/containerd/config.toml`

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]  
runtime_type = "io.containerd.runc.v2" # <- note this, this line might have been missed  
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]  
SystemdCgroup = true # <- note this, this could be set as false in the default configuration, please make it true
```
*Configure and Start the service of ContainerD*

```
sudo systemctl restart containerd  
sudo systemctl enable containerd  
systemctl status containerd
```


*Testing*

 `sudo crictl ps`

 *If displaying warning, Try to do the following*

 `sudo apt install cri-tools`

 `sudo vim /etc/crictl.yaml`

 *Paste The following"

 ```
 untime-endpoint: unix:///run/containerd/containerd.sock  
image-endpoint: unix:///run/containerd/containerd.sock  
timeout: 2  
debug: false # <- if you don't want to see debug info you can set this to false  
pull-image-on-create: false

```

*Test Again*

`sudo crictl ps`

*Start The service of Kubelet*

`sudo systemctl enable kubelet`














## Initializing Control-Plane Node


**To Validate that there is no images pulled on your system**

```
sudo crictl images 
```

**To Pull necessary images**

```
sudo kubeadm config images pull --cri-socket unix:///var/run/containerd/containerd.sock 
```

**To Validate Images presence**

```
sudo crictl images
```

**To initialize Control Plane**

```
sudo kubeadm init \  
--pod-network-cidr=10.244.0.0/16 \  
--cri-socket unix:///var/run/containerd/containerd.sock \  
--v=5

```

**Do necessary Configurations**

```
mkdir -p $HOME/.kube  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Configure alias**
```
alias k="kubectl"
```
 *Add this line to .bashrc Then*
 ```
 source .bashrc
 ```

 **Testing**

```
kubectl get nodes

```

**Install Network plugin calico**

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml


curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml -O

```
**Edit it before apply to add 10.244.0.0/16 --> Pod Cidr Range
Then :**

```
    kubectl create -f custom-resources.yaml
```

**To Join WorkerNodes
use that command on Controlplane then paste on workernode**

```
kubeadm token create --print-join-command

```

**Installing Helm on controlplane**

```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update

sudo apt-get install helm -y

```


**Enable SSH keybased Auth between nodes**

```
ssh-keygen -t rsa -b 4096
ssh-copy-id user@target_machine

```

**Install ingress-controller**

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx 

helm repo update

kubectl create ns ingress-nginx

helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --set controller.service.type=NodePort --set controller.service.nodePorts.http=30080 --set controller.service.nodePorts.https=30443 --set controller.admissionWebhooks.enabled=true --set controller.admissionWebhooks.patch.enabled=true
```

**Validations & Testing:**

```
kubectl get svc -n ingress-nginx
kubectl get pods -n ingress-nginx

```

**Edit ingressclass and make it default**

```
kubectl edit ingressclass nginx

```

```
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"   --->  add this line
spec:
  controller: k8s.io/ingress-nginx

```

**Create your first pod and expose it using svc:**

```
kubectl run podname --image=nginx
kubectl expose pod podname --port=80

```


**Add ingress-resources by paste the following into yaml file** 


```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-name
            port:
              number: service-port-number
```

*Then*

```
kubectl apply -f filename.yaml

```
## Useful Links

- https://medium.com/@priyantha.getc/step-by-step-guide-to-creating-a-kubernetes-cluster-on-ubuntu-22-04-using-containerd-runtime-0ead53a8d273

- https://docs.docker.com/engine/install/ubuntu/

- https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises

- https://helm.sh/docs/intro/install/

