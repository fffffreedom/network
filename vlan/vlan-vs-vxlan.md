# vlan和vxlan简介

> http://blog.sina.com.cn/s/blog_e01321880101i7lw.html  

## vlan

VLAN：Virtual LAN，又叫“虚拟局域网”。VLAN的作用，主要是将一个大的广播域隔离开来，形成多个小的广播域，各个广播域内可以互通，
广播域之间默认不能直接通讯。  

### 为什么需要VLAN?

如果一个公司的网络，有1000台主机，有10个部门。 如果用交换机将这些设备互联起来，不做任何隔离的行为，默认这些主机都在同一个广播域，
那么部门与部门之间的流量是可以互相传递、收到的。如下图。  
![vlan-before](https://github.com/fffffreedom/Pictures/blob/master/vlan/vlan-before.gif)

如果做任何物理或者逻辑上的隔离，加入销售部的一台电脑发出了一个广播包，这个广播包是会泛洪到整个网络的所有设备上，每台电脑都会收到这个广播包。  

这会严重的浪费系统资源，也会造成安全隐患，比如有一个用户在不停的发ARP包，每秒钟发10万个，那整个网络的所有设备每秒钟都要去处理大量的ARP包，
这会消耗大量的CPU、内存以及带宽，从而无法利用空闲的资源去完成真正需要的通讯。所以，使用VLAN技术，可以将一个大的广播域，根据需求去切割开来，
分成若干的小型的广播域，每种数据帧（单播、广播、组播）都只能在同一个VLAN里泛洪，如果要传到其他VLAN去，就必须要经过三层路由来转发。  

如下图，这就是规划了三个VLAN之后网络的通讯方式。（红色虚线之间的设备是可以通信的）  
![vlan-after](https://github.com/fffffreedom/Pictures/blob/master/vlan/vlan-after.gif)

### access、trunk和hybrid

交换机的端口加入vlan有三种方式，分别是access、trunk和hybrid。  

#### access

access端口只能加入一个vlan，一般用来连接交换机和pc，也可以连接交换机和交换机。 

#### trunk

trunk端口可以加入多个vlan，就是说可以允许多个vlan的报文通过。trunk端口有一个默认vlan，如果收到的报文没有vlan ID，
就把这个报文当做默认vlan的报文处理。**trunk口一般用于连接两台交换机**，这样可以只用一条trunk连接实现多个vlan的扩展
（因为trunk允许多个vlan的数据通过，如果用access口，那么一个vlan就要一条连接，多个vlan要多个连接，而交换机的接口是有限的）。  

对于trunk口发送出去的报文，只有默认vlan的报文不带vlan ID，其它vlan的报文都要带vlan ID（要不然，对端的交换机不知道该报文
属于哪个vlan，无法处理，也就不能实现vlan跨交换机扩展了）。简而言之，trunk端口的设计目的就是通过一条连接实现多个vlan的跨交换机扩展。  

#### hybrid

trunk端口是hybrid端口的特例，就是说hybrid端口可以实现比trunk端口更多的功能。hybrid端口可以加入多个vlan，
并可以设置该vlan的报文通过该端口发送是是否带vlan ID（trunk端口不能设置，只有默认vlan的报文不带vlan ID进行发送）。  

### vlan的限制

VID”（VLAN ID）是对VLAN的识别字段，为12位。支持4096（2^12）VLAN的识别。在4096可能的VID中，VID=0用于识别帧优先级。
4095（FFF）作为预留值，所以VLAN配置的最大可能值为4094。  

## vxlan

### 为什么需要Vxlan ?

1. vlan的数量限制  
   4096个vlan远不能满足大规模云计算数据中心的需求

2. 物理网络基础设施的限制  
   基于IP子网的区域划分限制了需要二层网络连通性的应用负载的部署

3. TOR交换机MAC表耗尽  
    虚拟化以及东西向流量导致更多的MAC表项

4. 多租户场景  
    IP地址重叠

### 什么是Vxlan ?

> http://blog.51cto.com/justim/1745351  

#### Vxlan报文

vxlan(Virtual eXtensible LAN)虚拟可扩展局域网，是一种overlay的网络技术，使用MAC in UDP的方法进行封装，共50字节的封装报文头。
具体的报文格式如下：  
![vlan-after](https://github.com/fffffreedom/Pictures/blob/master/vlan/vxlan-pkg.gif)  

- vxlan header  
增加VXLAN头（8字节），其中包含24比特的VNI字段，用来定义VXLAN网络中不同的租户。
此外，还包含VXLAN Flags（8比特，取值为00001000）和两个保留字段（分别为24比特和8比特）。  

- UDP Header  
VXLAN头和原始以太帧一起作为UDP的数据。UDP头中，目的端口号（VXLAN Port）固定为4789，
源端口号（UDP Src. Port）是原始以太帧通过哈希算法计算后的值。  

- Outer IP Header  
封装外层IP头。其中，源IP地址（Outer Src. IP）为源VM所属VTEP的IP地址，目的IP地址（Outer Dst. IP）为目的VM所属VTEP的IP地址。  

- Outer MAC Header  
封装外层以太头。其中，源MAC地址（Src. MAC Addr.）为源VM所属VTEP的MAC地址，目的MAC地址（Dst. MAC Addr.）
为到达目的VTEP的路径上下一跳设备的MAC地址。  

#### VTEP (Vxlan Tunnel EndPoint)

VXLAN 使用 VXLAN tunnel endpoint (VTEP) 设备处理 VXLAN 的封装和解封。 每个 VTEP 有一个 IP interface，配置了一个 IP 地址。  
VTEP 使用该 IP 封装 Layer 2 frame，并通过该 IP interface 传输和接收封装后的 VXLAN 数据包。  

### more

vlan原理  
https://www.cnblogs.com/frankielf0921/p/5931690.html  

VxLAN 技术介绍及华为VxLAN方案  
https://wenku.baidu.com/view/970f70c9a6c30c2259019ed8.html  

