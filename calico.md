# calico

> https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/  

## Calico Kubernetes Hosted Install

calico和kubernetes集成，一共有三种方法，具体使用哪种由实际需求决定。  

### [Standard Hosted Install](https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/hosted)

此种方法需要有一个已有的etcd集群，推荐在生产环境中使用。  

#### RBAC

If deploying Calico on an RBAC enabled cluster, you should first apply the `ClusterRole` and `ClusterRoleBinding` specs:  

```
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/rbac.yaml
```

#### Install Calico

To install Calico:  
  - Download [calico.yaml](https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/calico.yaml)  
  - Configure etcd_endpoints in the provided ConfigMap to match your etcd cluster.  
    
Then simply apply the manifest: 
```
kubectl apply -f calico.yaml
``` 

> Note: Make sure you configure the provided ConfigMap with the location of your etcd cluster before running the above command.  

#### Configuration Options

The above manifest supports a number of configuration options documented [here](https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/index#configuration-options)  

### [Kubeadm Hosted Install](https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/)

此方法主要用于搭建测试环境。  

### [Kubernetes Datastore](https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubernetes-datastore/)

此方法和kubernetes集群共用etcd集群。  

## configuration

### [calico/node](https://docs.projectcalico.org/v3.0/reference/node/configuration)

### [Felix](https://docs.projectcalico.org/v3.0/reference/felix/configuration)

## other

Integration Guide  

https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/integration  

reference

https://docs.projectcalico.org/v3.0/reference/  

