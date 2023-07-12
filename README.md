# Tech Talk - k0smotron - Part 1 - Introduction

This repository contains all necessary code snippets and a guide replicate the demo shown in the Tech Talk [Revolutionizing Kubernetes: Harness the Power of k0smotron for Scalable Control Planes](https://www.mirantis.com/labs/revolutionizing-kubernetes-harness-the-power-of-k0smotron-for-scalable-control-planes). 

### Versions
The following version will be used in this repository: 

- k0s (v1.27.3)
- k0smotron (v0.4.2)
- AWS Cloud Controller Manager (v0.0.7)
- AWS EBS CSI Driver (v2.17.2)
- AWS EBS StorageClass 

### Preperation

Like in the Tech Talk Demo we will use AWS to create the k0s Management Cluster. 
For preparation, get and apply your AWS credentials in the terminal, so we can use Terraform to create necessary resources.
> **NOTE**: The k0s Management Cluster can run in any infrastructure / cloud (private/public). In this demo we will use AWS. 

## Management Cluster Installation
Please check the ```terraform.tfvars``` and adjust the AMI in ```main.tf``` as needed. 

``` shell
terraform init

terraform apply -auto-approve

terraform output -raw k0s_cluster | k0sctl apply --no-wait --debug --config -

terraform output -raw k0s_cluster | k0sctl kubeconfig --config -
```

et voilà, a k0s cluster with 1 controller, 1 worker, integration into AWS and k0smotron.

> **NOTE**: If you want to create a high available cluster, please set the controller_count to 3. This will automatically create a ELB. 



## Create k0s Control Planes

Next, you can use k0smotron, running in the Management Cluster, to create k0s control planes:
``` yaml=
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: k0smotron.io/v1beta1
kind: Cluster
metadata:
  name: k0s-demo-cluster
  namespace: demo
spec:
  replicas: 1
  k0sImage: k0sproject/k0s
  k0sVersion: v1.27.3-k0s.0
  service:
    type: LoadBalancer
    apiPort: 6443
    konnectivityPort: 8132
  persistence:
    type: emptyDir
```
To get kubeconfig of the newly created Cluster / Control Plane, we execute the following command:
``` shell
kubectl get secret k0s-demo-cluster-kubeconfig -n demo -o jsonpath='{.data.value}' | base64 -d > ~/.kube/child.conf
```
This will allow us to connect to the new cluster. Please keep in mind that it can take some minutes for the AWS Cloud Controller Manager to create the LoadBalancer. 

Now, we want to add Worker Nodes. To do that we will create a `JoinTokenRequest` on the Management Cluster:
``` yaml=
apiVersion: k0smotron.io/v1beta1
kind: JoinTokenRequest
metadata:
  name: edge-token
  namespace: demo
spec:
  clusterRef:
    name: k0s-demo-cluster
    namespace: demo
```
Let's get the token to add worker nodes.:
```shell
kubectl get secret edge-token -n demo -o jsonpath='{.data.token}' | base64 -d
```

With the join token, we can create a file, including the join token, on the desired nodes, install k0s and add it to our cluster:
``` shell
echo '[Join-Token]' > token

curl -sSLf https://get.k0s.sh | sudo sh

sudo k0s install worker --token-file /path/to/token

sudo k0s start
```
You will see the new worker node when running the following command: 
``` shell
kubectl get no --kubeconfig ~/.kube/child.conf
```

## Test Workload
Now that we have a k0s cluster with the Control Plane running in the Management Cluster and a worker node running outside, we can run a test workload and simply expose the pod via NodePort:
```
kubectl create deployment hello-world --image=jlnhnng/hello-world --kubeconfig ~/.kube/child.conf

kubectl expose deployment hello-world --port=80 --type=NodePort --kubeconfig ~/.kube/child.conf
```

Happy testing! 

## Notes

### Configuration
This is a simple demo of k0s on AWS, k0smotron and k0s control planes in the management cluster. If you need more advanced configurations please check our documentation of k0s and k0smotron: 
- k0s Docs -> https://docs.k0sproject.io/
- k0smotron Docs -> https://docs.k0smotron.io/

### ClusterAPI
Since this is only half the solution and we don't want to manually create VMs for k0s workers, there is a close integration between k0smotron and ClusterAPI. Please have a look for the second part of the k0smotron series at https://www.mirantis.com/labs/ and the second repository that will be available shortly after the Tech Talk.
Otherwise you can find a detailed guide on how this works [here](https://docs.k0smotron.io/v0.4.2/cluster-api/).