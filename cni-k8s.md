#　kubernetes的CNI插件初始化与Pod网络设置  

## cni installation

> http://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/05/03/Kubernetes-pod-network.html  
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

## 流程分析

> http://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/05/03/Kubernetes-pod-network.html  
> https://my.oschina.net/jxcdwangtao/blog/907806  

### CNI网络插件加载前

k8s.io/kubernetes/cmd/kubelet/kubelet.go:  
```
if err := app.Run(s, nil); err != nil {
```
k8s.io/kubernetes/cmd/kubelet/app/server.go:  
```
if err := run(s, kubeDeps); err != nil {
```
k8s.io/kubernetes/cmd/kubelet/app/server.go, run():  
```
kubeDeps, err = UnsecuredKubeletDeps(s)
```
k8s.io/kubernetes/cmd/kubelet/app/server.go, UnsecuredKubeletDeps():  
```
NetworkPlugins:     ProbeNetworkPlugins(s.NetworkPluginDir, s.CNIConfDir, s.CNIBinDir),
```
k8s.io/kubernetes/cmd/kubelet/app/plugins.go:  
```
// ProbeNetworkPlugins collects all compiled-in plugins
func ProbeNetworkPlugins(pluginDir, cniConfDir, cniBinDir string) []network.NetworkPlugin {
    allPlugins := []network.NetworkPlugin{}

    // for backwards-compat, allow pluginDir as a source of CNI config files
    if cniConfDir == "" {
        cniConfDir = pluginDir
    }
    // for each existing plugin, add to the list
    allPlugins = append(allPlugins, cni.ProbeNetworkPlugins(cniConfDir, cniBinDir)...)
    allPlugins = append(allPlugins, kubenet.NewPlugin(pluginDir))

    return allPlugins
}
```
这里将所有的插件都存放在了NetworkPlugin中。

第一个插件是cni.ProbeNetworkPlugins()创建的cniNetworkPlugin，pkg/kubelet/network/cni/cni.go:  
```
func probeNetworkPluginsWithVendorCNIDirPrefix(pluginDir, binDir, vendorCNIDirPrefix string) []network.NetworkPlugin {
	if binDir == "" {
		binDir = DefaultCNIDir
	}
	plugin := &cniNetworkPlugin{
		defaultNetwork:     nil,
		loNetwork:          getLoNetwork(binDir, vendorCNIDirPrefix),
		execer:             utilexec.New(),
		pluginDir:          pluginDir,
		binDir:             binDir,
		vendorCNIDirPrefix: vendorCNIDirPrefix,
	}

	// sync NetworkConfig in best effort during probing.
	plugin.syncNetworkConfig()
	return []network.NetworkPlugin{plugin}
}
```
第二个是kubenet.NewPlugin()创建的kubenetNetworkPlugin,  
pkg/kubelet/network/kubenet/kubenet_unsupported.go：  
```
func NewPlugin(networkPluginDir string) network.NetworkPlugin {
	return &kubenetNetworkPlugin{}
}
```
这里主要分析第一个插件，也就是CNI插件加载和使用，记住它的类型是cniNetworkPlugin。  

### CNI网络插件的加载

k8s.io/kubernetes/pkg/kubelet/network/cni/cni.go:  
```
func ProbeNetworkPlugins(pluginDir, binDir string) []network.NetworkPlugin {
	return probeNetworkPluginsWithVendorCNIDirPrefix(pluginDir, binDir, "")
}
```
k8s.io/kubernetes/pkg/kubelet/network/cni/cni.go，probeNetworkPluginsWithVendorCNIDirPrefix():  
```
func probeNetworkPluginsWithVendorCNIDirPrefix(pluginDir, binDir, vendorCNIDirPrefix string) []network.NetworkPlugin {
	if binDir == "" {
		binDir = DefaultCNIDir
	}
	plugin := &cniNetworkPlugin{
		defaultNetwork:     nil,
		loNetwork:          getLoNetwork(binDir, vendorCNIDirPrefix),
		execer:             utilexec.New(),
		pluginDir:          pluginDir,
		binDir:             binDir,
		vendorCNIDirPrefix: vendorCNIDirPrefix,
	}
	// sync NetworkConfig in best effort during probing.
	plugin.syncNetworkConfig()
	return []network.NetworkPlugin{plugin}
}
```

### 加载CNI插件的配置

k8s.io/kubernetes/pkg/kubelet/network/cni/cni.go:  
```
func (plugin *cniNetworkPlugin) syncNetworkConfig() {
	network, err := getDefaultCNINetwork(plugin.pluginDir, plugin.binDir, plugin.vendorCNIDirPrefix)
	if err != nil {
		glog.Warningf("Unable to update cni config: %s", err)
		return
	}
	plugin.setDefaultNetwork(network)
}
```
加载配置文件，k8s.io/kubernetes/pkg/kubelet/network/cni/cni.go:  
```
func getDefaultCNINetwork(pluginDir, binDir, vendorCNIDirPrefix string) (*cniNetwork, error) {
	...
	// 从pluginDir目录中读取所有.conf文件
	files, err := libcni.ConfFiles(pluginDir)
	...
	for _, confFile := range files {
		conf, err := libcni.ConfFromFile(confFile)
		if err != nil {
			glog.Warningf("Error loading CNI config file %s: %v", confFile, err)
			continue
		}
		vendorDir := vendorCNIDir(vendorCNIDirPrefix, conf.Network.Type)
		cninet := &libcni.CNIConfig{
			Path: []string{binDir, vendorDir},
		}
		network := &cniNetwork{name: conf.Network.Name, NetworkConfig: conf, CNIConfig: cninet}
    // 找到一个可用的插件即返回，vendorDir指定了插件程序的路径。
		return network, nil
	}
	...
```
得到的插件的类型是cniNetwork.  

### CNI的配置文件

cniNetwork定义在 pkg/kubelet/network/cni/cni.go:  
```
type cniNetwork struct {
	name          string
	NetworkConfig *libcni.NetworkConfig
	CNIConfig     libcni.CNI
}
```
Bytes是配置文件的原始内容，Network是从配置文件中解读出的，
containernetworking/cni/libcni/api.go：  
```
type NetworkConfig struct {
	Network *types.NetConf
	Bytes   []byte
}

type NetConf struct {
	Name string `json:"name,omitempty"`
	Type string `json:"type,omitempty"`
	IPAM struct {
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
```
这是在cni项目中定义的，配置文件的内容保存在Bytes中，因此具体的CNI插件可能会再次解读配置文件。  

### CNI网络插件初始化

前面的过程结束后，kubeDeps.NetworkPlugins中就设置好了指定的插件。

```
run() ->　RunKubelet() -> CreateAndInitKubelet() -> NewMainKubelet() -> InitNetworkPlugin() -> chosenPlugin.Init()
```

pkg/kubelet/network/cni/cni.go:  
```
func (plugin *cniNetworkPlugin) Init(host network.Host, hairpinMode componentconfig.HairpinMode, nonMasqueradeCIDR string, mtu int) error {
	var err error
	plugin.nsenterPath, err = plugin.execer.LookPath("nsenter")
	if err != nil {
		return err
	}
	plugin.host = host

	plugin.syncNetworkConfig()
	return nil
}
```
这里设置了命令nsenter的路径和host，最后执行的plugin.syncNetworkConfig()其实在前面创建插件的时候已经调用过一次，应当是可以去掉的。  


### CNI网络插件的使用

`pkg/kubelet/network/cni/cni.go`, cniNetworkPlugin的定义:  

```
--cniNetworkPlugin : struct
    [fields]
   -binDir : string
   -defaultNetwork : *cniNetwork
   -execer : utilexec.Interface
   -host : network.Host
   -loNetwork : *cniNetwork
   -nsenterPath : string        //二进制文件nsenter的路径
   -pluginDir : string
   -vendorCNIDirPrefix : string
    [embedded]
   +network.NoopNetworkPlugin : network.NoopNetworkPlugin
   +sync.RWMutex : sync.RWMutex
    [methods]
   +GetPodNetworkStatus(namespace string, name string, id kubecontainer.ContainerID) : *network.PodNetworkStatus, error
   +Init(host network.Host, hairpinMode componentconfig.HairpinMode, nonMasqueradeCIDR string, mtu int) : error
   +Name() : string
   +SetUpPod(namespace string, name string, id kubecontainer.ContainerID, annotations map[string]string) : error
   +Status() : error
   +TearDownPod(namespace string, name string, id kubecontainer.ContainerID) : error
   -checkInitialized() : error
   -getDefaultNetwork() : *cniNetwork
   -setDefaultNetwork(n *cniNetwork)
   -syncNetworkConfig()
```
Init()用于初始化，setUpPod用于设置容器的网络，重点关注setUpPod。  

### setUpPod()

```
func (plugin *cniNetworkPlugin) SetUpPod(namespace string, name string, id kubecontainer.ContainerID, annotations map[string]string) error {
	if err := plugin.checkInitialized(); err != nil {
		return err
	}
	netnsPath, err := plugin.host.GetNetNS(id.ID)
	if err != nil {
		return fmt.Errorf("CNI failed to retrieve network namespace path: %v", err)
	}

	_, err = plugin.addToNetwork(plugin.loNetwork, name, namespace, id, netnsPath)
	if err != nil {
		glog.Errorf("Error while adding to cni lo network: %s", err)
		return err
	}

	_, err = plugin.addToNetwork(plugin.getDefaultNetwork(), name, namespace, id, netnsPath)
	if err != nil {
		glog.Errorf("Error while adding to cni network: %s", err)
		return err
	}

	return err
}
```

上面的addToNetwork会终将调到`containernetworking/cni/libcni/api.go`中的AddNetwork()：  
```
// AddNetwork executes the plugin with the ADD command
func (c *CNIConfig) AddNetwork(net *NetworkConfig, rt *RuntimeConf) (types.Result, error) {
	pluginPath, err := invoke.FindInPath(net.Network.Type, c.Path)
	if err != nil {
		return nil, err
	}

	net, err = injectRuntimeConfig(net, rt)
	if err != nil {
		return nil, err
	}

	return invoke.ExecPluginWithResult(pluginPath, net.Bytes, c.args("ADD", rt))
}
```
该函数调用插件，传入的是"ADD"参数！  

这里必须要说明一下，k8s中的pod至少是包含两个容器的，其中一个是pause容器，同一个pod中的其它容器和pause容器共享一个网络ns。
创建pod的时候，kubelet首先创建pause容器，得到pause容器的ID，然后创建其它容器。  

`pkg/kubelet/dockershim/docker_sandbox.go`  
```
// RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
// the sandbox is in ready state.
// For docker, PodSandbox is implemented by a container holding the network
// namespace for the pod.
// Note: docker doesn't use LogDirectory (yet).
func (ds *dockerService) RunPodSandbox(config *runtimeapi.PodSandboxConfig) (id string, err error) {
  ......
	// Step 2: Create the sandbox container.
	createConfig, err := ds.makeSandboxDockerConfig(config, image)
	if err != nil {
		return "", fmt.Errorf("failed to make sandbox docker config for pod %q: %v", config.Metadata.Name, err)
	}
	createResp, err := ds.client.CreateContainer(*createConfig)
  ......
  err = ds.network.SetUpPod(config.GetMetadata().Namespace, config.GetMetadata().Name, cID, config.Annotations)
  ......
```

### CNI Plugin在kubelet管理的PLEG中何时被调用？

> https://my.oschina.net/jxcdwangtao/blog/907806  
