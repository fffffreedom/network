# flannel intro

## readme

`flannel`是一个简单、易用的开源工具，可用来配置专为`Kubenetes`设计的第3层网络结构`（layer 3 network fabric）`。  

每个主机上会运行一个小的二进制代理`flanneld`，它主要负责从预先准备好的大网段里分配一个子网租约。`flannel`可以使用`Kubernetes API`或者直接使用`etcd
来存储分配到的网络配置（如分配到的子网）和任何其它数据（比如主机的IP）。`flannel`使用多种`backend mechaninms`
(including VXLAN and various cloud integrations.)`中的一种来转发数据包。  

`Flannel`负责在集群中的多个节点之间提供第3层`IPv4`网络。`flannel`不控制容器是如何连接到主机网络的，而只是关注主机之间的数据包是如果传输的。
同时，`flannel`为`kubernetes`提供了`CNI`插件，也提供和`docker`集成的指南。  

`flannel`只关注于网络。对于网络策略，可以看下其它项目，如`calico`.  

## Getting started on Kubernetes (kube subnet manager)

Though not required, it's recommended that flannel uses the Kubernetes API as its backing store 
which avoids the need to deploy a discrete etcd cluster for flannel. This flannel mode is known as the `kube subnet manager`.  

就是说，如果`flannel`使用`Kubernetes API`作为后端存储，就不需要再部署`etcd`，这种方式就称为 `kube subnet manager` ；如果不是，就需要再部署。
比如和`docker`集成时，就需要再额外的部署`etcd`集群。  

## 编译 flannel

> https://github.com/coreos/flannel/blob/master/Documentation/building.md  

## 配置

> https://github.com/coreos/flannel/blob/master/Documentation/configuration.md  

flannel reads its configuration from etcd. By default, it will read the configuration from /coreos.com/network/config 
(which can be overridden using --etcd-prefix). Use the etcdctl utility to set values in etcd.  

The value of the config is a JSON dictionary with the following keys:  
- **`Network (string)`: IPv4 network in CIDR format to use for the entire flannel network. (This is the only mandatory key.)**  
- `SubnetLen (integer)`: The size of the subnet allocated to each host. Defaults to 24 (i.e. /24) unless Network was configured 
to be smaller than a /24 in which case it is one less than the network.  
- `SubnetMin (string)`: The beginning of IP range which the subnet allocation should start with. Defaults to the first subnet of Network.  
- `SubnetMax (string)`: The end of the IP range at which the subnet allocation should end with. Defaults to the last subnet of Network.  
- `Backend (dictionary)`: Type of backend to use and specific configurations for that backend. 
The list of available backends and the keys that can be put into the this dictionary are listed below. Defaults to **udp** backend.  

Subnet leases have a duration of 24 hours. Leases are renewed within 1 hour of their expiration, 
unless a different renewal margin is set with the `--subnet-lease-renew-margin` option.  

子网租约的有效期为24小时，默认情况下，租约会在截止前1小时被续约！除非配置了`--subnet-lease-renew-margin`选项。  

### 
