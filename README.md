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

