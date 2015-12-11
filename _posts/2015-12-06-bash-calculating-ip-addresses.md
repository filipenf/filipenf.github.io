---
layout: post
title: "Bash: Simple network ip/mask calculations with bash"
description: ""
category:
tags: [bash, networking]
---
Here is quick-and-dirty network subnet calculator for bash:

```bash
#converts an int to a netmask as 24 -> 255.255.255.0
netmask()
{
    local mask=$((0xffffffff << (32 - $1))); shift
    local ip n
    for n in 1 2 3 4; do
        ip=$((mask & 0xff))${ip:+.}$ip
        mask=$((mask >> 8))
    done
    echo $ip
}

#receives a ip/mask parameter and returns the nth ip in that range:
get_nth_ip()
{
    IFS=". /" read -r i1 i2 i3 i4 mask <<< $1
    IFS=" ." read -r m1 m2 m3 m4 <<< $(netmask $mask)
    printf "%d.%d.%d.%d\n" "$((i1 & m1))" "$((i2 & m2))" "$((i3 & m3))" "$(($2 + (i4 & m4)))"
}
```

To find the 10th ip in a subnet you just need to:

```bash
➜  ~  get_nth_ip 10.240.128.8/24 10
10.240.128.10
```

Now, the same IP with a different netmask:

```bash
➜  ~  get_nth_ip 10.240.128.8/8 10
10.0.0.10
```

When working with AWS, you could use something like:

```bash
MY_MAC=$(curl --silent http://169.254.169.254/latest/meta-data/mac)
MY_SUBNET=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/${MY_MAC}/subnet-ipv4-cidr-block)
# My firewall will always be the first available addr in the subnet
MY_FW_ADDR=$(get_nth_ip $MY_SUBNET 1)
#...
```

