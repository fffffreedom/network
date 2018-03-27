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


