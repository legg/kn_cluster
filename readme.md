# Knative Cluster Setup

Quick setup guide follow [this](https://www.knative.dev/docs/install/index.html) and other docs and blogs to get a knative cluster working.

See knative release pages:
* ()[https://github.com/knative/serving/releases]
* ()[https://github.com/knative/build/releases]
* ()[https://github.com/knative/eventing/releases]
* ()[https://github.com/knative/eventing-sources/releases]


## Create Kubernetes Cluster 

** Docker for mac doesn't work with Knative 0.5.0 (requries Kubernetes version 1.11) **

Dependency versions for local kube:
* minikube version: v1.0.0
* kubernetes version: v1.14.1
* virtualbox version: 6.0.4
* hyperkit version: 0.20180403

Start 

    minikube start --memory=8192 --cpus=4 \
    --kubernetes-version=v1.14.1 \
    --vm-driver=hyperkit \
    --disk-size=30g \
    --extra-config=apiserver.enable-admission-plugins="LimitRanger,NamespaceExists,NamespaceLifecycle,ResourceQuota,ServiceAccount,DefaultStorageClass,MutatingAdmissionWebhook"

Check kubernetes is running:

    kubectl cluster-info


## Knative components

* knative version: 0.5.0

Knative is setup with 5 separate components (you don't have to use them all).

* Istio - load balancer, routing, and service mesh for kube and consul (alternatively use [gloo](https://www.knative.dev/docs/install/knative-with-gloo/))
* Serving - serves your application, includes activator and autoscaler
* Build - defined by build templates and builds
* Eventing - cluster events, webhooks, build triggers
* Monitoring - prometheus, grafana, elasticsearch, kibana, fluentd, zipkin, 


### Istio

Run these

    kubectl apply --filename istio-crds.yaml 
    kubectl apply --filename istio.yaml 

Check all services are up and running (takes about 5 mins):

    kubectl get pods --namespace istio-system

Enable istio [injection](https://istio.io/help/ops/setup/injection/)  into your `default` namespace (application)

    kubectl label namespace default istio-injection=enabled

Check default namespace allows istio injection

    kubectl get namespace -L istio-injection

    ```
    NAME                 STATUS   AGE   ISTIO-INJECTION
    default              Active   12m   enabled
    istio-system         Active   12m   disabled
    ```


### Knative

Install serving, build, eventing

    kubectl apply --filename serving.yaml 
    kubectl apply --filename build.yaml 
    kubectl apply --filename release.yaml 
    kubectl apply --filename eventing-sources.yaml 
    kubectl apply --filename monitoring.yaml 
    kubectl apply --filename clusterrole.yaml

Wait 10 mins for all pods to be running or complete, keep checking with these commands

    kubectl get pods --namespace knative-serving
    kubectl get pods --namespace knative-build
    kubectl get pods --namespace knative-eventing
    kubectl get pods --namespace knative-sources
    kubectl get pods --namespace knative-monitoring

** re-run commands if errors, I experienced many issues with incorrect versions and not following the correct order of installation or not waiting for everything to be running. **


### Application

Hello world go app

    kubectl apply --filename service.yaml

To test, it get dirty, first you need to know the cluster ip address and ingress gateway port number, then you need to know the virtual host name so you can add it as a header to the request.

Virtual hostname will be: <name>.<namespace>.example.com

    export IP_ADDRESS=$(kubectl get node -o 'jsonpath={.items[0].status.addresses[0].address}')
    export PORT_NUMBER=$(kubectl get svc istio-ingressgateway --namespace istio-system -o 'jsonpath={.spec.ports[?(@.port==80)].nodePort}')
    export VIRTUAL_HOST_URL=$(kubectl get route helloworld-go  --output jsonpath='{.status.domain}')

    curl -H "Host: $VIRTUAL_HOST_URL" $IP_ADDRESS:$PORT_NUMBER

Response:

    > GET / HTTP/1.1
    > Host: helloworld-go.default.example.com
    
    < HTTP/1.1 200 OK
    < content-length: 20
    < content-type: text/plain; charset=utf-8
    < server: envoy

    Hello Go Sample v1!

