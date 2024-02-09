# Set up MetalLB to the installed Kubernetes Cluster

__MetalLB__ provides a __network load-balancer__ implementation to baremetal Kubernetes clusters as these do not offer an implementation of network load balancers (Services of type LoadBalancer). It allows you to create __services__ of type __LoadBalancer__ in baremetal clusters.

It has two features that work together to provide this service: ```1. address allocation```, and ```2. external announcement```.

- __Address Allocation__ :- 

  In a cloud-based Kubernetes cluster, the cloud provider assigns an IP for your request of a LoadBalancer. In a bare-metal cluster, MetalLB is responsible for that allocation.
  
  MetalLB takes care of assigning and unassigning individual IPs to services from a __configured pool__ of IP addresses.

  This pool of IPs should be a __subset of the cluster nodes' CIDR__.

- __External Announcement__ :-

  MetalLB uses standard __networking or routing protocols__, such as __ARP, NDP, or BGP__, to inform the external network that an assigned IP address to the service "lives" within the cluster.

## Requirements 

- A Kubernetes cluster (v1.13 or later) that does not already have network load-balancing functionality.
- A [cluster network configuration](https://metallb.universe.tf/installation/network-addons/) (Flannel) that can coexist with MetalLB.
- A range of IPv4 addresses.

## [Installation](https://metallb.universe.tf/installation/)

> There are three supported ways to install MetalLB: ```1. Kubernetes manifests```, ```2. Kustomize```, or ```3. Helm```.

> In this use case, we have used the "__Kubernetes manifests__" method.

- If you’re using __kube-proxy__ in IPVS mode, since Kubernetes v1.14.2 you have to __enable strict ARP__ mode : 
  > Note : You don’t need this if you’re using __kube-router__ as service-proxy because it is enabling strict ARP by default.
  ```
  kubectl edit configmap -n kube-system kube-proxy
  ```
  and set : 
  ```
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  kind: KubeProxyConfiguration
  mode: "ipvs"
  ipvs:
    strictARP: true
  ```
- To install MetalLB, apply the manifest :
  ```
  kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
  ```
- This will deploy MetalLB to your cluster, under the ```metallb-system``` namespace. The components in the manifest are:

  - The __metallb-system/controller deployment__ : This is the cluster-wide controller that handles __IP address assignments__.
  - The __metallb-system/speaker daemonset__ : This is the component that speaks the protocol(s) of your choice to make the services reachable.
  - __Service accounts__ for the controller and speaker, along with the __RBAC__ permissions that the components need to function.

- Verify MetallB Installation :
  ```
  kubectl get pods -n metallb-system
  ```
    - Output : 
      ```
      NAME                          READY   STATUS    RESTARTS   AGE
      controller-5c46fdb5b8-b45wg   1/1     Running   0          1h
      speaker-77pqr                 1/1     Running   0          1h
      speaker-9gmwj                 1/1     Running   0          1h
      speaker-ccd4g                 1/1     Running   0          1h
      speaker-gm6np                 1/1     Running   0          1h
      speaker-r6nr2                 1/1     Running   0          1h
      speaker-x4qgr                 1/1     Running   0          1h
      ```
  ```
  kubectl api-resources | grep metallb
  ```
    - Output : 
      ```
      bfdprofiles         metallb.io/v1beta1    true    BFDProfile
      bgpadvertisements   metallb.io/v1beta1    true    BGPAdvertisement
      bgppeers            metallb.io/v1beta2    true    BGPPeer
      communities         metallb.io/v1beta1    true    Community
      ipaddresspools      metallb.io/v1beta1    true    IPAddressPool
      l2advertisements    metallb.io/v1beta1    true    L2Advertisement
      ```

#### Create IP address pool 

- Create a file named "ipaddresspool.yaml" :
  ```
  nano ipaddresspool.yaml
  ```
  - Add below lines into the file : 
    ```
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: default-pool
      namespace: metallb-system
    spec:
      addresses:
      - 192.168.10.*-192.168.10.*
    ```
    In above file, as per my use case I have used IP range : 192.168.10.*-192.168.10.* since my cluster nodes are a part of this CIDR.

    You can change this range accordingly.

- Apply the file : 
  ```
  kubectl apply -f ipaddresspool.yaml  -n metallb-system
  ```

#### Create L2Advertisement

- Create a file named "l2advertisement.yaml" :
  ```
  nano l2advertisement.yaml
  ```
  - Add below lines into the file : 
    ```
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: default-l2
      namespace: metallb-system
    spec:
      ipAddressPools:
      - default-pool
    ```
- Apply the file : 
  ```
  kubectl -n metallb-system apply -f l2advertisement.yaml
  ```

## __Congratulations__

You have installed and configured MetalLB in the baremetal kubernetes cluster.

Now test the setup using a [sample deployment](https://github.com/PrajwalP7295/baremetal-kubernetes-ha-multi-master-cluster/tree/main/sample-deployment) on the cluster.


> REFERENCE : 
- [https://github.com/morrismusumi/kubernetes/tree/main/clusters/homelab-k8s/apps/metallb-plus-nginx-ingress](https://github.com/morrismusumi/kubernetes/tree/main/clusters/homelab-k8s/apps/metallb-plus-nginx-ingress)
- [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)