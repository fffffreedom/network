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

### Example configuration JSON
```
{
	"Network": "10.0.0.0/8",
	"SubnetLen": 20,
	"SubnetMin": "10.10.0.0",
	"SubnetMax": "10.99.0.0",
	"Backend": {
		"Type": "udp",
		"Port": 7890
	}
}
```

## Key command line options

```
--public-ip="": IP accessible by other nodes for inter-host communication. Defaults to the IP of the interface being used for communication.
--etcd-endpoints=http://127.0.0.1:4001: a comma-delimited list of etcd endpoints.
--etcd-prefix=/coreos.com/network: etcd prefix.
--etcd-keyfile="": SSL key file used to secure etcd communication.
--etcd-certfile="": SSL certification file used to secure etcd communication.
--etcd-cafile="": SSL Certificate Authority file used to secure etcd communication.
--kube-subnet-mgr: Contact the Kubernetes API for subnet assignment instead of etcd.
--iface="": interface to use (IP or name) for inter-host communication. Defaults to the interface for the default route on the machine. This can be specified multiple times to check each option in order. Returns the first match found.
--iface-regex="": regex expression to match the first interface to use (IP or name) for inter-host communication. If unspecified, will default to the interface for the default route on the machine. This can be specified multiple times to check each regex in order. Returns the first match found. This option is superseded by the iface option and will only be used if nothing matches any option specified in the iface options.
--iptables-resync=5: resync period for iptables rules, in seconds. Defaults to 5 seconds, if you see a large amount of contention for the iptables lock increasing this will probably help.
--subnet-file=/run/flannel/subnet.env: filename where env variables (subnet and MTU values) will be written to.
--subnet-lease-renew-margin=60: subnet lease renewal margin, in minutes.
--ip-masq=false: setup IP masquerade for traffic destined for outside the flannel network. Flannel assumes that the default policy is ACCEPT in the NAT POSTROUTING chain.
-v=0: log level for V logs. Set to 1 to see messages related to data path.
--healthz-ip="0.0.0.0": The IP address for healthz server to listen (default "0.0.0.0")
--healthz-port=0: The port for healthz server to listen(0 to disable)
--version: print version and exit
```

#### MTU是由flannel计算出来的，它的值保存在`subnet.env`里，这个值不能被修改！  
#### iptables-resync是iptable规则重新同步的间隔，如果大量的进程争夺`iptables lock`，可以增大这个值。  
#### 上面讲到的命令行选项，可以通过指定环境变量来设置！例如：  
```
--etcd-endpoints=http://10.0.0.2:2379
is equivalent to:  
FLANNELD_ETCD_ENDPOINTS=http://10.0.0.2:2379 environment variable
```
**规则如下：将选项字母改为大写，并将中划线`-`改为下划线，再在选项前面加上FLANNEL_**  

## Health Check （healthz）

Flannel provides a health check http endpoint healthz.  Currently this endpoint will blindly return http status ok(i.e. 200) when flannel is running. This feature is by default disabled. **Set healthz-port to a non-zero value will enable a healthz server for flannel**.  

## Backends

> https://github.com/coreos/flannel/blob/master/Documentation/backends.md  

`flannel`支持多个backends，一旦配置好，在运行时是不能改变的！  

### VXLAN
VXLAN is the recommended choice. Use in-kernel VXLAN to encapsulate the packets.  
据说性能太差了！  

### host-gw（需要主机在二层连通）
host-gw is recommended for more experienced users who want the performance improvement and whose infrastructure support it (typically it can't be used in cloud environments).  

Use host-gw to create IP routes to subnets via remote machine IPs. **Requires direct layer2 connectivity between hosts running flannel.** host-gw provides good performance, with few dependencies, and easy set up.  

### udp
UDP is suggested for debugging only or for very old kernels that don't support VXLAN.  

### others
AWS, GCE, and AliVPC are experimental and unsupported.  

## 运行flannel

Once you have pushed configuration JSON to etcd, you can start flanneld. 
If you published your config at the default location, you can start flanneld with no arguments.  

Flannel will acquire a subnet lease, configure its routes based on other leases in the overlay network and start routing packets.  

It will also monitor etcd for new members of the network and adjust the routes accordingly.  
路由是动态调整的！  

