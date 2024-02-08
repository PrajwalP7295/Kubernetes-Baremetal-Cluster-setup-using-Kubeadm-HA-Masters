# Set up a Highly Available Baremetal Kubernetes Cluster using kubeadm
Follow this documentation to set up a highly available Kubernetes cluster using __CentOS Linux 7__.

This documentation guides you in setting up a cluster with 3 Master nodes, 3 Worker nodes and a load balancer node using HAProxy.

## VM Details
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Load Balancer|haproxy-lb.example.com|192.168.10.148|Ubuntu 20.04|1G|1|
|Master|k8s-master-1.example.com|192.168.10.31|Ubuntu 20.04|8G|2|
|Master|k8s-master-2.example.com|192.168.10.13|Ubuntu 20.04|8G|2|
|Master|k8s-master-3.example.com|192.168.10.147|Ubuntu 20.04|8G|2|
|Worker|k8s-worker-1.example.com|192.168.10.151|Ubuntu 20.04|8G|2|
|Worker|k8s-worker-2.example.com|192.168.10.152|Ubuntu 20.04|8G|2|
|Worker|k8s-worker-3.example.com|192.168.5.207|Ubuntu 20.04|8G|2|

> * Perform all the commands as root user unless otherwise specified



