# ☸️ Kubernetes Cluster on Proxmox

A practical guide to building a **3-node Kubernetes cluster on Proxmox VE** using **Ubuntu Server 24.04.4**, `containerd`, `kubeadm`, `kubectl`, **Flannel**, and **MetalLB**.

The final setup will look like this:

```text
┌───────────────────────────────────────────────┐
│                  Proxmox VE                   │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │          Kubernetes Cluster             │  │
│  │                                         │  │
│  │  ┌──────────────┐                       │  │
│  │  │ k8s-master   │  Control Plane        │  │
│  │  │ 10.10.8.215  │                       │  │
│  │  └──────┬───────┘                       │  │
│  │         │                               │  │
│  │    ┌────┴─────┐                         │  │
│  │    │          │                         │  │
│  │ ┌──▼──────┐ ┌─▼────────┐                │  │
│  │ │ worker1 │ │ worker2  │                │  │
│  │ │ .216    │ │ .217     │                │  │
│  │ └─────────┘ └──────────┘                │  │
│  │                                         │  │
│  │              Flannel                    │  │
│  │                 │                       │  │
│  │              MetalLB                    │  │
│  │                 │                       │  │
│  └─────────────────┼───────────────────────┘  │
│                    │                          │
└────────────────────┼──────────────────────────┘
                     │
                     ▼
              10.10.8.240-245
              LoadBalancer IPs
```
------
## 📋 Table of Contents
### Architecture Prerequisites
1. Create the Ubuntu VMs
2. Configure Ubuntu
3. Disable Swap
4. Configure Kernel Modules
5. Configure Kernel Networking
6. Install containerd
7. Configure containerd
8. Install Kubernetes
9. Initialize the Control Plane
10. Configure kubectl
11. Install Flannel
12. Join the Worker Nodes
13. Verify the Cluster
14. Install MetalLB
15. Configure the MetalLB IP Pool
16. Deploy a Test Application
17. Test the LoadBalancer
-------
## 🏗 Architecture
```
| Node Name    | IP Address    | Role             |
| ---------    | ------------- | ---------        |
| k8s-master   | 10.10.8.215   | control Plane    |
| k8s-worker1  | 10.10.8.216   | worker1          |
| k8s-worker2  | 10.10.8.217   | worker2          |
```
---
## ✅ Prerequisites

### Before starting, make sure you have:

* A working Proxmox VE server
* Ubuntu Server 24.04.4 ISO
* Network connectivity between all three VMs
* Static IP addresses or DHCP reservations
* Internet access from the VMs
* SSH access to the VMs
* At least 2 CPU cores per VM recommended
* At least 4–6 GB RAM per VM recommended
* Sufficient disk space for Kubernetes workloads
* `Important: The Kubernetes Pod CIDR must not overlap with your physical network.`

This guide uses:
```
Pod network: 10.244.0.0/16
```
Your node network is:
```
10.10.8.0/24
```
## 1. 🖥 Create the Ubuntu VMs
Create three Ubuntu Server VMs from the Proxmox GUI.
```
Suggested configuration:
VM 1:
    Hostname: k8s-master
    IP:       10.10.8.215

VM 2:
    Hostname: k8s-worker1
    IP:       10.10.8.216

VM 3:
    Hostname: k8s-worker2
    IP:       10.10.8.217
```
### During Ubuntu installation:

* Create your administrative user
* Set a strong password
* Install OpenSSH Server
* Configure networking
* Set the hostname correctly
## 2. 🔧 Configure Ubuntu
After installation, open the Proxmox console for each VM and log in.
```
# Update the os
sudo apt update && sudo apt upgrade -y
sudo reboot
```
`make sure to install openssh during setup`:

Now - still in Proxmox - log on to each in console using the user/pass you've just configured and run: <br /> 

```
sudo apt update && sudo apt upgrade -y
```

Then when they finish `- run`:
```
sudo reboot
```
### After reboot, SSH into each VM.
```
# For Example
ssh your-user@10.10.8.215
```
Install the Proxmox QEMU guest agent on all `nodes`:
```
sudo apt install qemu-guest-agent -y
## Enable it
sudo systemctl enable --now qemu-guest-agent
## Check the status
systemctl status qemu-guest-agent
```
## 3. Disable Swap
Kubernetes expects swap to be disabled unless you intentionally configure kubelet for swap support.
```
sudo swapoff -a
```
Now edit `/etc/fstab:` and find the swap entry and comment it out.
```
sudo vi /etc/fstab
```
## 4. 🔩 Configure Kernel Modules
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
Load the modules immediately:
```
sudo modprobe overlay
sudo modprobe br_netfilter
## Verify 
lsmod | grep overlay
lsmod | grep br_netfilter
```
### What do these modules do?
* `overlay` Used by container storage drivers to create layered filesystems.

* `br_netfilter` Allows bridged network traffic to be processed by Linux networking/filtering mechanisms such as iptables.

## 5. 🌐 Configure Kernel Networking
```
# Create the Kubernetes sysctl configuration:

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
Apply the configuration:
```
sudo sysctl --system
# Verify IP forwarding:
sysctl net.ipv4.ip_forward 
```
## 6.  Install containerd
```
sudo apt install -y containerd
## Cheack Status
systemctl status containerd
```
## 7. ⚙️ Configure containerd
```
# Create the configuration directory:
sudo mkdir -p /etc/containerd
# Generate the default configuration:
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```
The configuration is now located at: `/etc/containerd/config.toml `
```
# Check the current cgroup setting:
grep SystemdCgroup /etc/containerd/config.toml
# Set it to true
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart Containerd and check the status
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd
```
## 8. ☸️ Install Kubernetes
Install the required packages:
```
sudo apt update

sudo apt install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gpg

# Create the Kubernetes keyring directory:
sudo mkdir -p -m 755 /etc/apt/keyrings

# Download the Kubernetes repository signing key:
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the repository:
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package information:
sudo apt update

# Install Kubernetes:
sudo apt install -y kubelet kubeadm kubectl

# Prevent automatic package upgrades:
sudo apt-mark hold kubelet kubeadm kubectl
```
verify : 
````
kubeadm version
kubectl version --client
kubelet --version
````
## 9. 🚀 Initialize the Control Plane
This command should be run only on `k8s-master`

```
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16
```
The initialization may take several minutes.

At the end, `kubeadm` will display a command similar to:
```
kubeadm join 10.10.8.215:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```
⚠️ Save this command!
If you lose the command, generate a new one:
```
kubeadm token create --print-join-command
```
Now SSH into each of the nodes :   

Now we can run:  
```
sudo apt install qemu-guest-agent -y
```
## Each command should now run for all VMs

Now we have to disable swap file as otherwise our kubelet service might behave unpredictibly 
```
sudo swapoff -a
sudo vi /etc/fstab
```
## 10. 👤 Configure kubectl
After `kubeadm` init completes, configure `kubectl` for your normal user.

Run these commands on k8s-master:
````
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf \
  $HOME/.kube/config

sudo chown $(id -u):$(id -g) \
  $HOME/.kube/config
````
## 11. 🕸 Install Flannel
Kubernetes requires a CNI (Container Network Interface) to provide Pod networking.

This guide uses Flannel.

On `k8s-master:`
````
kubectl apply -f \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
````
Check the Flannel pods:

```
kubectl get pods -n kube-flannel
```
## 12. 🔗 Join the Worker Nodes
Go to `k8s-worker` and run on both workers.
Run the join command generated by kubeadm init.
```
sudo kubeadm join 10.10.8.215:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```
## 13. 🔍 Verify the Cluster
Return to `k8s-master`.
```
kubectl get nodes
```

## 14. ⚖️ Install MetalLB
A standard bare-metal Kubernetes cluster does not automatically provide a cloud-provider-style LoadBalancer implementation.

For this lab, we can use `MetalLB`.

First, inspect the kube-proxy configuration:
```
kubectl -n kube-system edit configmap kube-proxy
```
If the configuration uses `IPVS and strictARP` is present, change it to:
```
ipvs:
  strictARP: true
```
### Install MetalLB
Install MetalLB using its official manifest.
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```
## 15. 🌐 Configure the MetalLB IP Pool
MetalLB needs a range of IP addresses that it can assign to Kubernetes `LoadBalancer` services.
```
vi proxmox-ip-pool.yaml
```
Add:
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: mainul-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.10.8.240-10.10.8.245

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: k8s-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - mainul-pool
```
Apply :
```
kubectl apply -f proxmox-ip-pool.yaml
```
Verify :
```
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```
## 16. 🧪 Deploy a Test static web Application
on k8s-master
```
mkdir project
cd project
git clone https://github.com/Mainul41561/retro-portfolio-mainul.git
```
`add:`
vi nginx-test.yaml
 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        kubernetes.io/hostname: k8s-master
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      initContainers:
      - name: copy-portfolio
        image: busybox:latest
        command: ['sh', '-c', 'cp -r /src-portfolio/* /target-html/']
        volumeMounts:
        - name: host-portfolio
          mountPath: /src-portfolio
        - name: shared-html
          mountPath: /target-html
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: host-portfolio
        hostPath:
          path: /home/mainul/project/retro-portfolio-mainul
          type: Directory
      - name: shared-html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```
Apply it:
```
kubectl apply -f nginx-test.yaml
```
Check the Pods:
```
kubectl get pods
```
## 17. 🌍 Test the LoadBalancer
```
kubectl get svc nginx-service
```
### Access your static website at this URL
```
http://10.10.8.240
```
![Portfolio Image](https://github.com/Mainul41561/Kubernetes-Cluster-on-Proxmox/blob/main/Screenshot%202026-08-19%20125747.png)

### cluster details output
![cluster](https://github.com/Mainul41561/Kubernetes-Cluster-on-Proxmox/blob/main/Screenshot%202026-08-20%20173554.png)


