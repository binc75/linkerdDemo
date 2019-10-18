# Linkerd 2 -- Service Mesh Demo
## Introduction
I've been asked to come up with a demo of a K8s cluster with a service mesh installed.  
I decided to proceed with the use of Linkerd (https://linkerd.io/) because of different reasons:
 - semplicity of the installation
 - clean architectural design
 - really lightweight for the infrastructure
 - good documentation 

Other tools were available, one over all Istio (https://istio.io/), but because no specific requirements were asked I opted for what I though it would be the best solution for a demo/poc.

## Description
From the official documentation:  
*Linkerd works by installing a set of ultralight, transparent proxies next to each service instance.  
These proxies automatically handle all traffic to and from the service.  
Because they’re transparent, these proxies act as highly instrumented out-of-process network stacks, sending telemetry to, and receiving control signals from, the control plane.  
This design allows Linkerd to measure and manipulate traffic to and from your service without introducing excessive latency.
In order to be as small, lightweight, and safe as possible, Linkerd’s proxies are written in Rust and specialized for Linkerd.*

Architecture official documentation:  
https://linkerd.io/2/reference/architecture/

**At a glance:**    
***Linkerd inject into the pods a ultralight proxy (sidecar) that, via iptables, redirect all the traffic of the pod through itself.***

# Installation
## Kubernetes environment
For the purpose of this POC an installation of minikube is sufficient. 

``` bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start
```

## Linkerd  
The installation process it's pretty smooth and well documented. 

### Requirements
Kubernets cluster 1.12 or later and *kubectl* configured correctly.
Check it:
``` bash
kubectl version --short
```

### Client installation
``` bash
curl -sL https://run.linkerd.io/install | sh
echo 'export PATH=$PATH:$HOME/.linkerd2/bin' >> ~/.bashrc
source ~/.bashrc
```
Check
``` bash
linkerd version
```
The output of the command will report *Server version: unavailable*, that's ok, don't worry, this will be installed later.

### Kubernetes cluster validation
To check that the kubernets cluster is configured correctly and it's ready to install the control plane run:
``` bash
linkerd check --pre
```

### Control Plane installation
``` bash
linkerd install | kubectl apply -f -
```
It may take few minutes to pull and run all the Linkerd images, wait until the output of the following command it's ok:
``` bash
linkerd check
```


At this point the installation it's complete and you should e able to navigate the dashboard.  
This command will launch a browser:
``` bash
linkerd dashboard &
```
![Alt text](images/linkerd_dashboard_1.png?raw=true "Dashboard")


# Demo
## Deploy the demo app "Books"
The Linkerd team provides a couple of application ready to deploy on the K8s cluster in order to test the service mesh.  
We are going to use the **Books** app: https://run.linkerd.io/booksapp.yml  

This app come in a vanilla state (no Linkerd "agent" installed) and with a bug so that we can see how Linkerd helps in the debug.  
Let's proceed with the deployment:
``` bash
kubectl create ns booksapp
curl -sL https://run.linkerd.io/booksapp.yml | kubectl -n booksapp apply -f -
```
Wait for the deployment to complete, check with:
``` bash
kubectl get all -n booksapp
kubectl -n booksapp rollout status deploy webapp
```
## Apply the service mesh to the app
The following command will get the yaml file of the running deployment, pass it to *linkerd inject* that will inject the linkerd-proxy into the pods and then pass back the new yaml to kubectl.
``` bash
kubectl get -n booksapp deploy -o yaml | linkerd inject - | kubectl apply -f -
```
At this point kubernetes will restart the needed pods in order to get them running with the modified configuration (sidecar container).
## Verify the situation
In the dashboard is now possible to drill down to the specific microservice that is giving problems, in this case *deploy/books*

![Alt text](images/linkerd_dashboard_2.png?raw=true "Dashboard Failure")


![Alt text](images/linkerd_dashboard_3.png?raw=true "Dashboard Failure")


![Alt text](images/linkerd_dashboard_4.png?raw=true "Dashboard Failure")


We can also use CLI to inspect the status (https://linkerd.io/2/reference/cli/stat/):
``` bash
linkerd stat deployments -n booksapp
```
Here something similar to tcpdump (https://linkerd.io/2/reference/cli/tap/index.html):
``` bash
linkerd tap ns/booksapp
```
