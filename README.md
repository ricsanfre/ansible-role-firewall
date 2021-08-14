Ansible Role: Firewall
=========

[![CI](https://github.com/ricsanfre/ansible-role-firewall/actions/workflows/ci.yml/badge.svg)](https://github.com/ricsanfre/ansible-role-firewall/actions/workflows/ci.yml)


Install and configure firewall (nftables based) on Linux.

Requirements
------------

None.

Role Variables
--------------

Available variables are listed below along with default values (see `defaults\main.yaml`)

### Enabling NAT and traffic forwarding

* **firewall_forward_enabled** : Enable or disable traffic forwarding support on a given host. [default : `false`].
* **firewall_nat_enabled** : Enable or disable NAT support on a given host. [default : `false`].

### Define default open ports

* **in_tcp_port** : IN TCP traffic is allowed on these ports. [default : `{ ssh }`].
* **in_udp_port** : IN UDP traffic is allowed on these ports. [default : `{ snmp }`].
* **out_tcp_port** : OUT TCP traffic is allowed on these ports. [default : `{ http, https, ssh }`].
* **out_udp_port** : OUT UDP traffic is allowed on these ports. [default : `{ domain, bootps , ntp }`].


### Rules Dictionaries

For each chain within ip table (input, output, forward) and nat table (nat-prerouting and nat-postrouting), nft rules are stored in  dictionaries that can be overriding at group and host level.

**Variables and definition sets and global rules**

Set of var and set definitions used by the different rule chains and global chain rules that can be invoke from other chain using the rule `jump global` 

|define             |sets            | global chain          |
|-------------------|----------------|-----------------------|
|nft_define_default |nft_set_default |nft_global_default     |
|nft_define_group   |nft_set_group   |nft_global_group_rules |
|nft_define_host    |nft_set_host    |nft_global_host_rules  |


**IP table rules**

|input chain             |output chain             | forward chain            |
|------------------------|-------------------------|--------------------------|
|nft_input_default_rules |nft_output_default_rules |nft_forward_default_rules |
|nft_input_group_rules   |nft_output_group_rules   |nft_forward_group_rules   |
|nft_input_host_rules    |nft_output_host_rules    |nft_forward_host_rules    |

**Nat table rules**

|prerouting chain                 |postrouting chain                 |
|---------------------------------|----------------------------------|
|nft_nat_default_prerouting_rules |nft_nat_default_postrouting_rules |
|nft_nat_group_prerouting_rules   |nft_nat_group_postrouting_rules   |
|nft_nat_host_prerouting_rules    |nft_nat_host_postrouting_rules    |

Each type of rules dictionaries will be merged and rules will be applied in the alphabetical order of the keys (the reason to use 000 to 999 as prefix). So :
  * **nft_*_default_rules** : Define default rules for all nodes. You can define it in `group_vars/all`.
  * **nft_*_group_rules** : Can add rules and override those defined by **nft_*_default_rules** and **nft_*_rules**. You can define it in `group_vars/group_servers`.
  * **nft_*_host_rules** : Can add rules and override those define by **nft_*_default_rules**, **nft_*_group_rules** and **nft_*_rules**. You can define it in `host_vars/specific_host`.


## Default nftables configuration

By default the role will generate the following configurations file:

`/etc/nftables.conf`

```
#!/usr/sbin/nft -f
# Ansible managed

# clean
flush ruleset

include "/etc/nftables.d/defines.nft"

table inet filter {
	chain global {
		# 000 state management
		ct state established,related accept
		ct state invalid drop
	}
	include "/etc/nftables.d/sets.nft"
	include "/etc/nftables.d/filter-input.nft"
	include "/etc/nftables.d/filter-output.nft"
}
```

`/etc/nftables.d/defines.nft`

```

  # broadcast and multicast
  define badcast_addr = { 255.255.255.255, 224.0.0.1, 224.0.0.251 }

  # broadcast and multicast
  define ip6_badcast_addr = { ff02::16 }

  # in_tcp_accept
  define in_tcp_accept = { ssh }

  # in_udp_accept
  define in_udp_accept = { snmp }

  # out_tcp_accept
  define out_tcp_accept = { http, https, ssh }

  # out_udp_accept
  define out_udp_accept = { domain, bootps , ntp }

```

`/etc/nftables.d/sets.nft`

```
  set blackhole {
        type ipv4_addr;
        elements = $badcast_addr
    }

  set in_tcp_accept {
        type inet_service; flags interval;
        elements = $in_tcp_accept
    }

  set in_udp_accept {
        type inet_service; flags interval;
        elements = $in_udp_accept
    }

  set ip6blackhole {
        type ipv6_addr;
        elements = $ip6_badcast_addr
    }

  set out_tcp_accept {
        type inet_service; flags interval;
        elements = $out_tcp_accept
    }

  set out_udp_accept {
        type inet_service; flags interval;
        elements = $out_udp_accept
    }

```

`/etc/nftables.d/filter-input.nft`

```
chain input {
        # 000 policy
        type filter hook input priority 0; policy drop;
        # 005 global
        jump global
        # 010 drop unwanted
        ip daddr @blackhole counter drop
        # 011 drop unwanted ipv6
        ip6 daddr @ip6blackhole counter drop
        # 015 localhost
        iif lo accept
        # 050 icmp
        meta l4proto {icmp,icmpv6} accept
        # 200 input udp accepted
        udp dport @in_udp_accept ct state new accept
        # 210 input tcp accepted
        tcp dport @in_tcp_accept ct state new accept
  }

```

`/etc/nftables.d/filter-output.nft`

```
chain output {
        # 000 policy
        type filter hook output priority 0; policy drop;
        # 005 global
        jump global
        # 015 localhost
        oif lo accept
        # 050 icmp
        ip protocol icmp accept
        ip6 nexthdr icmpv6 counter accept
        # 200 output udp accepted
        udp dport @out_udp_accept ct state new accept
        # 210 output tcp accepted
        tcp dport @out_tcp_accept ct state new accept
        # 250 reset-ssh
        tcp sport ssh tcp flags { rst, psh | ack } counter accept
  }
```


And the followint ruleset on the host (showed executing `nft list ruleset` command):

```
table inet filter {
        set blackhole {
                type ipv4_addr
                elements = { 224.0.0.1, 224.0.0.251,
                             255.255.255.255 }
        }

        set in_tcp_accept {
                type inet_service
                flags interval
                elements = { 22 }
        }

        set in_udp_accept {
                type inet_service
                flags interval
                elements = { 161 }
        }

        set ip6blackhole {
                type ipv6_addr
                elements = { ff02::16 }
        }

        set out_tcp_accept {
                type inet_service
                flags interval
                elements = { 22, 80, 443 }
        }

        set out_udp_accept {
                type inet_service
                flags interval
                elements = { 53, 67, 123 }
        }

        chain global {
                ct state established,related accept
                ct state invalid drop
        }

        chain input {
                type filter hook input priority filter; policy drop;
                jump global
                ip daddr @blackhole counter packets 0 bytes 0 drop
                ip6 daddr @ip6blackhole counter packets 0 bytes 0 drop
                iif "lo" accept
                meta l4proto { icmp, ipv6-icmp } accept
                udp dport @in_udp_accept ct state new accept
                tcp dport @in_tcp_accept ct state new accept
        }

        chain output {
                type filter hook output priority filter; policy drop;
                jump global
                oif "lo" accept
                ip protocol icmp accept
                ip6 nexthdr ipv6-icmp counter packets 0 bytes 0 accept
                udp dport @out_udp_accept ct state new accept
                tcp dport @out_tcp_accept ct state new accept
                tcp sport 22 tcp flags { rst, psh | ack } counter packets 0 bytes 0 accept
        }
}

```


Dependencies
------------

None

Example Playbooks
-----------------

### Apply default rules

Install and configure a firewall on a host with default rules

```
- hosts: serverx
  roles:
    - ricsanfre.firewall
```

in `group_vars/all.yml` you could override the default rules for all your hosts:

```yml
nft_input_default_rules:
  000 policy:
    - type filter hook input priority 0; policy drop;
  005 global:
    - jump global
  010 drop unwanted:
    - ip daddr @blackhole counter drop
  011 drop unwanted ipv6:
    - ip6 daddr @ip6blackhole counter drop
  015 localhost:
    - iif lo accept
  050 icmp:
    - meta l4proto {icmp,icmpv6} accept
  200 input udp accepted:
    - udp dport @in_udp_accept ct state new accept
  210 input tcp accepted:
    - tcp dport @in_tcp_accept ct state new accept
```

### Modify default rules at group level

Openning HTTP incoming traffic for `webservers` group:

In `group_vars/webservers.yml` this can be done modifying `in_tcp_port` :

```yml
in_tcp_port: { ssh, http }
```

Or adding a new specific rule

```yml
nft_input_group_rule:
  220 input web accepted:
    - tcp dport http ct state new accept
```

### Modify defaults + group rules at host level

Openning HTTPS incoming traffic for `secureweb` host
in `host_vars/secureweb.yml` you would want to open https and remove access to http:

```yml
nft_input_group_rule:
  220 input web accepted: []
  230 input secure web accepted:
    - tcp dport https ct state new accept
```

### Default rules can be deleted

To "delete" rules, you just assign an empty list to an existing dictionary key:
Example: remove disable ICMP incoming traffic

```yml
nft_input_host_rules:
  050 icmp: []
```

Remove default rules for allowing all outgoing traffic. in a specific `group_vars/group.yml`

```yml
nft_output_group_rules:
  000 policy:
    - type filter hook output priority 0;
  005 global:
    -
  015 localhost:
    -
  050 icmp:
    -
  200 output udp accepted:
    -
  210 output tcp accepted:
    -
  250 reset-ssh:
    -
```

`000 policy` default rule is overridden to accept all traffic and the rest of rules are removed.

To summarize, rules in `nft_X_host_rules` will overwrite rules in `nft_X_group_rules`, and then rules in `nft_X_group_rules` will overwrite rules in `nft_X_default_rules`.


License
-------

MIT/BSD

Author Information
------------------

Ricardo Sanchez (ricsanfre)
