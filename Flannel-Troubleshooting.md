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
Depending on the backend being used, flannel may need to run with super user permissions.  

