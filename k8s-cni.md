# Container Network Interface (CNI)

Container Network Interface - networking for Linux containers

> http://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/05/03/Kubernetes-pod-network.html  

## cni installation

> https://kubernetes.io/docs/concepts/cluster-administration/network-plugins/  

CNI插件的加载是由kubelet完成的。kubelet默认在/etc/cni/net.d目录寻找配置文件，在/opt/bin/目录中寻找二进制程序文件。  

```
kubelet \
	...
	--network-plugin=cni 
	--cni-conf-dir=/etc/cni/net.d 
	--cni-bin-dir=/opt/cni/bin 
	...
```

通过`--network-plugin`指定要使用的网络插件类型。kubelet在启动容器的时候调用CNI插件，完成容器网络的设置。  

## configuration

下面是官网给出的网络插件的配置项，

> https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration  

The network configuration is described in JSON form. The configuration may be stored on disk 
or generated from other sources by the container runtime. The following fields are well-known and have the following meaning:  

- cniVersion (string)  
Semantic Version 2.0 of CNI specification to which this configuration conforms.  

- name (string)  
Network name. This should be unique across all containers on the host (or other administrative domain).  

- type (string)  
Refers to the filename of the CNI plugin executable.  

- args (dictionary)
Optional additional arguments provided by the container runtime. For example a dictionary of labels could be passed to 
CNI plugins by adding them to a labels field under args.  

- ipMasq (boolean)  
Optional (if supported by the plugin). Set up an IP masquerade on the host for this network. 
This is necessary if the host will act as a gateway to subnets that are not able to route to the IP assigned to the container.  

- ipam  
Dictionary with IPAM specific values:
  - type (string): Refers to the filename of the IPAM plugin executable.  

- dns  
Dictionary with DNS specific values:  
  - nameservers (list of strings): list of a priority-ordered list of DNS nameservers that this network is aware of. Each entry in the list is a string containing either an IPv4 or an IPv6 address.
  - domain (string): the local domain used for short hostname lookups.
  - search (list of strings): list of priority ordered search domains for short hostname lookups. Will be preferred over domain by most resolvers.
  - options (list of strings): list of options that can be passed to the resolver

Plugins may define additional fields that they accept and may generate an error if called with unknown fields. 
The exception to this is the args field may be used to pass arbitrary data which should be ignored by plugins if not understood.  

插件可以定义自己的配置项，如果有未定义的配置项，插件将会报错！如果插件不能理解arg配置项传递的参数，arg项将被忽略，因为arg配置项是用来传递任意数据的！

## CNI支持的网络

> https://github.com/containernetworking/cni#3rd-party-plugins 

|name|desc|
|:--|:--|
|Project Calico|a layer 3 virtual network|
|Weave|a multi-host Docker network|
|Contiv Networking|policy networking for various use cases|
