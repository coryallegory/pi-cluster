# Prepare a master/worker node

## Install OS with SSH

- Ubuntu Server 20 LTS (focal)
- `openssh` is installed

https://ubuntu.com/download/raspberry-pi
- use raspberry pi imager
- choose ubuntu 20 64-bit

Machine will initially be named `ubuntu` with ssh and dhcp enabled.
- Change `ubuntu` password on initial login

## Name Machine

- Ensure machine name is set, `hostname`
  - `sudo hostnamectl set-hostname <currentname>`
- Ensure domain is configured in `/etc/hosts`
  - Add/Edit a line such as
    - `127.0.1.1 <prevname>`
  - to
    - `127.0.1.1 <currentname>.<domain> <currentname>`

## Assign IP Address

- Set static IP address in 192.168.8.X range, use netplan
  - check adapter name is eth0 or other etc, `ip link`
  - `sudo vi /etc/netplan/50-cloud-init.yaml`
  - Set dhcp4 to `false`, settings as follows

```
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: false
            addresses:
                - 192.168.101.80/24
            gateway4: 192.168.101.1
            nameservers:
                addresses: [8.8.8.8, 1.1.1.1]
```

  - `sudo netplan apply`

  You'll be disconnected, reconnect.

### Disable swap

- `sudo swapoff -a`
Run the following to update dstab so swap stays off after reboot
- `sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`

### Install Docker

- `sudo apt-get update`
- `sudo apt-get install -y docker.io`

Enable docker service
- `sudo systemctl enable docker.service`

Set up the Docker daemon
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

`mkdir -p /etc/systemd/system/docker.service.d`

Restart Docker
`sudo systemctl daemon-reload`
`sudo systemctl restart docker`

### Install Kubernetes Tooling

Add kubernetes repo
`curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`
```
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
`sudo apt-get update && sudo apt-get upgrade`
`sudo apt-get install -y kubelet=1.19.0-00 kubeadm=1.19.0-00 kubectl=1.19.0-00`
`sudo apt-mark hold kubelet kubeadm kubectl`

# Cgroup memory

Append `cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1` to the end of `/boot/firmware/cmdline.txt` and `reboot`

`cat /proc/cgroups` to check for memory process running.
