# iptables command detail

iptables的详细[教程](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)

本文将主要讲一下 iptables 命令行工具的用法，先来一幅图：  

![cmd](https://github.com/fffffreedom/Pictures/blob/master/iptables/iptables-cmd1.jpg)

该图大致给出了 iptables 的组成：  
```
iptables [-t table] command chain matches target
```

**iptables applies to IPv4, ip6tables to IPv6, arptables to ARP, and ebtables to Ethernet frames.**  

## man iptables

### SYNOPSIS

```
iptables [-t table] {-A|-C|-D} chain rule-specification
ip6tables [-t table] {-A|-C|-D} chain rule-specification
iptables [-t table] -I chain [rulenum] rule-specification
iptables [-t table] -R chain rulenum rule-specification
iptables [-t table] -D chain rulenum
iptables [-t table] -S [chain [rulenum]]
iptables [-t table] {-F|-L|-Z} [chain [rulenum]] [options...]
iptables [-t table] -N chain
iptables [-t table] -X [chain]
iptables [-t table] -P chain target
iptables [-t table] -E old-chain-name new-chain-name

rule-specification = [matches...] [target]
match = -m matchname [per-match-options]
target = -j targetname [per-target-options]
```

### DESCRIPTION

Iptables  and ip6tables are used to set up, maintain, and inspect the tables of IPv4 and IPv6 packet filter rules in the Linux kernel.
everal different tables may be defined.  Each table contains a number of built-in chains and may also contain user-defined chains.

Each chain is a list of rules which can match a set of packets.  Each rule specifies what to do with a packet that matches.  
This is called a `target', which may be a jump to a user-defined chain in the same table.  

## TARGETS

A  firewall rule  specifies criteria for a packet and a target.  If the packet does not match, the next rule in the chain is examined; 
if it does match, then the next rule is specified by the value of the target, which can be the name of a user-defined chain, 
one of the targets described in iptables-extensions(8), or one of the special values ACCEPT, DROP or RETURN.  

- ACCEPT means to let the packet through.  
- DROP means to drop the packet on the floor.  
- RETURN means stop traversing this chain and resume at the next rule in the previous  (calling)  chain.  

If the end of a built-in chain is reached or a rule in a built-in chain with target RETURN is matched, 
the target specified by the chain policy determines the fate of the packet.  

## TABLES

If the end of a built-in chain is reached or a rule in a built-in chain with target RETURN is matched, 
the target specified by the chain policy determines the fate of the packet.  

```
-t, --table table
```

filter(default), nat, mangle, raw, security

## OPTIONS

The options that are recognized by iptables and ip6tables can be divided into several different groups.  

### COMMANDS
### PARAMETERS
### OTHER OPTIONS
### MATCH AND TARGET EXTENSIONS

## Iptables matches

> https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#MATCHES  

## 常用操作

- save and restore
```
iptables-save > /etc/sysconfig/iptables
iptables-restore < /etc/sysconfig/iptables
```

- 查看默认filter表中所有的链  
```
iptables -L
```

- 删除已有的rule  
```
iptables -L --line-number
iptables [-t table] -D chain rulenum
```

- 拒绝进入防火墙的所有ICMP协议数据包  
```
iptables -I INPUT -p icmp -j REJECT
```

- 允许防火墙转发除ICMP协议以外的所有数据包  
```
iptables -A FORWARD -p ! icmp -j ACCEPT
```

- 拒绝转发来自192.168.1.10主机的数据，允许转发来自192.168.0.0/24网段的数据  
```
iptables -A FORWARD -s 192.168.1.11 -j REJECT 
iptables -A FORWARD -s 192.168.0.0/24 -j ACCEPT
```
说明：注意要把拒绝的放在前面不然就不起作用了啊。  

- 丢弃从外网接口（eth1）进入防火墙本机的源地址为特定网段的数据包  
```
iptables -A INPUT -i eth1 -s 192.168.0.0/16 -j DROP 
iptables -A INPUT -i eth1 -s 172.16.0.0/12 -j DROP 
iptables -A INPUT -i eth1 -s 10.0.0.0/8 -j DROP
```

- 封堵网段（192.168.1.0/24），两小时后解封  
```
# iptables -I INPUT -s 10.20.30.0/24 -j DROP 
# iptables -I FORWARD -s 10.20.30.0/24 -j DROP 
# at now 2 hours at> iptables -D INPUT 1 at> iptables -D FORWARD 1
```
说明：这个策略咱们借助crond计划任务来完成，就再好不过了。  

- 只允许管理员从202.13.0.0/16网段使用SSH远程登录防火墙主机  
```
iptables -A INPUT -p tcp --dport 22 -s 202.13.0.0/16 -j ACCEPT 
iptables -A INPUT -p tcp --dport 22 -j DROP
```

- 允许本机开放从TCP端口20-1024提供的应用服务  
```
iptables -A INPUT -p tcp --dport 20:1024 -j ACCEPT 
iptables -A OUTPUT -p tcp --sport 20:1024 -j ACCEPT
```

- 禁止其他主机ping防火墙主机，但是允许从防火墙上ping其他主机  
```
iptables -I INPUT -p icmp --icmp-type Echo-Request -j DROP 
iptables -I INPUT -p icmp --icmp-type Echo-Reply -j ACCEPT 
iptables -I INPUT -p icmp --icmp-type destination-Unreachable -j ACCEPT
```

- 禁止转发来自MAC地址为00：0C：29：27：55：3F的和主机的数据包  
```
iptables -A FORWARD -m mac --mac-source 00:0c:29:27:55:3F -j DROP
```

- 允许防火墙本机对外开放TCP端口20、21、25、110以及被动模式FTP端口1250-1280
```
iptables -A INPUT -p tcp -m multiport --dport 20,21,25,110,1250:1280 -j ACCEPT
```
说明：这里用“-m multiport –dport”来指定目的端口及范围。  

- 禁止转发源IP地址为192.168.1.20-192.168.1.99的TCP数据包
```
iptables -A FORWARD -p tcp -m iprange --src-range 192.168.1.20-192.168.1.99 -j DROP
```
说明：此处用“-m –iprange –src-range”指定IP范围。

- 禁止转发与正常TCP连接无关的非—syn请求数据包
```
ptables -A FORWARD -m state --state NEW -p tcp ! --syn -j DROP
```
说明：“-m state”表示数据包的连接状态，“NEW”表示与任何连接无关的，新的嘛！

- 拒绝访问防火墙的新数据包，但允许响应连接或与已有连接相关的数据包
```
iptables -A INPUT -p tcp -m state --state NEW -j DROP 
iptables -A INPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
```
说明：“ESTABLISHED”表示已经响应请求或者已经建立连接的数据包，“RELATED”表示与已建立的连接有相关性的，比如FTP数据连接等。  

- 只开放本机的web服务（80）、FTP(20、21、20450-20480)，放行外部主机发住服务器其它端口的应答数据包，将其他入站数据包均予以丢弃处理
```
iptables -I INPUT -p tcp -m multiport --dport 20,21,80 -j ACCEPT 
iptables -I INPUT -p tcp --dport 20450:20480 -j ACCEPT 
iptables -I INPUT -p tcp -m state --state ESTABLISHED -j ACCEPT 
iptables -P INPUT DROP
```
