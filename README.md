# Monitor easytravel on kubernetes with DynaKube

## Install k3s rancher
Rollout the easytravel application on bare metal VM (VM on a cloud provider) with k3s  
(tested with Azure VM B4ms - 4 vCPU, 8 GB)  

    #install k3s
    cd ~
    curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.21 K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--disable=traefik" sh -s -

## Test your k3s installation  
    
    kubectl get ns

## Deploy Dynakube
From Dynatrace > Deploy Dynatrace > Start Installation > Kubernetes
Generate DynaKube script installation 
    
    1) Configuration
    Name (required)
    Host Group (required)
    Token (required)
    Skip SSL certificat check 
    
    2) Download
    dynakube.yaml
    
    3) Run the script 
    kubectl create namespace dynatrace
    kubectl apply -f https://github.com/Dynatrace/dynatrace-operator/releases/download/v0.8.2/kubernetes.yaml
    kubectl -n dynatrace wait pod --for=condition=ready --selector=app.kubernetes.io/name=dynatrace-operator,app.kubernetes.io/component=webhook --timeout=300s
    kubectl apply -f dynakube.yaml

    kubectl get ns

## Deploy Easytravel
    #easytravel
    kubectl create -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest-easytravel/easytravel.yaml
    kubectl get ns

## Install Gateway Istio 
Install istio (if you use the shell in the box as putty access on tcp 443, you will lost your access and need to have a direct access with putty on tcp 22)

    echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> .profile; source ~/.profile
    curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.9.1 sh -
    sudo mv istio-1.9.1/bin/istioctl /usr/local/bin/istioctl
    istioctl install -y
    kubectl label namespace default istio-injection=enabled

    #waiting for easytravel_www pod report READY before installing istio gateway > 3 minutes
    while [[ `kubectl get pods -n easytravel | grep easytravel-www | grep "0/"` ]];do kubectl get pods -n easytravel;echo "==> waiting for easytravel_www pod ready";sleep 1; done
    kubectl apply -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest-easytravel/istio-easytravel.yaml
   
    kubectl get ns
    #at the end of this step, shell-in-the-box with port 443 will be stopped by istio - acces with putty on tcp port 22 is possible
    
## Kubernetes integration configuration
From `Kubernetes > <k8s> > Settings (in Actions menu)`. 

    Monitor event : `enable`  
    Opt in to the Kubernetes events feature for analysis and alerting : `enable`  
    Include all events relevant for Davis : `enable`  
        
## Monitor Logs

    enable the log on the cluster : Settings > Log Monitoring > Log sources and storage 

## Monitor Istio
   
   (Under construction - only heathz/ready with Opentracing)  
   enable Envoy on the monitorid technology : Settings > Monitoring > Monitored Technologies > Envoy


## Prometheus integration with Kube State Metric (KSM)   

1) Install Kube State Metric (KSM) 

kube-state-metrics is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects.    
It is not focused on the health of the individual Kubernetes components, but rather on the health of the various objects inside, such as deployments, nodes and pods.   
More details [here](https://github.com/kubernetes/kube-state-metrics)  
Install KSM in your k3s cluster with this command :   

    kubectl create -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest-kube-state-metrics/kube-state-metrics.yaml  

Now, we have some metrics with the prometheus format generated by the KSM service from the Kubernetes API.    
No need to install the prometheus solution to collect these metrics, just use Dynatrace to collect all the prometheus metrics :)  

  2) Enable metrics scraping for Dynatrace  

The process to enable *metric scraping for Dynatrace* is described [here](https://www.dynatrace.com/support/help/shortlink/monitor-prometheus-metrics#enable-metrics-scraping-required)   
Apply the updated ksm manifiest with the dynatrace metric scraping enabled 

    kubectl apply -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest-kube-state-metrics/kube-state-metrics-with-dynatrace-integration.yaml  

Look at the KSM manifest section **Deployment** with the dynatrace metric scraping enabled
![image](https://user-images.githubusercontent.com/40337213/145271037-41097192-6143-47f7-a8d7-43fcef53488b.png)  


   Return to Dynatrace  `Settings > Cloud and virtualization > Kubernetes`.   
        Monitor annotated Prometheus exporter : `enable`   
        
   On the metric explorer seacrh 'kube_'   
   ![image](https://user-images.githubusercontent.com/40337213/145270856-e741523b-fc47-430d-b257-526648052241.png)    


## Usefull command
Generate Bearer for Dynakube Integration

    kubectl get secret $(kubectl get sa dynatrace-kubernetes-monitoring -o jsonpath='{.secrets[0].name}' -n dynatrace) -o jsonpath='{.data.token}' -n dynatrace | base64 --decode;echo

Verify istio:

    istioctl analyze
    
Show all namesace

    kubectl get ns
    
Delete All pods (to restart the service)

    kubectl delete --all pods -n easytravel
    kubectl delete --all pods -n system-istio
    
Get pod installed by DynaKube

    kubectl get pods -n dynatrace

Get pod installed by easytravel

    kubectl get pods -n easytravel

Get pod installed by istio

    kubectl get pods -n istio-system

Get all pods system

    kubectl get all -n kube-system
    
Stop loadgen : 

    kubectl -n easytravel scale --replicas=0 deployment/loadgen

Restart Services 

    kubectl delete --all pods -n easytravel
    kubectl delete --all pods -n istio-system

Uninstall Istio

    istioctl x uninstall --purge

Restart k3s server:

    sudo systemctl restart k3s
    
Uninstall k3s :

    /usr/local/bin/k3s-uninstall.sh
