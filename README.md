# Linkerd  -- Service Mesh Demo
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
> *Linkerd works by installing a set of ultralight, transparent proxies next to each service instance.  
> These proxies automatically handle all traffic to and from the service.  
> Because they’re transparent, these proxies act as highly instrumented out-of-process network stacks, sending telemetry to, and receiving control signals from, the control plane.  
> This design allows Linkerd to measure and manipulate traffic to and from your service without introducing excessive latency.
> In order to be as small, lightweight, and safe as possible, Linkerd’s proxies are written in Rust and specialized for Linkerd.*

Architecture official documentation:  
https://linkerd.io/2/reference/architecture/

At a glance:    
***Linkerd inject into the pods a ultralight proxy (sidecar) that, via iptables, redirect all the traffic of the pod through itself.
All this process is completely transparent for the application and out of the box the solution give a pretty good set of tooling for monitoring and debugging.***

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
Kubernets cluster 1.12 or later and *kubectl* configured correctly, check it out:
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


# Demo 1
## Deploy the demo app "Books"
***The scope of this demo is to demonstrate the observability and monitoring capabilities of Linkerd.***  

The Linkerd team provides a couple of application ready to deploy on the K8s cluster in order to test the service mesh.  
We are going to use the **Books** app: https://run.linkerd.io/booksapp.yml  

This app come in a vanilla state (no Linkerd "agent" installed) and with a bug so that we can see how Linkerd helps in the debug.
The deployment also provides a "client" that generate traffic towards the web app.


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

A Grafana dashboard is provided by default in the deployment of the control plane. Here below an image.
![Alt text](images/grafana_1.png?raw=true "Dashboard Failure")

## Notes
It's interesting to inspect one of the pods with the injected sidecar and see how it's configured.
``` bash
kubectl -n booksapp describe pod books
```
   ...  
   linkerd-proxy:  
     Container ID:   docker://fb430c724aa13b5c1e8ad0af2bbc034cf7c2d8c8bdb8e0c1a63e734eda1e18d1  
     Image:          gcr.io/linkerd-io/proxy:stable-2.6.0  
     Image ID:       docker-pullable://gcr.io/linkerd-io/proxy@sha256:633b8b3434f4229b219274c7d452335e3f93300a30d01d465dd80102a7df3a50  
     Ports:          4143/TCP, 4191/TCP  
     Host Ports:     0/TCP, 0/TCP  
     State:          Running  
       Started:      Fri, 18 Oct 2019 22:02:45 +0200  
     Ready:          True  
     Restart Count:  0  
     Liveness:       http-get http://:4191/metrics delay=10s timeout=1s period=10s #success=1 #failure=3  
     Readiness:      http-get http://:4191/ready delay=2s timeout=1s period=10s #success=1 #failure=3  
     Environment:  
       LINKERD2_PROXY_LOG:                           warn,linkerd2_proxy=info  
       LINKERD2_PROXY_DESTINATION_SVC_ADDR:          linkerd-dst.linkerd.svc.cluster.local:8086  
       LINKERD2_PROXY_CONTROL_LISTEN_ADDR:           0.0.0.0:4190  
       LINKERD2_PROXY_ADMIN_LISTEN_ADDR:             0.0.0.0:4191  
    ...  

# Demo 2
***The scope of this demo is to demonstrate the ability of Linkerd to completely automate the injection of the sidecar proxy for defined namespaces***  

## Namespace creation
We are going to create a new namespace with the necessary configurations so that all the pods that will be run on it will automatically be injected with the linkerd proxy.  
Inspecting the yaml file you will notice the important part regarding the annotation to enable the injection:
>  annotations: {"linkerd.io/inject":"enabled"}

Create the ns in k8s:
``` bash
kubectl apply -f https://raw.githubusercontent.com/binc75/linkerdDemo/master/autoinject-ns.yaml
```

## Pods creation
In order to verify that the pods on the namespace are actually injected with the linkerd proxy let's try to launch a simple deployment and then check if everything looks right.  
``` bash
kubectl apply -f https://raw.githubusercontent.com/binc75/linkerdDemo/master/nginx-deployment.yaml -n autoinject
```
Now inspect the pods, you should see the injected container
``` bash
kubectl get pods -n autoinject -o=jsonpath='{.items[0].spec.containers[*].name}'
```
Output should be like:
>  my-nginx ***linkerd-proxy***

## Verify the situation
With the Linkerd's CLI we can see the traffic on the namespace:
```bash
linkerd stat pods -n autoinject
```
Output:
> NAME                         STATUS   MESHED   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99   TCP_CONN
> my-nginx-75897978cd-n7shs   Running      1/1   100.00%   2.7rps           1ms           1ms           2ms          2
> my-nginx-75897978cd-rltk7   Running      1/1   100.00%   2.8rps           1ms           1ms           1ms          2
> traffic-generator           Running      1/1         -        -             -             -             -          -


We can also see what the Web UI says.
![Alt text](images/autoinject-ns.png?raw=true "Auto Inject NS")

# Conclusions
Linkerd offers out of the box many interesting features like automatic TLS, Automatic Proxy Injection, Dashboard and Grafana, Telemetry and Monitoring.  
Beside this it also enable canary & blue/green deploys via the **Traffic Split** feature, beacuse of time constraint (time boxing of 1 day) I was unable to explore this part and propose a demo.


