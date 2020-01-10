# treafik-on-k8s
prereqs - you need to install vagrant and virtual box to get it running.

clone the repo and get the vm up by 
```shell
vagrant up 
Bringing machine 'ubuntuvm01' up with 'virtualbox' provider...
==> ubuntuvm01: Checking if box 'ubuntu/bionic64' version '20200107.0.0' is up to date...
==> ubuntuvm01: Clearing any previously set network interfaces...
    ubuntuvm01: Guest Additions Version: 5.2.34
    ubuntuvm01: VirtualBox Version: 6.0
```
once your vm is up , you can get login to the vagrant box by - vagrant ssh 

https://linuxcontainers.org/lxd/introduction/ - please go though the link to get some insight on lxc containers.

LXD isn't a rewrite of LXC, in fact it's building on top of LXC to provide a new, better user experience. Under the hood, LXD uses LXC through liblxc and its Go binding to create and manage the containers.

It's basically an alternative to LXC's tools and distribution template system with the added features that come from being controllable over the network

First we are going to create our k8s cluster in LXC containers and then we wil implement ingress controller Treafik on it.

### steps
```shell
vagrant@ubuntuvm01:~$ sudo systemctl start lxd
vagrant@ubuntuvm01:~$ sudo lxd init # initilze the daemon 

Would you like to use LXD clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: 
Name of the storage backend to use (btrfs, dir, lvm) [default=btrfs]: dir 
Would you like to connect to a MAAS server? (yes/no) [default=no]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: 
What should the new bridge be called? [default=lxdbr0]: 
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
Would you like LXD to be available over the network? (yes/no) [default=no]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes] 
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: 
```
# Create profile for k8s conatiners  
```shell
vagrant@ubuntuvm01:~$ sudo lxc list 
+------+-------+------+------+------+-----------+
| NAME | STATE | IPV4 | IPV6 | TYPE | SNAPSHOTS |
+------+-------+------+------+------+-----------+
To check what profile currently being used.

vagrant@ubuntuvm01:~$ sudo lxc profile list 
+---------+---------+
|  NAME   | USED BY |
+---------+---------+
| default | 0       |
+---------+---------+

copy the current profile , so that we can add our changes and create the config for k8s.

vagrant@ubuntuvm01:~$ sudo lxc profile copy default k8s

# Edit recently created config , which is k8s and replace all the contents from the k8s-profile-config from the repo.
vagrant@ubuntuvm01:~$ sudo lxc profile edit k8s

# after save you can verify list one again and this time you will see 2 names on the profile list. 
vagrant@ubuntuvm01:~$ sudo lxc profile list k8s
+---------+---------+
|  NAME   | USED BY |
+---------+---------+
| default | 0       |
+---------+---------+
| k8s     | 0       |
+---------+---------+
```
# Now we are going to create lxc containers and going to name as kmaster and worker and worker2
```shell
vagrant@ubuntuvm01:~$ sudo lxc launch images:centos/7 kmaster --profile k8s
Creating kmaster
Starting kmaster                            
vagrant@ubuntuvm01:~$ sudo lxc launch images:centos/7 kworker1 --profile k8s
Creating kworker1
Starting kworker1
vagrant@ubuntuvm01:~$ sudo lxc launch images:centos/7 kworker2 --profile k8s
Creating kworker2
Starting kworker2
 # verify them 
$ sudo lxc list 
+----------+---------+----------------------+-----------------------------------------------+------------+-----------+
|   NAME   |  STATE  |         IPV4         |                     IPV6                      |    TYPE    | SNAPSHOTS |     |
+----------+---------+----------------------+-----------------------------------------------+------------+-----------+
| kmaster  | RUNNING | 10.44.199.137 (eth0) | fd42:17ea:2f9b:7d3f:216:3eff:fe68:62e7 (eth0) | PERSISTENT | 0         |
+----------+---------+----------------------+-----------------------------------------------+------------+-----------+
| kworker1 | RUNNING | 10.44.199.49 (eth0)  | fd42:17ea:2f9b:7d3f:216:3eff:fe9b:1b1a (eth0) | PERSISTENT | 0         |
+----------+---------+----------------------+-----------------------------------------------+------------+-----------+

```
# Bootstrap these containers , which wil install k8 components and docker engine.
```shell
vagrant@ubuntuvm01:~/kubernetes/lxd-provisioning$ cat bootstrap-k8s.sh | sudo lxc exec kmaster bash
[TASK 1] Install docker container engine
[TASK 2] Enable and start docker service
[TASK 3] Add yum repo file for kubernetes
[TASK 4] Install Kubernetes (kubeadm, kubelet and kubectl)
[TASK 5] Enable and start kubelet service
[TASK 6] Install and configure ssh
[TASK 7] Set root password
[TASK 8] Install additional packages
[TASK 9] Initialize Kubernetes Cluster
[TASK 10] Copy kube admin config to root user .kube directory
[TASK 11] Deploy flannel network
[TASK 12] Generate and save cluster join command to /joincluster.sh

vagrant@ubuntuvm01:~/kubernetes/lxd-provisioning$ cat bootstrap-kube.sh | sudo lxc exec kworker1 bash
[TASK 1] Install docker container engine
[TASK 2] Enable and start docker service
[TASK 3] Add yum repo file for kubernetes
[TASK 4] Install Kubernetes (kubeadm, kubelet and kubectl)
[TASK 5] Enable and start kubelet service
[TASK 6] Install and configure ssh
[TASK 7] Set root password
[TASK 8] Install additional packages
[TASK 9] Join node to Kubernetes Cluster
vagrant@ubuntuvm01:~/kubernetes/lxd-provisioning$ cat bootstrap-kube.sh | sudo lxc exec kworker2 bash
[TASK 1] Install docker container engine
[TASK 2] Enable and start docker service
[TASK 3] Add yum repo file for kubernetes
[TASK 4] Install Kubernetes (kubeadm, kubelet and kubectl)
[TASK 5] Enable and start kubelet service
[TASK 6] Install and configure ssh
[TASK 7] Set root password
[TASK 8] Install additional packages
[TASK 9] Join node to Kubernetes Cluster
```
# login to the master node and verify details 
```
vagrant@ubuntuvm01:~/kubernetes/lxd-provisioning$ sudo lxc exec kmaster bash 
[root@kmaster ~]# kubectl get nodes 
NAME       STATUS   ROLES    AGE     VERSION
kmaster    Ready    master   13m     v1.17.0
kworker1   Ready    <none>   7m11s   v1.17.0
kworker2   Ready    <none>   3m19s   v1.17.0
[root@kmaster ~]# kubectl version --short 
Client Version: v1.17.0
Server Version: v1.17.0
```
Profit : we just got up and running a k8s cluster on lxc containers. 


#Treafik - Provision Traefik as an Ingress Controller

   Apply role based access control to authorize Traefik to use the Kubernetes API:
   ```shell
         kubectl apply -f traefik/1-traefik-rbac.yaml
         clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created
         clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created
 ```
 Daemon Set
 ```
        [root@kmaster traefik]# kubectl apply -f 2-traefik-ds.yaml
         serviceaccount/traefik-ingress-controller created
         daemonset.apps/traefik-ingress-controller created
         service/traefik-ingress-service created
  ```
