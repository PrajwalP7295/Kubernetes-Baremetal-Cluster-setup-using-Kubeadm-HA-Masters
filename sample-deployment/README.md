# Deploy a test application on the kubernetes cluster

#### Create a deployment file "[sample-deploy.yaml](https://github.com/PrajwalP7295/baremetal-kubernetes-ha-multi-master-cluster/blob/main/sample-deployment/sample-deploy.yaml)" :

```
nano sample-deploy.yaml
```
This manifest file deploys a single pod running an nginx web server and exposes it to external traffic through a LoadBalancer service on port 80.

#### Apply the manifest file to provision resources :

```
kubectl apply -f sample-deploy.yaml
```
  - Output : 
    ```
    deployment.apps/web-app created
    service/web-app created
    ```

#### Verify deployment :

Here we are not specifying namespace i.e. __-n default__ because the __et__ command displays resources from default namespace by default.

```
kubectl get deployment
```
  - Output : 
    ```
    NAME      READY   UP-TO-DATE   AVAILABLE   AGE
    web-app   1/1     1            1           1h
    ```

```
kubectl get pods
```
  - Output : 
    ```
    NAME                       READY   STATUS    RESTARTS   AGE
    web-app-55df88b4d9-mkr9l   1/1     Running   0          1h
    ```

```
kubectl get svc
```
  - Output : 
    ```
    NAME         TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
    kubernetes   ClusterIP      10.96.0.*     <none>           443/TCP        2d13h
    web-app      LoadBalancer   10.96.55.80   192.168.10.*     80:30266/TCP   1h
    ```

#### Output : 

<div align="center">
  <img src="./images/sample_deploy_output.png" alt="Cluster_info" width="100%" height="100%">
</div>

## __Congratulations__

You have deployed a sample application with a load balancer type service which is accessible on the web-browser.