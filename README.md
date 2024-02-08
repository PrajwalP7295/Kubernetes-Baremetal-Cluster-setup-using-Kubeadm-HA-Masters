# Set up a Highly Available Baremetal Kubernetes Cluster using kubeadm
Follow this documentation to set up a highly available Kubernetes cluster using __CentOS Linux 7__.

Refer official kubernetes document for latest updates to install cluster using [kubeadm](https://v1-27.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

This documentation guides you in setting up a cluster with __3 Master__ nodes, __3 Worker__ nodes and a __load balancer__ node using HAProxy.

## VM Details
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Load Balancer|haproxy-lb.example.com|192.168.10.7|CentOS Linux 7|4G|1|
|Master|k8s-master-1.example.com|192.168.10.1|CentOS Linux 7|16G|2|
|Master|k8s-master-2.example.com|192.168.10.2|CentOS Linux 7|8G|2|
|Master|k8s-master-3.example.com|192.168.10.3|CentOS Linux 7|8G|2|
|Worker|k8s-worker-1.example.com|192.168.10.4|CentOS Linux 7|8G|2|
|Worker|k8s-worker-2.example.com|192.168.10.5|CentOS Linux 7|8G|2|
|Worker|k8s-worker-3.example.com|192.168.10.6|CentOS Linux 7|8G|2|

> * Perform all the commands as __root__ user unless otherwise specified

## Pre-requisites
For a baremetal kubernetes cluster :
- A compatible Linux host. The official k8s site provides generic instructions for Linux distributions based on __Debian__ and __Red Hat__ (CentOS/RHEL), and those distributions without a package manager.
- __2 GB__ or more of __RAM__ and __2 CPUs__ or more per machine. (except load balancer)
- __Host machine__ has atleast __2 cores__ and __16GB memory__ in this use case. Each machine has __50 GB storage__.
- __Full network connectivity__ between all machines in the cluster (public or private network is fine).
- __Unique hostname, MAC address, and product_uuid__ for every node :-
  - You can get the __MAC address__ of the network interfaces using the command : 
  ```
  ip link
  ``` 
  or 
  ```
  ifconfig -a
  ```

  - To check the __product_uuid__ : 
  ```
  cat /sys/class/dmi/id/product_uuid
  ```
  - Kubernetes uses these values to uniquely identify the nodes in the cluster. If these values are not unique to each node, the installation process may fail.
- Set __UTC__ (or preferred timezone) with same time on all nodes :-
  - Check timezone : 
  ```
  timedatectl
  ```
  - List timezones : 
  ```
  timedatectl list-timezones
  ```
  - Set the timezone (UTC or timezone of your preference) : 
  ```
  timedatectl set-timezone UTC
  ``` 
  - Restart __chronyd__ service : 
  ```
  systemctl restart chronyd
  ```
  - If the time is not synced on all nodes run the below commands :-
    - Disable NTP time sync : 
    ```
    systemctl disable --now chronyd
    ```
    - Set time manually : 
    ```
    timedatectl set-time HH:MM:SS
    ```
  - If the time on all nodes is not synced, it may lead to __pod scheduling problems, certificate validity issues, desynchronization of cluster-wide operations__, etc. 
- Certain ports need to be open on your machines :-
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
    
  - You can use tools like __netcat__ to check if a port is open. For example : 
  ```
  nc 127.0.0.1 6443
  ```

  - The above mentioned ports can be opened specifically if the machines are in a public VPC to keep the cluster secure. If the nodes/machines are operating in a private VPC then you can shutdown the firewall service instead of opening each port.
    - Check firewall __status__ : 
    ```
    firewall-cmd --state
    ```
    - __Stop__ firewall : 
    ```
    systemctl stop firewalld
    ```
    - Turn off firewall service across reboots : 
    ```
    systemctl disable firewalld
    ```
- Set __hostnames__ of each node :-
  ```
  hostnamectl set-hostname '<node_hostname>'
  ```
  For example : ```hostnamectl set-hostname '<k8s-master-1>'```

  Start new shell session / reflect changes :
  ```
  bash
  ``` 
- Configure __/etc/hosts__ file :
  > - To manage local DNS resolution and allow the nodes in the cluster to communicate with each other using hostnames.
  ```
  nano /etc/hosts
  ```
  - Make changes as per your __IPs__ and __hostnames__, and add the below lines in the __hosts__ file of each node : 
  ```
  192.168.10.1 k8s-master-1
  192.168.10.2 k8s-master-2
  192.168.10.3 k8s-master-3
  192.168.10.4 k8s-worker-1
  192.168.10.5 k8s-worker-2
  192.168.10.6 k8s-worker-3
  192.168.10.7 haproxy-lb
  ```

## Set up the Load Balancer node
- __Update packages__ : 
```
yum update
```
-  __Install Haproxy__ : 
```
yum install -y haproxy
```
- __Configure__ Haproxy :
```
nano /etc/haproxy/haproxy.cfg
```
Append the below lines to **/etc/haproxy/haproxy.cfg** - 
```
frontend kubernetes-frontend
    bind lb_node_ip:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master_node1_hostname master_node1_ip:6443 check fall 3 rise 2      
    server master_node2_hostname master_node2_ip:6443 check fall 3 rise 2
    server master_node3_hostname master_node3_ip:6443 check fall 3 rise 2
```
  > For example : 
  ```
  frontend kubernetes-frontend
    bind 192.168.10.7:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

  backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server k8s-master-1 192.168.10.1:6443 check fall 3 rise 2      
    server k8s-master-2 192.168.10.2:6443 check fall 3 rise 2
    server k8s-master-3 192.168.10.3:6443 check fall 3 rise 2
  ```
- Restart haproxy service :
```
systemctl restart haproxy
```

## Run the below commands on all the k8s nodes (3 Masters and 3 Workers)

- Disable __Swap__ memory :- 
  - You MUST disable swap in order for the kubelet to work properly.
  - __Disable__ swapping temporarily : 
  ```
  swapoff -a
  ```
  - To make this change __persistent__ across reboots : 
  ```
  sed -i '/swap/d' /etc/fstab
  ```

- Configure __SELinux__ :-
  - Temporarily set the SELinux mode to __"Permissive"__ :
    > In this mode, you can monitor policy violations without enforcing them.
  ```
  setenforce 0
  ```
  - Make this change __persistent__ across system reboots :
  ```
  sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

- Install latest stable __docker__ version :- 
  - [Uninstall Docker Engine](https://docs.docker.com/engine/install/centos/#uninstall-docker-engine)
  - [Docker installation on CentOS](https://docs.docker.com/engine/install/centos/).
  - Incase you receive an error -  __"No package found"__ during ```yum remove docker *``` command : 
  ```
  yum remove docker-ce docker-ce-cli containerd.io
  ``` 
  - If you get an __error__ during ```yum install docker ...```, then execute below commands and continue installation :
  ```
  yum remove docker-1.13.1-209.git7d71120.el7.centos.x86_64
  yum clean all
  yum update
  ```
  - __Start__ docker service :
  ```
  systemctl start docker
  ```
  - __Enable__ the docker service to start automatically at boot time using systemd as the init system :
  ```
  systemctl enable docker
  ```
  - [Post installation steps to run docker as non-root user](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)
    - Even after following above steps you receive __docker.sock__ permission error, execute below commands as __non-root__ user: 
      - Check the user is a member of grp __"docker"__ :
      ```
      groups $USER
      ```
      - __Restart__ docker :
      ```
      sudo systemctl restart docker
      ```
      - __Verify__ docker.sock perms :
      ```
      ls -l /var/run/docker.sock
      ```
      It should show the owner as __'root'__ and the group as __'docker'__. (If the group is root, then we need to execute the below command):
      - __Change__ group ownership :
      ```
      sudo chown root:docker /var/run/docker.sock
      bash
      ```
  - Verify : 
  ```
  docker ps -a
  ```

- [Install __Container Runtime__ and configure __cgroup driver__]((https://kubernetes.io/docs/setup/production-environment/container-runtimes/)) :-
  > - A container runtime is a software component responsible for executing and managing containers on a host system. __containerd, CRI-O, Docker Engine, Mirantis Container Runtime__ are the popular ones used with Kubernetes. The Container Runtime must be configured to load the CNI plugins required to implement the Kubernetes network model.
  > - Containerd is an open-source container runtime developed by the Docker. It is used to manage the lifecycle of containers on a host system and widely adopted in production environments for deploying and managing containerized applications.
  - Pre-requisites :
    - Configure __kernel modules__ : 
      ```
      cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
      overlay
      br_netfilter
      EOF
      ```
    - __Load__ the kernel modules : 
      ```
      sudo modprobe overlay
      sudo modprobe br_netfilter
      ```
    - Verify the modules are loaded : 
      ```
      lsmod | grep br_netfilter
      lsmod | grep overlay
      ```
    - Configure __sysctl parameters__ :
      > - Essential for networking and IP forwarding configurations
      ```
      cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1
      EOF
      ```
    - __Apply__ sysctl params without reboot : 
      ```
      sudo sysctl --system
      ```
    - __Verify__ the sysctl config : 
      ```
      sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
      ```
  - [Configure __containerd__](https://github.com/containerd/containerd/blob/main/docs/getting-started.md) : 
    - Step 1 : Install __containerd__

      ```Already installed during docker installation.```

    - Step 2 : Install __runc__ 

      ```The containerd.io package distributed by docker contains runc, but does not contain CNI plugins.```

    - Step 3 : Install __CNI plugins__

      - Download the __"cni-plugins-<OS>-<ARCH>-<VERSION>.tgz"__ archive from [https://github.com/containernetworking/plugins/releases](https://github.com/containernetworking/plugins/releases) - 
        ```
        wget https://link_to_cni_plugin_version.tgz
        ```
        For example, here I used "cni-plugins-linux-amd64-v1.4.0" :
        ```wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz```
      - Extract it under __/opt/cni/bin__:
        ```
        mkdir -p /opt/cni/bin
        sudo tar Cxzvf /opt/cni/bin cni_plugin_version.tgz
        ```
  - Configure __systemd cgroup driver__ :- 
    > - On Linux, __control groups__ (cgroups) are used to __constrain resources__ that are allocated to processes.
    > - Both the __kubelet__ and the underlying __container runtime__ need to interface with control groups to enforce __resource management__ for pods and containers. To __interface__ with control groups, the kubelet and the container runtime need to use a __cgroup driver__.
    > - It's critical that the kubelet and the container runtime use the __same cgroup driver__ and are __configured__ the same.
    > - There are two cgroup drivers available : ```1. cgroupfs``` ```2. systemd```
    > - __systemd__ cgroup driver is recommended for __kubeadm__ based setups instead of the __kubelet's default cgroupfs__ driver, because kubeadm manages the kubelet as a systemd service.
    - To use the systemd cgroup driver with runc, you need to make changes in the containerd __config__ file (/etc/containerd/config.toml) : 
      - Generate a default containerd config file :
        ```
        containerd config default > /etc/containerd/config.toml
        ```
      - Edit the config file : 
        ```
        nano /etc/containerd/config.toml
        ```
        - Search for the following properties in the file : 
          ```
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc] 
          ...
            [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
          ```
          - In this property, set : 
            ```
            SystemdCgroup = true
            ```
        - To prevent inconsistent sandboximage warning, search below property : 
          ```
          [plugins."io.containerd.grpc.v1.cri"]
          ```
          - Set : 
            ```
            sandbox_image = "registry.k8s.io/pause:3.9"
            ```
    - Restart containerd service :
      ```
      systemctl restart containerd
      ```
- Install __kubeadm, kubelet, kubectl___ :-
  > - __kubeadm__ : the command to bootstrap the cluster.
  > - __kubelet__ : the component that runs on all of the machines in your cluster and does things like starting pods and containers.
  > - __kubectl__: the command line util to talk to your cluster.
