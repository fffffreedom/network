# Container Network Interface (CNI)

> https://github.com/containernetworking/cni  

Container Network Interface - networking for Linux containers

## what is CNI ?

CNI (Container Network Interface), a Cloud Native Computing Foundation project, consists of a specification and libraries for writing plugins to configure network interfaces in Linux containers, along with a number of supported plugins.  

## Network Configuration

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

## CNI实现

> https://github.com/containernetworking/cni#3rd-party-plugins  
> http://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/05/03/cni  

[cni-plugins](https://github.com/containernetworking/plugins)中提供了几个cni的实现:  
```
bridge
dhcp
flannel
host-local 
ipvlan
loopback
macvlan
portmap
ptp
sample
tuning
vlan
```

还有一些容器的网络项目自己实现了cni插件，[cni](https://github.com/containernetworking/cni)中列出的：  
```
Project Calico - a layer 3 virtual network
Weave - a multi-host Docker network
Contiv Networking - policy networking for various use cases
SR-IOV
Cilium - BPF & XDP for containers
Infoblox - enterprise IP address management for containers
Multus - a Multi plugin
Romana - Layer 3 CNI plugin supporting network policy for Kubernetes
CNI-Genie - generic CNI network plugin
Nuage CNI - Nuage Networks SDN plugin for network policy kubernetes support
Silk - a CNI plugin designed for Cloud Foundry
Linen - a CNI plugin designed for overlay networks with Open vSwitch and fit in SDN/OpenFlow network environment
Vhostuser - a Dataplane network plugin - Supports OVS-DPDK & VPP
Amazon ECS CNI Plugins - a collection of CNI Plugins to configure containers with Amazon EC2 elastic network interfaces (ENIs)
```

## CNI代码实现

### 重要结构体

`cni/libcni`的代码中定义CNI相关的结构体，可参见相应的源码：  

网络配置结构体：  
```
// NetConf describes a network.
type NetConf struct {
	CNIVersion string `json:"cniVersion,omitempty"`
	
	Name         string          `json:"name,omitempty"`
	Type         string          `json:"type,omitempty"`
	Capabilities map[string]bool `json:"capabilities,omitempty"`
	IPAM         struct {
		Type string `json:"type,omitempty"`
	} `json:"ipam,omitempty"`
	DNS DNS `json:"dns"`
}

type DNS struct {
	Nameservers []string `json:"nameservers,omitempty"`
	Domain      string   `json:"domain,omitempty"`
	Search      []string `json:"search,omitempty"`
	Options     []string `json:"options,omitempty"`
}

// NetConfList describes an ordered list of networks.
type NetConfList struct {
	CNIVersion string `json:"cniVersion,omitempty"`

	Name    string     `json:"name,omitempty"`
	Plugins []*NetConf `json:"plugins,omitempty"`
}
```

运行时配置：  
```
type RuntimeConf struct {
	ContainerID string
	NetNS       string
	IfName      string
	Args        [][2]string
	// A dictionary of capability-specific data passed by the runtime
	// to plugins as top-level keys in the 'runtimeConfig' dictionary
	// of the plugin's stdin data.  libcni will ensure that only keys
	// in this map which match the capabilities of the plugin are passed
	// to the plugin
	CapabilityArgs map[string]interface{}
}
```

### CNI接口函数

CNI接口定义了4个方法，CNIconfig实现了该接口：  

```
type CNI interface {
	AddNetworkList(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
	DelNetworkList(net *NetworkConfigList, rt *RuntimeConf) error

	AddNetwork(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
	DelNetwork(net *NetworkConfig, rt *RuntimeConf) error
}

type CNIConfig struct {
	Path []string
}
```

```
-+CNIConfig : struct
   [fields]
   +Path : []string
   [methods]
   +AddNetwork(net *NetworkConfig, rt *RuntimeConf) : types.Result, error
   +AddNetworkList(list *NetworkConfigList, rt *RuntimeConf) : types.Result, error
   +DelNetwork(net *NetworkConfig, rt *RuntimeConf) : error
   +DelNetworkList(list *NetworkConfigList, rt *RuntimeConf) : error
   +GetVersionInfo(pluginType string) : version.PluginInfo, error
   -args(action string, rt *RuntimeConf) : *invoke.Args
```

### 环境变量参数

```
CNI_COMMAND: indicates the desired operation; ADD, DEL or VERSION.
CNI_CONTAINERID: Container ID
CNI_NETNS: Path to network namespace file
CNI_IFNAME: Interface name to set up; if the plugin is unable to use this interface name it must return an error
CNI_ARGS: Extra arguments passed in by the user at invocation time. Alphanumeric key-value pairs separated by semicolons; for example, "FOO=BAR;ABC=123"
CNI_PATH: List of paths to search for CNI plugin executables. Paths are separated by an OS-specific list separator; for example ':' on Linux and ';' on Windows
```
