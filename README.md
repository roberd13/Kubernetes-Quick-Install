# Kubernetes-Quick-Install
Start with at least a 3 node cluster.  I used Openstack c3.2xlarge instances


# Install Docker on all of your nodes

On each of your machines, install Docker. Version 17.03 is recommended, but 1.11, 1.12 and 1.13 are known to work as well. Versions 17.06+ might work, but have not yet been tested and verified by the Kubernetes node team. Keep track of the latest verified Docker version in the Kubernetes release notes.

Install Docker from Ubuntu’s repositories:

```
apt-get update
apt-get install -y docker.io
or install Docker CE 17.03 from Docker’s repositories for Ubuntu or Debian:

apt-get update
apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
```

# Install Kubeadm on all of your nodes

The "sudo su -" changes user to root.

```
sudo su -
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```

# Add a kubernetes repository for the latest stable one for the ubuntu flavor on the machine (here:xenial)

```
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
 
apt-get update
``` 

 
# To install latest version of Kubenetes packages. (recommended)

`apt-get install -y kubelet kubeadm kubectl`
 
 
# Verify version

```
kubectl version
kubeadm version
kubelet --version
``` 

# Append the following lines to ~/.bashrc to enable kubectl and kubeadm command auto-completion

```
echo "source <(kubectl completion bash)">> ~/.bashrc
echo "source <(kubeadm completion bash)">> ~/.bashrc
``` 
# Choose a Pod Networking Addon so the pods can communicate

A "Pod network" must be deployed to use the cluster. This will let pods to communicate with eachother.

There are many different pod networks to choose from. See https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network

I used Flannel 

For flannel to work correctly, you must pass --pod-network-cidr=10.244.0.0/16 to kubeadm init.

and

Set /proc/sys/net/bridge/bridge-nf-call-iptables to 1 by running `sysctl net.bridge.bridge-nf-call-iptables=1` to pass bridged IPv4 traffic to iptables’ chains. This is a requirement for some CNI plugins to work, for more information please see here.


# Initialize your Master (only on the node you designate as master) 

```
kubeadm init --pod-network-cidr=10.244.0.0/16
```

Once the initializition is complete install your Pod Network addon *Flannel in used here* 

`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml`

Note that flannel works on amd64, arm, arm64 and ppc64le, but until flannel v0.11.0 is released you need to use the following manifest that supports all the architectures:

`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/c5d10c8/Documentation/kube-flannel.yml`

To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

If you are root you can run 
`export KUBECONFIG=/etc/kubernetes/admin.conf`
I also added it to my `.bash.rc`  so as to not have to export it on reboots

# Add Your Nodes to the Cluser

On all your nodes but the master run the kubeadm command you received at the end of master initilization

```
kubeadm join some_ip --token some_token --discovery-token-ca-cert-hash sha256:some_token
```

If you want to schedule pods on your master run the following command

`kubectl taint nodes --all node-role.kubernetes.io/master-`

Now you have a working Kubernetes Cluster

If you want to play here are some commands to get to know your cluster 

```
kubectl get node
kubectl get secret
kubectl config view
kubectl config current-context
kubectl get componentstatus
kubectl get clusterrolebinding --all-namespaces
kubectl get serviceaccounts --all-namespaces
kubectl get pods --all-namespaces -o wide
kubectl get services --all-namespaces -o wide
kubectl cluster-info
 ```


# Install the Kubernetes Dashboard

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml`

# Create service account to use a token to login to your  Kubernetes Dashboard


Create Service Account and ClusterRoleBinding 

We are creating Service Account with name admin-user in namespace kube-system

```
cat > admin-user-serviceaccount.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system 
EOF
```
Create a ServiceAccount and ClusterRoleBinding based on the created file.

`sudo kubectl create -f admin-user-serviceaccount.yaml`

# Allow Access to your Dashboard from Outside the Cluster

Edit kubernetes-dashboard service.

`kubectl -n kube-system edit service kubernetes-dashboard`

You should see yaml representation of the service. Change type: ClusterIP to type: NodePort and save file. 

```
apiVersion: v1
...
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "343478"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard-head
  uid: 8e48f478-993d-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.100.124.90
  externalTrafficPolicy: Cluster
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP <--- Change to NodePort
status:
  loadBalancer: {}
 ```

Next we need to check port on which Dashboard was exposed.

`kubectl -n kube-system get service kubernetes-dashboard`

You should see output like so 
```
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   10.100.124.90   <nodes>       443:31707/TCP   21h
```
In this example Dashboard has been exposed on port 31707 (HTTPS). 

To find the node your Dashboard is running on run the following command.

`kubectl get pods --all-namespaces --show-all -o wide | grep dashboard`

Now you can access it from your browser at: https://node-ip:nodePort

Now we need to find the token we can use to log in. Execute following command:

`kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')`
It should print something like:

```
Name:         admin-user-token-6gl6l
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=b16afba9-dfec-11e7-bbb9-901b0e532516

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZnbDZsIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiMTZhZmJhOS1kZmVjLTExZTctYmJiOS05MDFiMGU1MzI1MTYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.M70CU3lbu3PP4OjhFms8PVL5pQKj-jj4RNSLA4YmQfTXpPUuxqXjiTf094_Rzr0fgN_IVX6gC4fiNUL5ynx9KU-lkPfk0HnX8scxfJNzypL039mpGt0bbe1IXKSIRaq_9VW59Xz-yBUhycYcKPO9RM2Qa1Ax29nqNVko4vLn1_1wPqJ6XSq3GYI8anTzV8Fku4jasUwjrws6Cn6_sPEGmL54sq5R4Z5afUtv-mItTmqZZdxnkRqcJLlg2Y8WbCPogErbsaCDJoABQ7ppaqHetwfM_0yMun6ABOQbIwwl8pspJhpplKwyo700OSpvTT9zlBsu-b35lzXGBRHzv5g_RA
```

On your Dashboard copy the token and paste it into Enter token field on the log in screen. 
