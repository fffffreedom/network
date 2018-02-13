# Flannel-Troubleshooting.md

> https://github.com/coreos/flannel/blob/master/Documentation/troubleshooting.md  

## General

### Connectivity

In Docker v1.13 and later, the default iptables forwarding policy was changed to `DROP`.  
This problems manifests itself as connectivity problems between containers running on different hosts.  
**To resolve it upgrade to the latest version of flannel.**  

### Logging

Flannel uses the glog library but only supports logging to stderr.   
The severity level（严重程度） can't be changed but the verbosity can be changed with the -v option. 
To get the most detailed logs, use -v=10  
```
-v value
    	log level for V logs
-vmodule value
    	comma-separated list of pattern=N settings for file-filtered logging
-log_backtrace_at value
    	when logging hits line file:N, emit a stack trace
```

在使用systemd的linux版本中，使用`journalctl -u flanneld`来查看flanneld的日志。  
如果flannel是运行在Pod中，则使用如下命令查日志：  
```
kubectl logs --namespace kube-system <POD_ID> -c kube-flannel
kubectl get po --namespace kube-system -l app=flannel
```

### Interface selection and the public IP

Most backends require that each node has a unique "public IP" address. This address is chosen when flannel starts. 
Because leases are tied to the public address, if the address changes, flannel must be restarted.  
租约是和`public address`绑定的，address发生了改变，必须重新flannel进程！  

网口和public IP的相关日志：  
```
I0629 14:28:35.866793    5522 main.go:386] Determining IP address of default interface
I0629 14:28:35.866987    5522 main.go:399] Using interface with name enp62s0u1u2 and address 172.24.17.174
I0629 14:28:35.867000    5522 main.go:412] Using 10.10.10.10 as external address
```

### 权限
Depending on the backend being used, flannel may need to run with super user permissions. Examples include creating VXLAN devices or programming routes. If you see errors similar to the following, confirm that the user running flannel has the right permissions (or try running with sudo).  
- Error adding route...
- Add L2 failed
- Failed to set up IP Masquerade
- Error registering network: operation not permitted

## 性能

### Control plane
Flannel is known to scale to a very large number of hosts. A delay in contacting pods in a newly created host may indicate control plane problems. Flannel doesn't need much CPU or RAM but the first thing to check would be that it has adaquate resources available. Flannel is also reliant on the performance of the datastore, either etcd or the Kubernetes API server. Check that they are performing well.  

### Data plane
Flannel relies on the underlying network so that's the first thing to check if you're seeing poor data plane performance.  
There are two flannel specific choices that can have a big impact on performance:  
- The type of backend. For example, if encapsulation is used, vxlan will always perform better than udp. For maximum data plane performance, avoid encapsulation.  
- The size of the MTU can have a large impact. To achieve maximum raw bandwidth, a network supporting a large MTU should be used. Flannel writes an MTU setting to the subnet.env file. This file is read by either the Docker daemon or the CNI flannel plugin which does the networking for individual containers.  

## Firewalls
When using udp backend, flannel uses UDP port `8285` for sending encapsulated packets.  
When using vxlan backend, kernel uses UDP port `8472` for sending encapsulated packet.  
Make sure that your firewall rules allow this traffic for all hosts participating in the overlay network.  

## Kubernetes Specific
The flannel kube subnet manager relies on the fact that each node already has a podCIDR defined.  
You can check the podCidr for your nodes with one of the following two commands:  
- kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'  
- kubectl get nodes -o template --template={{.spec.podCIDR}}  

If your nodes do not have a podCIDR, then either use the `--pod-cidr kubelet` command-line option or the `--allocate-node-cidrs=true --cluster-cidr=<cidr>` controller-manager command-line options.  

It's possible to manually set the podCIDR for each node.  
```
kubectl patch node <NODE_NAME> -p '{"spec":{"podCIDR":"<SUBNET>"}}'
```

## Log messages

- failed to read net conf
flannel expects to be able to read the net conf from "/etc/kube-flannel/net-conf.json". In the provided manifest, this is set up in the kube-flannel-cfg ConfigMap.  
- error parsing subnet config
The net conf is malformed（异常）. Double check that it has the right content and is valid JSON.  
- node <NODE_NAME> pod cidr not assigned
The node doesn't have a podCIDR defined. See above for more info.  
- Failed to create SubnetManager: error retrieving pod spec for 'kube-system/kube-flannel-ds-abc123': the server does not allow access to the requested resource
The kubernetes cluster has RBAC enabled. Run https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml  
