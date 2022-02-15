# Submariner VXLAN Cable Driver Overview

The following is a topology overview for a simple Submariner deployment deployed with the IPSEC cable Driver:

```
make deploy using=lighthouse
```

The idea is to show a simple interaction (a ping from one pod in one cluster to another pod in a second cluster) when Submariner is used.

![Submariner IPSEC cable driver logical view](https://github.com/maryamtahhan/notes/blob/main/images/submariner/IPSEC_GW.png)

Please note: 4 tunnels are established to allow for:
- Pod subnet to Pod subnet connectivity
- Pod subnet to Service subnet connectivity
- Service subnet to Pod subnet connectivity
- Service subnet to Service subnet connectivity

# IPSEC manual configuration equivalent
An equivalent IPSEC configuration to what submariner sets up is shown below, this configuration is
applied to a VXLAN deployment.

> **_NOTE:_** you can apply the configuration below to a vxlan deployment to see it in action.

## Cluster Information
```
[submariner-operator]# subctl show all
Cluster "cluster1"
 ✓ Showing Connections
GATEWAY          CLUSTER   REMOTE IP   NAT  CABLE DRIVER  SUBNETS                    STATUS     RTT avg.
cluster2-worker  cluster2  172.18.0.7  no   vxlan         100.2.0.0/16, 10.2.0.0/16  connected  442.987µs

 ✓ Showing Endpoints
CLUSTER ID                    ENDPOINT IP     PUBLIC IP       CABLE DRIVER        TYPE
cluster1                      172.18.0.9      66.187.232.132  vxlan               local
cluster2                      172.18.0.7      66.187.232.132  vxlan               remote

 ✓ Showing Gateways
NODE                            HA STATUS       SUMMARY
cluster1-worker                 active          All connections (1) are established

    Discovered network details via Submariner:
        Network plugin:  weave-net
        Service CIDRs:   [100.1.0.0/16]
        Cluster CIDRs:   [10.1.0.0/16]
 ✓ Showing Network details

COMPONENT                       REPOSITORY                                            VERSION
submariner                      localhost:5000                                        local
submariner-operator             localhost:5000                                        local
service-discovery               localhost:5000                                        local
 ✓ Showing versions

Cluster "cluster2"
 ✓ Showing Connections
GATEWAY          CLUSTER   REMOTE IP   NAT  CABLE DRIVER  SUBNETS                    STATUS     RTT avg.
cluster1-worker  cluster1  172.18.0.9  no   vxlan         100.1.0.0/16, 10.1.0.0/16  connected  475.65µs

 ✓ Showing Endpoints
CLUSTER ID                    ENDPOINT IP     PUBLIC IP       CABLE DRIVER        TYPE
cluster2                      172.18.0.7      66.187.232.132  vxlan               local
cluster1                      172.18.0.9      66.187.232.132  vxlan               remote

 ✓ Showing Gateways
NODE                            HA STATUS       SUMMARY
cluster2-worker                 active          All connections (1) are established

    Discovered network details via Submariner:
        Network plugin:  weave-net
        Service CIDRs:   [100.2.0.0/16]
        Cluster CIDRs:   [10.2.0.0/16]
 ✓ Showing Network details

COMPONENT                       REPOSITORY                                            VERSION
submariner                      localhost:5000                                        local
submariner-operator             localhost:5000                                        local
service-discovery               localhost:5000                                        local
 ✓ Showing versions
```

## Cluster 1 Gateway configuration

```
#[root@cluster1-worker submariner]# cat /etc/ipsec.d/submariner.conf
conn test-vpn-svc
   also=tunnel
   leftsubnets=100.1.0.0/16
   rightsubnets=100.2.0.0/16

conn test-vpn-pods
   also=tunnel
   leftsubnets=10.1.0.0/16
   rightsubnets=10.2.0.0/16

conn test-vpn-mixed1
   also=tunnel
   leftsubnets=100.1.0.0/16
   rightsubnets=10.2.0.0/16

conn test-vpn-mixed2
   also=tunnel
   leftsubnets=10.1.0.0/16
   rightsubnets=100.2.0.0/16

conn tunnel
   left=172.18.0.9
   right=172.18.0.7
   authby=secret
   pfs=yes
   rekey=yes
   keyingtries=3
   type=tunnel
   auto=start
   ike=aes_gcm-sha2;modp2048
   phase2alg=aes_gcm-null;modp2048
```
Configure the IPSEC secret to use the PSK:

```
[root@cluster2-worker submariner]# cat /etc/ipsec.d/test.secrets
%any %any: PSK "yLvPmaM+/TxWHjkV5+upGHQWbwI+aif0ihchZpxght71zJFlXiKUBD31AXsLnGrt"
```

> **_NOTE:_** to configure Transport mode replace type=tunnel in the configuration
file with type=transport.

## Cluster 2 Gateway configuration
```
[root@cluster2-worker submariner]# cat /etc/ipsec.d/submariner.conf
conn test-vpn-svc
   also=tunnel
   leftsubnets=100.2.0.0/16
   rightsubnets=100.1.0.0/16

conn test-vpn-pods
   also=tunnel
   leftsubnets=10.2.0.0/16
   rightsubnets=10.1.0.0/16

conn test-vpn-mixed1
   also=tunnel
   leftsubnets=100.2.0.0/16
   rightsubnets=10.1.0.0/16

conn test-vpn-mixed2
   also=tunnel
   leftsubnets=10.2.0.0/16
   rightsubnets=100.1.0.0/16

conn tunnel
   left=172.18.0.7
   right=172.18.0.9
   authby=secret
   pfs=yes
   rekey=yes
   keyingtries=3
   type=tunnel
   auto=start
   ike=aes_gcm-sha2;modp2048
   phase2alg=aes_gcm-null;modp2048
```

Configure the IPSEC secret to use the PSK:

```
[root@cluster2-worker submariner]# cat /etc/ipsec.d/test.secrets
%any %any: PSK "yLvPmaM+/TxWHjkV5+upGHQWbwI+aif0ihchZpxght71zJFlXiKUBD31AXsLnGrt"
```

> **_NOTE:_** to configure Transport mode replace type=tunnel in the configuration
file with type=transport.

## Run Pluto on both gateways
```
[root@cluster1-worker submariner]# pluto --stderrlog
[root@cluster2-worker submariner]# pluto --stderrlog
```

## IPSEC STATUS on the gateways
You can see the state of the configured tunnels using:
```
ipsec whack --status
```

To see the stats for the tunnels use:
```
ipsec whack --trafficstatus
```

## Verification
You can verify connectivity and traffic flow across the tunnels by:
- Bringing up two Pods and pinging from one Pod to another.
- Or running the deployment verification steps from the [Submariner documentation](https://submariner.io/getting-started/quickstart/kind/#verify-deployment)
