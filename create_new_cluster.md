# Steps To Create a new kubemaster/cluster

## Initialize Cluster

`sudo kubeadm init --pod-network-cidr 10.244.0.0/16`

## Set up kubeconfig for user

Ensure logged in as root, `sudo su`

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install Pod Network Add-on

This should be committed as part of your cluster configuration, but we are using Flannel.

It can be manually installed with the following
`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

## Other

### kubeconfig

If running as root user can run
`export KUBECONFIG=/etc/kubernetes/admin.conf`

### Admin tokens for access, example

https://superuser.com/questions/1394619/how-to-get-the-admin-user-token-from-kubectl

Create user called k8sadmin
`sudo kubectl create serviceaccount k8sadmin -n kube-system`

Give the user admin privileges
`sudo kubectl create clusterrolebinding k8sadmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8sadmin`

Create token secret if k8s 1.25+

Show the contents of the secret to extract the token
`kubectl describe secret -n kube-system k8s-token`

Get the token
`sudo kubectl -n kube-system describe secret $(sudo kubectl -n kube-system get secret | (grep k8sadmin || echo "$_") | awk '{print $1}') | grep token: | awk '{print $2}'`

### Cluster Domain

By default `cluster.local` is used. kubeadm init param `--service-dns-domain` might change this.

Can check it in /etc/resolv.conf

Probably best to configure via CoreDNS ConfigMap changes when passing cluster state
https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/


More accurate check
`kubectl decribe configmap coredns -n kube-system`

## Addressing Issues

### Failed Cluster Init

If for whatever reason the cluster fails to initialize, you can start over with `kubeadm reset` then `kubeadm init` again.

However this may later lead to issues starting up some pods, stuck in `ContainerCreating` state.
Flannel may generate an error similar to `failed to set bridge addr: "cni0" already has an
IP address different from 10.245.1.1/24`

Resolution
https://github.com/kubernetes/kubernetes/issues/39557

```
kubeadm reset
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl start docker
systemctl start kubelet
```

Reboot for good measure.

### Generating join token and expiry

`-ttl 0` will create a non-expiring set of tokens. You can otherwise use a specified time range. The default is 24h.

### Manual Init

`kubeadm init --pod-network-cidr=10.244.0.0/16`

Flannel requires that the pod cidr is declared, this will match its config.

## Other Links

- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- https://phoenixnap.com/kb/how-to-install-kubernetes-on-a-bare-metal-server
