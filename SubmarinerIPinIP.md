Deploy 2 VXLAN submariner clusters from submariner-operator using

```
make deploy using=vxlan,lighthouse
```

This will produce 2 clusters:

```
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
```
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

```
kubectl exec -n submariner-operator --stdin --tty submariner-gateway-tkw4m  -- /bin/bash
```

To setup an IP in IP interface on each cluster gateway do the following:
Once in the pod, bring down the vxlan-tunnel interface and delete its IP address.

On Cluster1-worker:
```
ip link add name ipip0 type ipip local 172.18.0.8 remote 172.18.0.6
ip link set ipip0 up
ip addr add 240.18.0.8/8 dev ipip0
ip route add 100.2.0.0/16 via 241.18.0.6 dev ipip0
ip route add 10.2.0.0/16 via 241.18.0.6 dev ipip0
```

On Cluster2-worker:
```
ip link add name ipip0 type ipip local 172.18.0.6 remote 172.18.0.8
ip link set ipip0 up
ip addr add 240.18.0.6/8 dev ipip0
ip route add 100.1.0.0/16 via 241.18.0.8 dev ipip0
ip route add 10.1.0.0/16 via 241.18.0.8 dev ipip0
```

ON BOTH NODES
```
iptables -A INPUT -p 94 -j SUBMARINER-INPUT
iptables -A SUBMARINER-INPUT -p 94 -j ACCEPT
iptables -A FORWARD -o ipip0 -j ACCEPT
echo 2 > /proc/sys/net/ipv4/conf/ipip0/rp_filter
```

Once this is done you should be able to ping from one tunnel endpoint to the other.

NOTE: there's still an issue with this configuration as services can't be accessed across clusters.



