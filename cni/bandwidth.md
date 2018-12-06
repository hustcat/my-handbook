# CNI bandwidth plugin

CNI网络带宽隔离工具。

## 测试

* Create veth pair

```
# ip netns add netns1
# ip link add veth_host type veth peer name veth_con
# ip link set veth_con netns netns1
# ip link show veth_host
2719: veth_host@if2718: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether b2:1d:56:6a:ff:3c brd ff:ff:ff:ff:ff:ff link-netnsid 5
```

* Set `rate` to `8000bit` and `burt` to `80bit` by bandwidth

```
# cat bandwidth.conf 
{
"cniVersion": "0.3.1",
"name": "slowdown",
"type": "bandwidth",
"ingressRate": 8000,
"ingressBurst": 80,
"egressRate": 8000,
"egressBurst": 80,
"prevResult": {"cniVersion":"0.3.1", "interfaces": [{"name": "veth_host","mac": "b2:1d:56:6a:ff:3c", "sandbox": ""}]}
}

# CNI_PATH=`pwd`/bin CNI_COMMAND=ADD CNI_NETNS=/var/run/netns/netns1 CNI_CONTAINERID=netns1 CNI_IFNAME=veth_con CNI_ARGS="" bin/bandwidth < bandwidth.conf
{
    "cniVersion": "0.3.1",
    "interfaces": [
        {
            "name": "veth_host",
            "mac": "b2:1d:56:6a:ff:3c"
        },
        {
            "name": "5026",
            "mac": "7a:49:f8:e0:ff:9e"
        }
    ],
    "dns": {}
}

# ./tc -r qdisc show dev veth_host
qdisc tbf 1:[00010000] root refcnt 2 rate 8Kbit burst 80b [001312d0] lat 25.0ms limit 105b 
qdisc ingress ffff:[ffff0000] parent ffff:fff1 ---------------- 
```

* Set `rate` to `8000bit` and `burt` to `80bit(=10b)` by [tc](http://man7.org/linux/man-pages/man8/tc-tbf.8.html#SYNOPSIS)

```
# tc qdisc delete dev veth_host root handle 1:
# tc qdisc delete dev veth_host ingress


# tc qdisc add dev veth_host handle 1: root tbf rate 8kbit burst 10b latency 25ms

# ./tc  -r qdisc show dev veth_host                                                 
qdisc tbf 1:[00010000] root refcnt 2 rate 8Kbit burst 10b [0002625a] lat 25.0ms limit 35b
```

## Refs

* [CNI plugins #169](https://github.com/containernetworking/plugins/pull/169)