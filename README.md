# Kubernetes_HA_install
Use 7 VMs to create a Kubernetes_HA cluster

# Setup / Preporation
1. 7 VMs Ubuntu 18.04.1.0, 1 Loadbalanser, 3 master, 3 nodes.
2. Static IPs on individual VMs
3. /etc/hosts hosts file includes name to IP mappings for VMs or DNS
4. Swap is disabled
5. Take snapshots prior to installations, this way you can install and revert to snapshot if needed

# REPEAT FOR ALL 7 VMs

1. Disable swap, swapoff then edit your fstab removing any entry for swap partitions. You can recover the space with fdisk. You may want to reboot to ensure your config is ok.
~~~~
sudo swapoff -a
sudo vi /etc/fstab
~~~~

2. Add Google's apt repository gpg key **NB NB! Not needed on Loadbalanser**
~~~~
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
~~~~

3. Add the Kubernetes apt repository **NB NB! Not needed on Loadbalanser**
~~~~
sudo bash -c 'cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF'
~~~~

4. Update the package list and use apt-cache to inspect versions available in the repository
~~~~
sudo apt-get update
sudo apt-get upgrade 
~~~~

5. Install the required packages, if needed we can request a specific version **NB NB! On Loadbalanser only install docker.io rest is not needed.**
~~~~
sudo apt-get install -y docker.io kubelet kubeadm kubectl
sudo apt-mark hold docker.io kubelet kubeadm kubectl
~~~~

6. Ensure both are set to start when the system starts up.
~~~~
sudo systemctl enable kubelet.service
sudo systemctl enable docker.service
~~~~

# Create Loadbalancer

1. Create nginx config file
~~~~
mkdir /etc/nginx
wget https://raw.githubusercontent.com/knutia/Kubernetes_HA_install/master/nginx.conf
mv ./nginx.conf /etc/nginx/nginx.conf
sudo vi /etc/nginx/nginx.conf
~~~~
Replase <IP/DNSNAME_MASTER_1>, <IP/DNSNAME_MASTER_2> and <IP/DNSNAME_MASTER_3> whit the real ip or FQDN for the 3 masters.

2. Start nginx server using config file
~~~~
sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 -p 6443:6443 -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf nginx:1.14
~~~~

# Create First Master

1. Only on the master, download the yaml files for the pod network
~~~~
wget https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
wget https://docs.projectcalico.org/v3.11/manifests/calico.yaml
~~~~

2. Look inside calico.yaml and find the network range, adjust if needed.
~~~~
vi calico.yaml
~~~~

3. Create our kubernetes cluster, specifying a pod network range matching that in calico.yaml!
~~~~
sudo kubeadm init --pod-network-cidr=CHOSEN_CIDER_FOR_POD_NETWORK --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
~~~~
Example:*.
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint "192.168.2.190:6443" --upload-certs*.


4. Configure our account on the master to have admin access to the API server from a non-privileged account.
~~~~
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~~

5. Download yaml files for your pod network
~~~~
kubectl apply -f rbac-kdd.yaml
kubectl apply -f calico.yaml
~~~~

6. Gives you output over time, rather than repainting the screen on each iteration.
~~~~
kubectl get pods --all-namespaces --watch
~~~~