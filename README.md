# Set up a Highly Available Baremetal Kubernetes Cluster using kubeadm
Follow this documentation to set up a highly available Kubernetes cluster using __CentOS Linux 7__.

Refer official kubernetes document for latest updates to install cluster using [kubeadm](https://v1-27.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

This documentation guides you in setting up a cluster with __3 Master__ nodes, __3 Worker__ nodes and a __load balancer__ node using HAProxy.

## VM Details
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Load Balancer|haproxy-lb.example.com|192.168.10.7|CentOS Linux 7|1G|1|
|Master|k8s-master-1.example.com|192.168.10.1|CentOS Linux 7|8G|2|
|Master|k8s-master-2.example.com|192.168.10.2|CentOS Linux 7|8G|2|
|Master|k8s-master-3.example.com|192.168.10.3|CentOS Linux 7|8G|2|
|Worker|k8s-worker-1.example.com|192.168.10.4|CentOS Linux 7|8G|2|
|Worker|k8s-worker-2.example.com|192.168.10.5|CentOS Linux 7|8G|2|
|Worker|k8s-worker-3.example.com|192.168.10.6|CentOS Linux 7|8G|2|

> * Perform all the commands as root user unless otherwise specified

## Pre-requisites
For a baremetal kubernetes cluster :
- A compatible Linux host. The official k8s site provides generic instructions for Linux distributions based on __Debian__ and __Red Hat__ (CentOS/RHEL), and those distributions without a package manager.
- __2 GB__ or more of __RAM__ and __2 CPUs__ or more per machine. (except load balancer)
- __Host machine__ has atleast __8 cores__ and __8GB memory__.
- __Full network connectivity__ between all machines in the cluster (public or private network is fine).
- __Unique hostname, MAC address, and product_uuid__ for every node :-
  - You can get the __MAC address__ of the network interfaces using the command : ```ip link``` or ```ifconfig -a```

  - To check the __product_uuid__ : ```cat /sys/class/dmi/id/product_uuid```
  - Kubernetes uses these values to uniquely identify the nodes in the cluster. If these values are not unique to each node, the installation process may fail.
- Set __UTC__ or same timezone with same time on all nodes :-
  - Check timezone : ```timedatectl```
  - List timezones : ```timedatectl list-timezones```
  - Set the timezone : ```timedatectl set-timezone UTC``` (or timezone of your preference)
  - Restart __chronyd__ service : ```systemctl restart chronyd```
  - If the time is not synced on all nodes run the below commands :-
    - Disable NTP time sync : ```systemctl disable --now chronyd```
    - Set time manually : ```timedatectl set-time HH:MM:SS```
  - If the time on all nodes is not synced, it may lead to __pod scheduling problems, certificate validity issues, desynchronization of cluster-wide operations__, etc. 
- Certain ports are open on your machines (excluding load balancer):-
  - __Control Plane(s)__ :
    |Protocol|Direction|Port Range|Purpose|Used By|
    |----|----|----|----|----|
    |TCP|Inbound|6443|Kubernetes API server|All|
    |TCP|Inbound|2379 - 2380|etcd server client API|kube-apiserver, etcd|
    |TCP|Inbound|10250|Kubelet API|Self, Control plane|
    |TCP|Inbound|10259|kube-scheduler|Self|
    |TCP|Inbound|10257|kube-controller-manager|Self|
    |TCP|Inbound|22|SSH connection|-|
    |UDP|-|8472|Cluster-wide network communication - Flannel VXLAN|-|

  - __Load Balancer__ : 
    - The same ports used by control planes.

  - __Worker Node(s)__ :
    |Protocol|Direction|Port Range|Purpose|Used By|
    |----|----|----|----|----|
    |TCP|Inbound|10250|Kubelet API|Self, Control plane|
    |TCP|Inbound|30000 - 32767|NodePort Services †|All|
    |TCP|Inbound|22|SSH connection|-|
    |UDP|-|8472|Cluster-wide network communication - Flannel VXLAN|-|

    - † - Default port range for NodePort Services.
    - All default port numbers can be overridden. 
  
  - To __open__ the respective ports given above (__Master Nodes__) : 
  ```
  firewall-cmd --zone=public --add-port=6443/tcp,2379-2380/tcp,10250/tcp,10259/tcp,10257/tcp,22/tcp,8472/tcp --permanent
  ```
  - To __open__ the respective ports given above (__Worker Nodes__) :
  ```
  firewall-cmd --zone=public --add-port=10250/tcp,30000–32767/tcp,22/tcp,8472/tcp --permanent
  ```
  - __Reload/restart__ the __firewall__ service so the changes are reflected :
  ```
  firewall-cmd --reload
  ```
    
  - You can use tools like __netcat__ to check if a port is open. For example : ```nc 127.0.0.1 6443```

  - The above mentioned ports can be opened specifically if the machines are in a public VPC to keep the cluster secure. If the nodes/machines are operating in a private VPC then you can shutdown the firewall service instead of opening each port.
    - Check firewall status : ```firewall-cmd --state```
    - Stop firewall : ```systemctl stop firewalld```
    - Turn off firewall service across reboots : ```systemctl disable firewalld```
