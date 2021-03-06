# iptables wiki

iptables是Linux防火墙的用户态配置工具，防火墙在内核态中依赖netfilter；firewalld，类似于iptables，也是防火墙的用户态配置工具。  

> https://en.wikipedia.org/wiki/Iptables

iptables is a user-space utility program that allows a system administrator to configure the tables[2] 
provided by the Linux kernel firewall (implemented as different Netfilter modules) and the chains and 
rules it stores. Different kernel modules and programs are currently used for different protocols; 
iptables applies to IPv4, ip6tables to IPv6, arptables to ARP, and ebtables to Ethernet frames.  

The term iptables is also commonly used to inclusively refer to the kernel-level components. 
x_tables is the name of the kernel module carrying the shared code portion used by all four modules 
that also provides the API used for extensions; subsequently, Xtables is more or less used to 
refer to the entire firewall (v4, v6, arp, and eb) architecture.  

Xtables allows the system administrator to define tables containing chains of rules for the 
treatment of packets. Each table is associated with a different kind of packet processing. 
Packets are processed by sequentially traversing the rules in chains.  

```
tables ----- chains
         |-- chains
         |-- ...
         |-- chains ----- rules
                      |-- rules
                      |-- ...
```

A rule in a chain can cause a goto or jump to another chain, and this can be repeated to whatever level of nesting is desired. 
Every network packet arriving at or leaving from the computer traverses at least one chain.  

The origin of the packet determines which chain it traverses initially. There are five predefined chains 
(mapping to the five available Netfilter hooks), though a table may not have all chains.  

Predefined chains have a policy, for example DROP, which is applied to the packet if it reaches the end of the chain.  
(数据包未匹配到chain中任何一条rule，则使用默认的rule，如果存在)  

The system administrator can create as many other chains as desired. These chains have no policy; 
if a packet reaches the end of the chain it is returned to the chain which called it. **A chain may be empty.**  

- PREROUTING: Packets will enter this chain before a routing decision is made.  
- INPUT: Packet is going to be locally delivered. It does not have anything to do with processes having an opened socket; 
local delivery is controlled by the "local-delivery" routing table: ip route show table local.  
- FORWARD: All packets that have been routed and were not for local delivery will traverse this chain.  
- OUTPUT: Packets sent from the machine itself will be visiting this chain.  
- POSTROUTING: Routing decision has been made. Packets enter this chain just before handing them off to the hardware.  

Each rule in a chain contains the specification of which packets it matches. It may also contain a target (used for extensions) or verdict (one of the built-in decisions).  As a packet traverses a chain, each rule in turn is examined. If a rule does not match the packet, the packet is passed to the next rule. If a rule does match the packet, the rule takes the action indicated by the target/verdict, which may result in the packet being allowed to continue along the chain or it may not. Matches make up the large part of rulesets, as they contain the conditions packets are tested for. These can happen for about any layer in the OSI model, as with e.g. the --mac-source and -p tcp --dport parameters, and there are also protocol-independent matches, such as -m time.

The packet continues to traverse the chain until either:  
1. a rule matches the packet and decides the ultimate fate of the packet, for example by calling one of the ACCEPT or DROP, or a module returning such an ultimate fate; or  
2. a rule calls the RETURN verdict, in which case processing returns to the calling chain; or  
3. the end of the chain is reached; traversal either continues in the parent chain (as if RETURN was used), or the base chain policy, which is an ultimate fate, is used.  

Targets also return a verdict like ACCEPT (NAT modules will do this) or DROP (e.g. the REJECT module), but may also imply CONTINUE (e.g. the LOG module; CONTINUE is an internal name) to continue with the next rule as if no target/verdict was specified at all.  
