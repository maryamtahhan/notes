# Manually deploying Submariner with IP in IP

## IP in IP point to point

The goal of this guide is to show how to replace the main vxlan-tunnel interface on the Submariner gateway with an 
IP in IP interface.

Deploy 2 VXLAN submariner clusters from submariner-operator using

```bash
# make deploy using=vxlan,lighthouse
```

This will produce 2 clusters:

```bash
# subctl show all
Cluster "cluster1"
 ✓ Showing Connections
GATEWAY          CLUSTER   REMOTE IP   NAT  CABLE DRIVER  SUBNETS                    STATUS     RTT avg.
cluster2-worker  cluster2  172.18.0.6  no   vxlan         100.2.0.0/16, 10.2.0.0/16  connected  287.129µs

 ✓ Showing Endpoints
CLUSTER ID                    ENDPOINT IP     PUBLIC IP       CABLE DRIVER        TYPE
cluster1                      172.18.0.8      66.187.232.132  vxlan               local
cluster2                      172.18.0.6      66.187.232.132  vxlan               remote

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
cluster1-worker  cluster1  172.18.0.8  no   vxlan         100.1.0.0/16, 10.1.0.0/16  connected  322.706µs

 ✓ Showing Endpoints
CLUSTER ID                    ENDPOINT IP     PUBLIC IP       CABLE DRIVER        TYPE
cluster2                      172.18.0.6      66.187.232.132  vxlan               local
cluster1                      172.18.0.8      66.187.232.132  vxlan               remote

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
...
```

You can see the submariner pods in each cluster context using

```bash
# export KUBECONFIG=$(find $(git rev-parse --show-toplevel)/output/kubeconfigs/ -type f -printf %p:)
# kubectl get pods -n submariner-operator
NAME                                             READY   STATUS    RESTARTS   AGE
submariner-gateway-tkw4m                         1/1     Running   0          107m
submariner-lighthouse-agent-759754c879-fscvk     1/1     Running   0          107m
submariner-lighthouse-coredns-7869cfcc87-gsnrl   1/1     Running   0          107m
submariner-lighthouse-coredns-7869cfcc87-vsk7b   1/1     Running   0          107m
submariner-operator-5588b78fb5-7nszn             1/1     Running   1          108m
submariner-routeagent-6frfq                      1/1     Running   0          107m
submariner-routeagent-kcxl9                      1/1     Running   0          107m
submariner-routeagent-rlqpk                      1/1     Running   0          107m
```

you can connect to the gateway pod/container using:

```bash
# kubectl exec -n submariner-operator --stdin --tty submariner-gateway-tkw4m  -- /bin/bash
```

## To setup an IP in IP interface on each cluster gateway do the following

Once in the gateway pod, bring down the vxlan-tunnel interface, take note of it's IP address
as this is what you are going to assign to the new interface ipip0. Finally delete vxlan-tunnel
IP address.

Please ensure to replace the IP addresses based on what is deployed for your cluster

### Cluster 1 configuration

On Cluster1-worker:

```bash
# ip link add name ipip0 type ipip local 172.18.0.8 remote 172.18.0.6
# ip link set ipip0 up
# ip addr add 240.18.0.8/8 dev ipip0
# ip route add 100.2.0.0/16 via 241.18.0.6 dev ipip0 table 100
# ip route add 10.2.0.0/16 via 241.18.0.6 dev ipip0 table 100
```

### Cluster 2 configuration

On Cluster2-worker:

```bash
# ip link add name ipip0 type ipip local 172.18.0.6 remote 172.18.0.8
# ip link set ipip0 up
# ip addr add 240.18.0.6/8 dev ipip0
# ip route add 100.1.0.0/16 via 241.18.0.8 dev ipip0 table 100
# ip route add 10.1.0.0/16 via 241.18.0.8 dev ipip0 table 100
```

## IP in IP single interface multiple tunnels

### Multiple Clusters

Start with deploying 4 vxlan Submariner clusters, by updating .shipyard.e2e.yml

```bash
$ git diff .shipyard.e2e.yml
diff --git a/.shipyard.e2e.yml b/.shipyard.e2e.yml
index f2f5acd..6a87646 100644
--- a/.shipyard.e2e.yml
+++ b/.shipyard.e2e.yml
@@ -1,7 +1,9 @@
 ---
 cni: weave
 submariner: true
-nodes: control-plane worker worker
+nodes: control-plane worker worker worker
 clusters:
   cluster1:
   cluster2:
+  cluster3:
+  cluster4:
```

Deploy 4 VXLAN submariner clusters from submariner-operator using

```bash
# make deploy using=vxlan,lighthouse
```

Check the deployment:

```bash
# subctl show all
Cluster "cluster1"
 ✓ Showing Connections
GATEWAY          CLUSTER   REMOTE IP    NAT  CABLE DRIVER  SUBNETS                    STATUS  RTT avg.
cluster4-worker  cluster4  172.18.0.14  no   vxlan         100.4.0.0/16, 10.4.0.0/16  error   360.108µs
cluster3-worker  cluster3  172.18.0.21  no   vxlan         100.3.0.0/16, 10.3.0.0/16  error   347.615µs
cluster2-worker  cluster2  172.18.0.7   no   vxlan         100.2.0.0/16, 10.2.0.0/16  error   387.623µs

 ✓ Showing Endpoints
CLUSTER ID                    ENDPOINT IP     PUBLIC IP       CABLE DRIVER        TYPE
cluster1                      172.18.0.17     66.187.232.132  vxlan               local
cluster4                      172.18.0.14     66.187.232.132  vxlan               remote
cluster3                      172.18.0.21     66.187.232.132  vxlan               remote
cluster2                      172.18.0.7      66.187.232.132  vxlan               remote

 ✓ Showing Gateways
NODE                            HA STATUS       SUMMARY
cluster1-worker                 active          0 connections out of 3 are established

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

Cluster "cluster3"
 ✓ Showing Connections
GATEWAY          CLUSTER   REMOTE IP    NAT  CABLE DRIVER  SUBNETS                    STATUS  RTT avg.
cluster1-worker  cluster1  172.18.0.17  no   vxlan         100.1.0.0/16, 10.1.0.0/16  error   363.136µs
cluster4-worker  cluster4  172.18.0.14  no   vxlan         100.4.0.0/16, 10.4.0.0/16  error   377.228µs
cluster2-worker  cluster2  172.18.0.7   yes  vxlan         100.2.0.0/16, 10.2.0.0/16  error   320.038µs

 ✓ Showing Endpoints
CLUSTER ID                    ENDPOINT IP     PUBLIC IP       CABLE DRIVER        TYPE
cluster3                      172.18.0.21     66.187.232.132  vxlan               local
cluster1                      172.18.0.17     66.187.232.132  vxlan               remote
cluster4                      172.18.0.14     66.187.232.132  vxlan               remote
cluster2                      172.18.0.7      66.187.232.132  vxlan               remote

 ✓ Showing Gateways
NODE                            HA STATUS       SUMMARY
cluster3-worker                 active          0 connections out of 3 are established

    Discovered network details via Submariner:
        Network plugin:  weave-net
        Service CIDRs:   [100.3.0.0/16]
        Cluster CIDRs:   [10.3.0.0/16]
 ✓ Showing Network details

COMPONENT                       REPOSITORY                                            VERSION
submariner                      localhost:5000                                        local
submariner-operator             localhost:5000                                        local
service-discovery               localhost:5000                                        local
 ✓ Showing versions

Cluster "cluster4"
 ✓ Showing Connections
GATEWAY          CLUSTER   REMOTE IP    NAT  CABLE DRIVER  SUBNETS                    STATUS  RTT avg.
cluster1-worker  cluster1  172.18.0.17  no   vxlan         100.1.0.0/16, 10.1.0.0/16  error   348.276µs
cluster3-worker  cluster3  172.18.0.21  no   vxlan         100.3.0.0/16, 10.3.0.0/16  error   377.784µs
cluster2-worker  cluster2  172.18.0.7   no   vxlan         100.2.0.0/16, 10.2.0.0/16  error   359.657µs

 ✓ Showing Endpoints
CLUSTER ID                    ENDPOINT IP     PUBLIC IP       CABLE DRIVER        TYPE
cluster4                      172.18.0.14     66.187.232.132  vxlan               local
cluster1                      172.18.0.17     66.187.232.132  vxlan               remote
cluster3                      172.18.0.21     66.187.232.132  vxlan               remote
cluster2                      172.18.0.7      66.187.232.132  vxlan               remote

 ✓ Showing Gateways
NODE                            HA STATUS       SUMMARY
cluster4-worker                 active          0 connections out of 3 are established

    Discovered network details via Submariner:
        Network plugin:  weave-net
        Service CIDRs:   [100.4.0.0/16]
        Cluster CIDRs:   [10.4.0.0/16]
 ✓ Showing Network details

COMPONENT                       REPOSITORY                                            VERSION
submariner                      localhost:5000                                        local
submariner-operator             localhost:5000                                        local
service-discovery               localhost:5000                                        local
 ✓ Showing versions

Cluster "cluster2"
 ✓ Showing Connections
GATEWAY          CLUSTER   REMOTE IP    NAT  CABLE DRIVER  SUBNETS                    STATUS  RTT avg.
cluster1-worker  cluster1  172.18.0.17  no   vxlan         100.1.0.0/16, 10.1.0.0/16  error   375.717µs
cluster4-worker  cluster4  172.18.0.14  no   vxlan         100.4.0.0/16, 10.4.0.0/16  error   363.328µs
cluster3-worker  cluster3  172.18.0.21  no   vxlan         100.3.0.0/16, 10.3.0.0/16  error   366.819µs

 ✓ Showing Endpoints
CLUSTER ID                    ENDPOINT IP     PUBLIC IP       CABLE DRIVER        TYPE
cluster2                      172.18.0.7      66.187.232.132  vxlan               local
cluster1                      172.18.0.17     66.187.232.132  vxlan               remote
cluster4                      172.18.0.14     66.187.232.132  vxlan               remote
cluster3                      172.18.0.21     66.187.232.132  vxlan               remote

 ✓ Showing Gateways
NODE                            HA STATUS       SUMMARY
cluster2-worker                 active          0 connections out of 3 are established

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

### Cluster 1 config

```bash
[root@cluster1-worker submariner]# cat setup.sh
#!/bin/bash
ip link add name ipip0 type ipip local 172.18.0.17 remote any
ip link set ipip0 up
ip addr add 241.18.0.17/8 dev ipip0
ip route add 10.2.0.0/16 encap ip id 100 dst 172.18.0.7 via 241.18.0.7 dev ipip0 table 100
ip route add 100.2.0.0/16 encap ip id 100 dst 172.18.0.7 via 241.18.0.7 dev ipip0 table 100
ip route add 10.3.0.0/16 encap ip id 100 dst 172.18.0.21 via 241.18.0.21 dev ipip0 table 100
ip route add 100.3.0.0/16 encap ip id 100 dst 172.18.0.21 via 241.18.0.21 dev ipip0 table 100
ip route add 10.4.0.0/16 encap ip id 100 dst 172.18.0.14 via 241.18.0.14 dev ipip0 table 100
ip route add 100.4.0.0/16 encap ip id 100 dst 172.18.0.14 via 241.18.0.14 dev ipip0 table 100
```

### Cluster 2 config

```bash
[root@cluster2-worker submariner]# cat setup.sh
#!/bin/bash
ip link add name ipip0 type ipip local 172.18.0.7 remote any
ip link set ipip0 up
ip addr add 241.18.0.7/8 dev ipip0
ip route add 10.1.0.0/16 encap ip id 100 dst 172.18.0.17 via 241.18.0.7 dev ipip0 table 100
ip route add 100.1.0.0/16 encap ip id 100 dst 172.18.0.17 via 241.18.0.7 dev ipip0 table 100
ip route add 10.3.0.0/16 encap ip id 100 dst 172.18.0.21 via 241.18.0.21 dev ipip0 table 100
ip route add 100.3.0.0/16 encap ip id 100 dst 172.18.0.21 via 241.18.0.21 dev ipip0 table 100
ip route add 10.4.0.0/16 encap ip id 100 dst 172.18.0.14 via 241.18.0.14 dev ipip0 table 100
ip route add 100.4.0.0/16 encap ip id 100 dst 172.18.0.14 via 241.18.0.14 dev ipip0 table 100
```

### Cluster 3 config

```bash
[root@cluster3-worker submariner]# cat setup.sh
#!/bin/bash
ip link add name ipip0 type ipip local 172.18.0.21 remote any
ip link set ipip0 up
ip addr add 241.18.0.21/8 dev ipip0
ip route add 10.1.0.0/16 encap ip id 100 dst 172.18.0.7 via 241.18.0.7 dev ipip0 table 100
ip route add 100.1.0.0/16 encap ip id 100 dst 172.18.0.7 via 241.18.0.7 dev ipip0 table 100
ip route add 10.2.0.0/16 encap ip id 100 dst 172.18.0.17 via 241.18.0.21 dev ipip0 table 100
ip route add 100.2.0.0/16 encap ip id 100 dst 172.18.0.17 via 241.18.0.21 dev ipip0 table 100
ip route add 10.4.0.0/16 encap ip id 100 dst 172.18.0.14 via 241.18.0.14 dev ipip0 table 100
ip route add 100.4.0.0/16 encap ip id 100 dst 172.18.0.14 via 241.18.0.14 dev ipip0 table 100
```

### Cluster 4 config

```bash
[root@cluster4-worker submariner]# cat setup.sh
#!/bin/bash
ip link add name ipip0 type ipip local 172.18.0.14 remote any
ip link set ipip0 up
ip addr add 241.18.0.14/8 dev ipip0
ip route add 10.1.0.0/16 encap ip id 100 dst 172.18.0.7 via 241.18.0.7 dev ipip0 table 100
ip route add 100.1.0.0/16 encap ip id 100 dst 172.18.0.7 via 241.18.0.7 dev ipip0 table 100
ip route add 10.2.0.0/16 encap ip id 100 dst 172.18.0.17 via 241.18.0.21 dev ipip0 table 100
ip route add 100.2.0.0/16 encap ip id 100 dst 172.18.0.17 via 241.18.0.21 dev ipip0 table 100
ip route add 10.3.0.0/16 encap ip id 100 dst 172.18.0.21 via 241.18.0.14 dev ipip0 table 100
ip route add 100.3.0.0/16 encap ip id 100 dst 172.18.0.21 via 241.18.0.14 dev ipip0 table 100
```
