# Submariner VXLAN Cable Driver Overview

The following is a topology overview for a simple Submariner deployment deployed with the VXLAN Cable Driver:

```bash
# make deploy using=lighthouse,vxlan
```

The idea is to show a simple interaction (a ping from one pod in one cluster to another pod in a second cluster) when Submariner is used.

![Submariner VXLAN cable driver logical view](/images/VXLAN_GW.png)
