# k8s-onpremise-baremetal

**HOW INSTALL KUBERNETES IN CLOUD,VMWARE,BAREMETAL,ONPREMISE etc...**

This is my cluster k8s deployment. im usually deploy on my private server,instance cloud or even baremetal. I tried in my own way, hopefully useful. please mention me if useful hahaha just kidding. thanks



im using Kubernetes version 1.25, You can install any version kubernetes you like. And of course i installed it on my cloud server, but i test it too on my baremetal,onpremise,virtual private server and even Vmware!



You can use any distro linux. i test it using ubuntu and centos. i will update asap.



k8s-master1: minimal 1 unit server. (2-4VCPU with memory minimum 2GB or more)

k8s-worker1: minimal 1 unit server. (2-4VCPU with memory minimum 2GB or more)

k8s-worker2: 



For example in this cluster configuration like this:

| Name        | IP Address | Role                 |
| ----------- | ---------- | -------------------- |
| k8s-master1 | 10.184.0.2 | master,control plane |
| k8s-worker1 | 10.184.0.3 | worker/minion        |
| k8s-worker2 | 10.184.0.4 | worker/minion        |

**Update the master and worker.**

```bash
apt update && apt upgrade -y
```

And apply this tune.

```bash
tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
reboot
```

**Run this in all server**. Actually we don't need docker for build k8s. For some reason my hands always itch to install it. you can skip installing docker if you want.

```bash
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates  iputils-ping telnet

apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo apt update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

systemctl enable containerd
systemctl restart containerd
```

If you want to install specific version of kubernetes. You can type like this:



```bash
apt install kubeadm=1.25.9-00 kubectl=1.25.9-00 kubelet=1.25.9-00
```

As you can see before, im using apt-mark hold to prevent packages upgrade automatically by the system. Because i want to upgrading kubernetes flow is under my control.

Okay, now the part create network cluster. ***Run this command only once, and run only on master.***

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

You will get output like this.

```textile
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.184.0.2:6443 --token zv4ph6.mj2x64qhxycqb35w \
        --discovery-token-ca-cert-ha:01284d332019c6f8a9ff6f977e8db5f202as7hcaf4584f97f8e48354c0639e49
```

**Run this part on the master server only:**

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Run this part on the worker node server only**

```bash
kubeadm join 10.184.0.2:6443 --token zv4ph6.mj2x64qhxycqb35w \
        --discovery-token-ca-cert-ha:01284d332019c6f8a9ff6f977e8db5f202as7hcaf4584f97f8e48354c0639e49
```

That join code command you can get from last command when you run on the master...



**Network Manifest**

somepeople maybe love flannel or other. But in this cluster i prefer to choose Callico. So we will deploy Callico as our CNI (container network interface). Run this command in master.

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```

And run again :

```bash
kubectl get nodes
```

Makesure all server status is ready. Sometimes you need 1-2 minutes for server to ready. (it takes long when im using slow spec server).



**LABEL the server kubernetes (k8s)**

sometimes i give label too on my kubernetes server, like this.

```bash
kubectl label nodes k8s-master1 kubernetes.io/role=master
kubectl label nodes k8s-worker1 kubernetes.io/role=worker
kubectl label nodes k8s-worker2 kubernetes.io/role=worker
```

that name of server depend on your hostname server. My output like this:

<img src="https://i.ibb.co/HFxFHcg/ibrahimzfe-k8s.png" title="" alt="" width="354">



**Dashboard Kubernetes**

Now we will installing kubernetes dashboard on our cluster. Run this on master.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

After install kubernetes dashboard you can  not access directly because it is only running on internally. I prefer expose it as NodePort so i can access through my IP.

Run this on master:

```bash
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```

On the bottom change Type: ClusterIP as NodePort

<img src="https://i.ibb.co/6m1KysT/ibrahimzfe-kubernetes-nodeport.png" title="" alt="" width="277">

Type** :wq! **for save. 

You can running this command to check what port running...

```bash
kubectl get svc kubernetes-dashboard -n kubernetes-dashboard
```

*root@k8s-master1:~# kubectl get svc kubernetes-dashboard -n kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.103.3.221   <none>        443:30000/TCP   3d11h*

As you can see my dashboard running on port 30000. So you can access the dashboard now using address like this on the browser: **https://IPAddress:30000**

You can edit svc (last command) to custom port too if you want.Just change -NodePort on the bottom line

<img src="https://i.ibb.co/S541GYy/kubernetes-dashboard-ibrahimzfe.png" title="" alt="" width="629">

You will get warning for accessing https from the browser. (i will update the readme for securing https, but im so lazy right now)

Just prefer accept from the browser right now and select **Token **to login.

So how you login into dashboard kubernetes using token? This is how we doit.



**Create Token Kubernetes (k8s) To Login.**

```bash
kubectl create serviceaccount admin-dashboard
kubectl create clusterrolebinding admin-dashboard --clusterrole=cluster-admin --serviceaccount=default:admin-dashboard
```

And create this file using nano/vi.

```bash
nano token.yml
```

insert this line code YML.

```yml
apiVersion: v1
kind: Secret
metadata:
  name: admin-dashboard-token
  annotations:
    kubernetes.io/service-account.name: admin-dashboard
type:
kubernetes.io/service-account-token
```

Save the file and Apply the file:

```bash
kubectl apply -f token.yml
```

Describe the token:

```bash
kubectl describe secret admin-dashboard-token -n default
```

<img src="https://i.ibb.co/SRJgXMr/token-login-kubernetes-dashboard-ibrahimzfe.png" title="" alt="" width="500">

Now you can login using that token.

<img src="https://i.ibb.co/TRV44tN/kubernetes-dashboard-nodes-ibrahimzfe.png" title="" alt="" width="593">
