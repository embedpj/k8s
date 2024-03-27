*****Below instructions are taken from many online resouces over a perion of 1 week. Thanks to everyone on Internet******
*****Aim of this project*******
1) Set up one master and 3 worker on windows laptop running VirtualBox running UBUNTU server
2) Install latest Kubernetes version
3) Download Nginx from Github
4) Modify Nginx index.html and upload to personal github
5) 

******************************

*********set the VM networking to "bridge adapter" and keep the default ip

*******set host name for master & worker

sudo hostnamectl set-hostname "master-node"
sudo hostnamectl set-hostname "worker1-node"
sudo hostnamectl set-hostname "worker2-node"
sudo hostnamectl set-hostname "worker3-node"


****To refresh shell afer above
exec bash


*******Map ip address with names
sudo nano /etc/hosts

*******Host table
192.168.68.127 master-node
192.168.68.125 worker1-node
192.168.68.127 worker2-node
192.168.68.129 worker3-node

********ping master-node from 1 or 2 worker node

*******turn swap off and make it stay after reboot    

sudo swapoff -a 
free -h

sudo nano /etc/fstab
*******Comment Swap ***********
#/swap.img       none       swap       sw       0       0

sudo mount -a && free -h

sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF


sudo modprobe overlay
sudo modprobe br_netfilter


sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

*********Containerd run time

sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd


***********Install Kubernetes  
*********This must be the latest Kubernetes version

sudo mkdir /etc/apt/keyrings

*********Let’s fetch the public key from Google and store it in the folder we created in the previous step. 
*********This key is important to verify that the Kubernetes packages we download are genuine and haven't been tampered with.

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

**********Next, we need to tell the apt package manager where to find Kubernetes packages for downloading.


echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

#Only for master node - Start the kubeadm server

sudo kubeadm init \
  --pod-network-cidr=10.10.0.0/16 \
  --control-plane-endpoint=master-node    *--*our host name

#from instructions in startup.....
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#Please copy the join instructions
------------------------------------------------------------------
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join master-node:6443 --token r6whxl.zvok1oldzghydhj5 \
        --discovery-token-ca-cert-hash sha256:0ae2a4df389cc6c8a318bb46a448fcc279fae6f23a64c8df25800f95450a5a37 \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

sudo kubeadm join master-node:6443 --token 1nehga.tu2ekk3eg64q6zpd --discovery-token-ca-cert-hash sha256:848a1f4248c874b0225048f68aa1ab3e937173dcf9432d97ade6e0e79b810b80 --ignore-preflight-errors=Port-10250
--------------------------------------------------------------------------------

kubectl cluster-info
kubectl get nodes


#Go to worker node and join the workers

**************Please use your join key*********************
kubeadm join master-node:6443 --token r6whxl.zvok1oldzghydhj5 \
        --discovery-token-ca-cert-hash sha256:0ae2a4df389cc6c8a318bb46a448fcc279fae6f23a64c8df25800f95450a5a37

#check node status

kubectl get nodes

NAME                          STATUS     ROLES           AGE   VERSION
k8s-master.nvtienanh.local    NotReady   control-plane   58m   v1.26.1
k8s-worker1.nvtienanh.local   NotReady   <none>          87s   v1.26.1
k8s-worker2.nvtienanh.local   NotReady   <none>          44s   v1.26.1

#we have to install Calico pod network in Master

curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O

#open the calico.yaml file using nano and search for CALICO_IPV4POOL_CIDR
#update as below
-----------
# The default IPv4 pool to create on startup if none exists. Pod IPs will be
# chosen from this range. Changing this value after installation will have
# no effect. This should fall within `--cluster-cidr`.
- name: CALICO_IPV4POOL_CIDR
  value: '10.10.0.0/16'
# Disable file logging so `kubectl logs` works.
- name: CALICO_DISABLE_FILE_LOGGING
  value: 'true'

-------------------------
#save the nano file

kubectl apply -f calico.yaml  *Do not use sudo*

#it takes time to start......

k8smaster@master-node:~$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-658d97c59c-qqf7x   1/1     Running   0          2m4s
calico-node-28tqg                          1/1     Running   0          2m4s
calico-node-vqdg2                          1/1     Running   0          2m4s
calico-node-z8j65                          1/1     Running   0          2m4s
coredns-76f75df574-kmt65                   1/1     Running   0          39m
coredns-76f75df574-qm6jh                   1/1     Running   0          39m
etcd-master-node                           1/1     Running   0          39m
kube-apiserver-master-node                 1/1     Running   0          39m
kube-controller-manager-master-node        1/1     Running   5          39m
kube-proxy-22hpn                           1/1     Running   0          19m
kube-proxy-jqhmq                           1/1     Running   0          19m
kube-proxy-xx82r                           1/1     Running   0          39m
kube-scheduler-master-node                 1/1     Running   5          39m

k8smaster@master-node:~$ kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
master-node    Ready    control-plane   39m   v1.29.3
worker1-node   Ready    <none>          19m   v1.29.3
worker2-node   Ready    <none>          19m   v1.29.3




***********Note - Incase of incorrect install commands to completely remove kubeadm***************
kubeadm reset
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*   
sudo apt-get autoremove  
sudo rm -rf ~/.kube

************Clean cluster to join to new master*********************

sudo rm /etc/kubernetes/kubelet.conf
sudo rm /etc/kubernetes/pki/ca.crt
sudo systemctl restart kubelet

***************create NGINX deployment nginx.yaml***********

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: embedpj/nginxr
        ports:
        - containerPort: 80


2) Apply the new YAML file
kubectl apply -f nginx.yaml

3) Expose the service to open the port. This will open a random port which can be checked below.
kubectl expose deployment nginx-deployment --type=NodePort --port=80 

4) Check the service
k8smaster@master-node:~$ kubectl get services
NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.96.0.1      <none>        443/TCP        77m
nginx-deployment   NodePort    10.101.78.51   <none>        80:30706/TCP   6s


*************To find the IP address to unblock on windows machine

kubectl get svc nginx-deployment
NAME               TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-deployment   NodePort   10.99.138.94   <none>        80:30145/TCP   4h30m

************unblock 30145 in windows firewall.

access from any PC using the master IP address

nginx-deploy-86dcfdf4c6-m9xnb

kubectl delete all --all --namespace default

kubectl get services kube-dns --namespace=kube-system
kubectl get secrets





**********************DOCKER*********************
AIM
1) To edit nginx on docker desktop using custom index.html
2) Create Docker image
3) Test docker image locally
4) Move image to docker hub
5) Download the image to Kubernetes cluster
*************************************************


*************docker file**********
FROM nginx:stable-alpine
RUN rm -rf /usr/share/nginx/html/*
COPY ./dist/app /usr/share/nginx/html




*********index.html*********** 
path    /dist/app


<!doctype html>
<html>
 <body style="backgroud-color:rgb(49, 214, 220);"><center>
    <head>
     <title>Prabhu Projects</title>
    </head>
    <body>
     <p>Welcome to my Docker Project!<p>
     <p>*****Tell Me What You think**********<p>
        <p>Today's Date and Time is: <span id='date-time'></span><p>
        <script>
             var dateAndTime = new Date();
             document.getElementById('date-time').innerHTML=dateAndTime.toLocaleString();
        </script>
        </body>
</html>
   
*********On docker desktop************* 
docker build -t nginxr:latest .

docker run -d -p 8080:80 nginxr

*****Instruction to push from Docker desktop to online Dockerhub***********
$docker tag nginxr embedpj/nginxr
$docker push embedpj/nginxr
