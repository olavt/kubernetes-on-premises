# Kubernetes Cluster Setup On Premises (hosted on Windows Server 2019 running Hyper-V) 
### Author: [Olav Tollefsen](https://www.linkedin.com/in/olavtollefsen/)

# Introduction

This article documents the process of installing a Kubernetes Cluster on from scratch in an on premises environment. In my case I'm hosting everything on Windows Server 2019 running Hyper-V.

# What you need

You will a minumum of 3 machines (physical or virtual) running Linux. I will be using Ubuntu Server 18.04.4 LTS. Make sure you allocate at leasdt 2 CPU-cores to each node (otherwise kubeadm init will complain).

# Installing Ubuntu Server 18.04.4 LTS

Don't select to install Docker as a part of the operating system installation, since we want to have full control over the version we will install.

# Prepare Linux nodes (Ubuntu)

## Update operating system

After installing the operating system, make sure it's updated.

```
$ sudo apt-get update
$ sudo apt-get upgrade
```

## Install Docker

### Set up the repository

Install packages to allow apt to use a repository over HTTPS:

```
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

Add Docker’s official GPG key:

```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Verify that you now have the key with the fingerprint 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88, by searching for the last 8 characters of the fingerprint.

```
$ sudo apt-key fingerprint 0EBFCD88
```

Set up the stable repository:

```
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

### Install a specific version of Docker (19.03.8)

Update the apt package index

```
$ sudo apt-get update
```

 List the versions available in your repo:

```
$ apt-cache madison docker-ce
```

Install Docker version 19.03.8:

```
$ sudo apt-get install docker-ce=5:19.03.8~3-0~ubuntu-bionic docker-ce-cli=5:19.03.8~3-0~ubuntu-bionic containerd.io
```

To be able to issue Docker commands without sudo:
```
$ sudo usermod -aG docker <your-username>
```

Logout from the current user and login again for the above command to take effect.

### Configure Docker

Check that IP forwarding is allowed

```
$ sysctl net.ipv4.ip_forward
```

Check the current iptables rules:

```
$ sudo iptables -L
```

Enable forwarding from Docker containers to the outside world (https://docs.docker.com/network/bridge/):

Edit /etc/rc.local

```
$ sudo nano /etc/rc.local
```

Add the following lines

```
#!/bin/sh -e
# Loop until 'docker version' exits with 0.
until docker version > /dev/null 2>&1
do
  sleep 1
done

iptables -P FORWARD ACCEPT
```

Make sure /etc/rc.local is executeable:

```
$ sudo chmod +x /etc/rc.local
```

Create a file named /etc/docker/daemon.json

```
$ sudo nano /etc/docker/daemon.json
```

Add the following content to the file:

```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

```
$ sudo mkdir -p /etc/systemd/system/docker.service.d
```

## Configure a static IP address for the node

Edit the /etc/netplan/50-cloud-init.yaml file:

```
$ sudo nano /etc/netplan/50-cloud-init.yaml
```

Replace the following line:

```
            dhcp4: true
```

with:

```
            dhcp4: no
            addresses: [172.20.5.1/16]
            gateway4: 172.20.1.1
            nameservers:
              addresses: [8.8.8.8,8.8.4.4]
```

Apply changes:

```
$ sudo netplan apply
```

### Reboot

```
$ sudo reboot now
```

## Disable swap (required for Kubernetes)

Check if swap is enabled:

```
$ sudo swapon --show
```

If the above command doesn't produce any output swap is already disabled and you can skip the next steps.

Deactivate swap space:

```
$ sudo swapoff -v /swap.img
```

Remove (comment out) the swap file entry from /etc/fstab:

```
$ sudo nano /etc/fstab
```

Delete the actual swapfile file

```
$ sudo rm /swap.img
```

 # Setup Kubernetes

## Install required Kubernetes software (latest version)
```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
  echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
  sudo apt-get update -q && \
  sudo apt-get install -qy kubeadm
```

or

## Install required Kubernetes software (specific version)

If you want to install a specific version of Kubernets, the installation command looks like this:

```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
  echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
  sudo apt-get update -q && \
  sudo apt-get install -qy kubelet=1.9.4-00 kubectl=1.9.4-00 kubeadm=1.9.4-00
```

Replace the version in the command above with the version you want to install. To see a list of the available version numbers, you can issue the following command:
```
curl -s https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep Version | awk '{print $2}'
```

## Prepare the Master node

### Initializing your Master

Perform this operation on the node that you want to designate as the Master.

For Flannel networking:
```
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

It is recommended to enable bridged IPv4 traffic to iptables chains when using Flannel:

```
$ sudo sysctl net.bridge.bridge-nf-call-iptables=1 (Note! Run this on all Linux nodes)
```

or

For Wave networking:

```
$ sudo kubeadm init
```

This step will take a while.

Take a note of the node-join information printed at the end of this command. You will need that information when you join the other machines (Nodes) to the cluster.

### Prepare for running kubectl on the Master
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install networking for Kubernetes

Note! In order to support Windows nodes in the cluster, use Flannel.

Flannel (latest version):

Download the most recent Flannel manifest:
```
$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Note: The VNI must be set to 4096 and port 4789 for Flannel on Linux to interoperate with Flannel on Windows.

Modify the net-conf.json section of the flannel manifest in order to set the VNI to 4096 and the Port to 4789. It should look as follows:

```
net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "VNI" : 4096,
        "Port": 4789
      }
    }
```

Apply the Flannel manifest and validate:
```
$ kubectl apply -f kube-flannel.yml
```

or

Weave (latest version):
```
$ kubectl apply -f \
 "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

or (Weave specif version)

```
$ kubectl apply -f https://git.io/weave-kube-1.6
```

### Check status

You should see all services up and running (after giving it a few minutes after installing the network).
```
$ kubectl get pods --namespace=kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-66bff467f8-9c7sj                1/1     Running   0          26m
coredns-66bff467f8-vsnqc                1/1     Running   0          26m
etcd-k01-master-01                      1/1     Running   0          26m
kube-apiserver-k01-master-01            1/1     Running   0          26m
kube-controller-manager-k01-master-01   1/1     Running   3          26m
kube-proxy-9z88k                        1/1     Running   0          9m12s
kube-proxy-gvnm5                        1/1     Running   0          25m
kube-proxy-vp7dc                        1/1     Running   0          26m
kube-scheduler-k01-master-01            1/1     Running   1          26m
weave-net-7pdgh                         2/2     Running   0          19m
weave-net-gktdl                         2/2     Running   1          9m12s
weave-net-rdn9q                         2/2     Running   0          19m
```

## Add support for adding Windows nodes (can be skipped for Linux only cluster)

### Add Windows Flannel and kube-proxy DaemonSets

Add Windows-compatible versions of Flannel and kube-proxy. In order to ensure that you get a compatible version of kube-proxy, you’ll need to substitute the tag of the image. The following example shows usage for Kubernetes v1.18.1, but you should adjust the version for your own deployment.

```
$ curl -L https://github.com/kubernetes-sigs/sig-windows-tools/releases/latest/download/kube-proxy.yml | sed 's/VERSION/v1.18.1/g' | kubectl apply -f -

$ kubectl apply -f https://github.com/kubernetes-sigs/sig-windows-tools/releases/latest/download/flannel-overlay.yml
```

You have now performed the steps required to setup the Master.

## Join a Linux Node to the cluster

Remember to perform all the common preparation steps first!

Perform this for every machine you want to join as a Node to the Kubernetes cluster. NOTE! Replace the command below with the information you obtained when running "kubeadm init" (see below if more than 24 hours passed since you created the cluster or if you did not record the information).

```
$ sudo kubeadm join --token 218b7b.1f188d49758886cb 192.168.1.10:6443 --discovery-token-ca-cert-hash sha256:9b8fc6dcc53e2af8dc1c9093c6b3354f4767a644c1dd9dfeebc19c3c04bd6f17
```

### Joining a Node to the cluster if the join-token has expired

Alternative A: Issue the following command to generate a new token and see the command to use to join a new node to the cluster:

```
$ sudo kubeadm token create --print-join-command
```

Aletrnative B: Generate a new token by issuing the following command on the Master:

```
$ sudo kubeadm token create
```

This is the value you pass to the --token part of the command.

If you did not record the information from "kubeadm init" and need to get the value for the --discovery-token-ca-cert-hash, you can issue this command at the Master:

```
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

### Check the status of the nodes in the cluster

Run the following command on the Master

```
$ kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
k02-master-01   Ready    master   83m   v1.18.1
k02-node-01     Ready    <none>   14m   v1.18.1
```

## Adding a Windows node to the cluster

Run Windows Update until all updates are install first.

### Install Docker Engine - Enterprise

To install the Docker Engine - Enterprise on your hosts, Docker provides a OneGet PowerShell Module.

1. Open an elevated PowerShell command prompt, and type the following commands.

```
Install-Module DockerMsftProvider -Force
Install-Package Docker -ProviderName DockerMsftProvider -Force
```

2. Restart the server

3. Test your Docker Engine - Enterprise installation by running the hello-world container.

```
docker run hello-world:nanoserver
```

### Install Kubernetes software

Download the installation script and run it in an elevated PowerShell command window:
```
curl.exe -LO https://github.com/kubernetes-sigs/sig-windows-tools/releases/latest/download/PrepareNode.ps1
```

Replace the version number below with the version number you are using for Kubernetes in the cluster.

```
.\PrepareNode.ps1 -KubernetesVersion {{1.18.1}}
```

### Run kubeadm init

Run the kubeadmin -init command (use the command "kubeadm token create --print-join-command" on the master to see the command)


# Making Kubernetes cluster ready for deployment

## Install Helm

Install Helm on your Master node in order to be able to deploy to the cluster using Heml

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

### Add the official Helm stable charts repository
```
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com
```

### Update Helm repositories

```
$ helm repo update
```

## Create a secret for pulling the image from a private registry

If you want to be able to pull an image from a private Docker registry, you need to create a secret for it.

```
$ kubectl create secret docker-registry <your-secret-name> --docker-server=<your-docker-server> --docker-username=<your-username> --docker-password=<your-password> --docker-email=<your-email>
```

## Configure HTTPS Ingress using Nginx

### Install the NGINX ingress controller using Helm

```
$ kubectl create namespace ingress-nginx

$ helm install --set controller.service.type=NodePort nginx-ingress stable/nginx-ingress --namespace ingress-nginx
```

Check that the nginx services are configures:

```
$ kubectl get service --namespace ingress-nginx
NAME                            TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-controller        LoadBalancer   10.0.71.15   40.119.152.40   80:30528/TCP,443:32283/TCP   40s
nginx-ingress-default-backend   ClusterIP      10.0.38.76   <none>          80/TCP                       40s
```

Check the details of the nginx ingress controller in order to see port details

```
$ kubectl describe  service nginx-ingress-controller  --namespace ingress-nginx
```

Look for these lines:
```
NodePort:                 http  32320/TCP
NodePort:                 https  31710/TCP
```

These are the port number you need to redirect traffic to in order to hit the ingress. The source ports should be 80 and 443. The destination IP address could be any of the nodes in the cluster. 

## Install cert-manager (loosely based upon the work of kube-lego)

Run the following script to install cert-manager:

```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.yaml
```

## Create a CA cluster issuer

Before certificates can be issued, cert-manager requires an Issuer or ClusterIssuer resource. These Kubernetes resources are identical in functionality, however Issuer works in a single namespace, and ClusterIssuer works across all namespaces. For more information, see the cert-manager issuer documentation.
Create a cluster issuer, such as cluster-issuer.yaml, using the following example manifest.

Note! Update the email address with a valid address from your organization:
```
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: <Your-email-address>
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    # An empty 'selector' means that this solver matches all domains
    - selector: {}
      http01:
        ingress:
          class: nginx
```

Create CA Cluster Issues resource with the kubectl apply command:

```
$ kubectl apply -f cluster-issuer.yaml
```

## Create an ingress route (for Nginx)

This step assumes that you already have deployed some service (web application) to test with.

Create a test-ingress.yaml file with the following content:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"  # To avoid 502 Bad Gateway when using Azure AD authentication
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - <Your FQN domain name>
    secretName: tls-secret
  rules:
  - host: <Your FQN domain name>
    http:
      paths:
      - path: /
        backend:
          serviceName: <YourTestService>
          servicePort: 80
```

Create the ingress resource with the kubectl apply command:

```
$ kubectl apply -f test-ingress.yaml
```

Now you should be able to browse to https://<YourDNSName>.westeurope.cloudapp.azure.com

## Check generated certificates

To verify that the certificate was created successfully, issue the following command:
```
$ kubectl describe certificate tls-secret
```

If the certificate was issued, you will see output similar to the following:
```
Events:
  Type    Reason        Age    From          Message
  ----    ------        ----   ----          -------
  Normal  GeneratedKey  26m    cert-manager  Generated a new private key
  Normal  Requested     26m    cert-manager  Created new CertificateRequest resource "tls-secret-3533869916"
  Normal  Issued        7m17s  cert-manager  Certificate issued successfully
```

### Troubleshooting certificate requests

See this article for troubleshootin: https://cert-manager.io/docs/faq/acme/

# Useful Kubernetes admin commmands

## Drain node in preparation for maintenance or deletion

The given node will be marked unschedulable to prevent new pods from arriving. Issue the following command to safely evict all of your pods from the node:

```
$ kubectl drain --ignore-daemonsets <Node> 
```

## Make the node schedulable again

```
$ kubectl uncordon <Node> 
```

## Remove a node from the cluster

When the pods have been deleted and are up and running on other nodes in the cluster, delete the node with this command:

```
$ kubectl delete node <Node> 
```

# Other useful stuff

## Mount Windows Server SMB share on Linux machine

Enable Gues account on Windows Server machine first, then issue these commands on the Linux machine

Install the cifs-utils package:

$ sudo apt-get install cifs-utils

$ sudo mkdir /mnt/photos

$ sudo mount -t cifs //tiger.safari.webinnovation.no/photos /mnt/photos -o username=Guest,password=

To make the mount permanent add it to /etc/fstab

To (re)mount all entries listed in /etc/fstab:

$ sudo mount -a

