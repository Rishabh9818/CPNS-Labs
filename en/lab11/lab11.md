# 11. Lab: Network management with the Netfilter tool

## Instructions

0. Use the network and virtual machines from the previous exercises.
1. Enable the second virtual machine to access the Internet through the first virtual machine.
2. On the first virtual machine, prevent the second virtual machine from accessing the IP address 8.8.8.8.
3. On the first virtual machine, prevent the second virtual machine from accessing the first virtual machine via secure shell (SSH).

## More information

[Netfilter](https://en.wikipedia.org/wiki/Netfilter) is a framework that is part of the Linux kernel and allows us to manage the network, for example: filter packets, map IP addresses and ports for the purpose of routing and preventing access to parts of the network.

[iptables](https://en.wikipedia.org/wiki/Iptables) represents an outdated packet filtering tool based on the Netfilter hook system.

[nftables](https://en.wikipedia.org/wiki/Nftables) a new tool that replaces `iptables` and provides additional functionality, such as packet inspection.

## Detailed instructions

### 1. Task

If we used `iptables` until now, then we delete all the rules.

    iptables -t nat -L
    iptables -t nat -F

The current `nftables` rules can be checked with the `nft` command. [Manual](https://www.netfilter.org/projects/nftables/manpage.html) for using `nft`.

    nft list ruleset

The `nftables` tool uses tables (`tables`) and chains (`chains`) to store rules hierarchically (`rules`). All tables, chains and rules have their own identifiers (`handles') which we use for deletion. For example:

    nft -a list table mytable
    nft delete rule mytable mychain handle 10

On the first virtual machine, enable the gateway by creating a new `table`, a new `chain` and a new `rule` to map packet IP addresses.

    nft add table mytable
    nft 'add chain mytable postroutingchain { type nat hook postrouting priority -100; }'
    nft add rule mytable postroutingchain masquerade

We also need to enable routing on the first virtual machine in the `/etc/sysctl.conf` file.

    nano /etc/sysctl.conf

    net.ipv4.ip_forward=1

To take into account the changes in Linux kernel parameters, use the `sysctl` command.

    sysctl -p

### 2. Task

On the first VM, we add a rule to filter all packets with the sink IP address `8.8.8.8` originating from the second virtual machine.

    nft 'add chain mytable forwardchain { type filter hook forward priority -100; }'
    nft add rule mytable forwardchain ip daddr . ip saddr { 8.8.8.8 . SECOND_VM_IP } drop

### 3. Assignment

On the first virtual machine, add a rule to filter all SSH packets originating from the second virtual machine.

    nft 'add chain mytable inputchain { type filter hook input priority -100; }'
    nft add rule mytable inputchain tcp dport . ip saddr { 22 . SECOND_VM_IP } drop


