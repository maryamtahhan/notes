# TRex traffic generator

TRex is an open source traffic generator. It supports both stateful and stateless traffic generation modes. It supports standard Linux interfaces as well as DPDK.

## 1. Download TRex

```cmd
# mkdir trex
# cd trex
# wget --no-check-certificate https://trex-tgn.cisco.com/trex/release/v2.98.tar.gz
# tar -zxvf v2.98.tar.gz
# cd v2.98
```

## 2. create a /etc/trex_cfg.yaml file

The example below configures trex to use a single interface. The second interface
is a `dummy` interface.

```bash
- port_limit      : 2
  version         : 2
#List of interfaces. Change to suit your setup. Use ./dpdk_setup_ports.py -s to see available options
  interfaces    : ["eno2","dummy"]
  port_info       :  # Port IPs. Change to suit your needs. In case of loopback, you can leave as is.
          - ip         : 2.2.2.1
            default_gw : 2.2.2.2
```

> **_NOTE:_** There are more cfg samples in the cfg directory.

## 3. Generate a traffic stream

Create a copy of stl/bench.py. In this copy modify src ip (range), dst ip (range) and udp checksum params or whatever params you need to configure. An example is shown below

```bash
from trex_stl_lib.api import *
import argparse

class STLBench(object):
    """
    tunables:
        - vm : type (string)
            - var1: creates packets with one variable of src ip in the vm in the range 16.0.0.0 - 16.0.255.255 with auto increment
            - var2: creates packets with two variable, src ip and dst ip, in the vm with auto increment. The src ip range is
                    16.0.0.0 - 16.0.255.255 and the dst ip range is 48.0.0.0 - 48.0.255.255
            - random: create packets with random ip source address within the range 16.0.0.0 - 16.0.255.255
            - size: create packets with random size in the range 60-65490. This is done by the Field Engine so each
                    packet in the stream will have a random size.
            - tuple: make use of tuple variable in the vm. The tuple variable consist of (IP.src, Port.src),
                    where the ip.src values taken from the range 16.0.0.0 - 16.0.255.255 and the ports values taken
                    taken from the range 1234 - 65500.
            - cached: make use of cache with size 255
        size : type (int)
             - define the packet's size in the stream.
        flow : type (string)
            - fs: creates stream with flow stats
            - fsl: creates stream with latency
            - no-fs: creates stream without flow stats
        pg_id : type (int)
                default : 7
            - define the packet group ID
        direction type (int)
            - define the direction of the packets
            - 0: the direction is from src - dst
            - 1: the direction is from dst - src
    """
    ip_range = {}
    ip_range['src'] = {'start': '2.2.2.1', 'end': '2.2.2.1'} # MODIFY THIS RANGE
    ip_range['dst'] = {'start': '2.2.2.3', 'end': '2.2.2.3'} # MODIFY THIS RANGE
    ports = {'min': 1234, 'max': 65500}
    pkt_size = {'min': 64, 'max': 9216}
    imix_table = [ {'size': 68,   'pps': 28,  'isg': 0 },
                   {'size': 590,  'pps': 16,  'isg': 0.1 },
                   {'size': 1514, 'pps':  4,   'isg':0.2 } ]

    def __init__ (self):
        self.pg_id = 0

    def create_stream (self, stl_flow, size, vm, src, dst, pps = 1, isg = 0):
        # Create base packet and pad it to size
        #base_pkt = Ether()/IP(src=src, dst=dst)/UDP(dport=12,sport=1026,chksum=0) # MODIFY THIS for UDP parameters
        base_pkt = Ether()/IP(src=src, dst=dst)/UDP(dport=12,sport=1026)
        pad = max(0, size - len(base_pkt) - 4) * 'x'
        pkt = STLPktBuilder(pkt=base_pkt/pad,
                            vm=vm)
        return STLStream(packet=pkt,
                         mode=STLTXCont(pps=pps),
                         isg=isg,
                         flow_stats=stl_flow)

    def get_streams (self, port_id, direction, tunables, **kwargs):
        parser = argparse.ArgumentParser(description='Argparser for {}'.format(os.path.basename(__file__)),
                                         formatter_class=argparse.ArgumentDefaultsHelpFormatter)
        parser.add_argument('--size',
                            type=str,
                            default=64,
                            help="""define the packet's size in the stream.
                                    choose imix or positive integ
                                    imix - create streams with packets size 60, 590, 1514.
                                    positive integer number - the packets size in the stream.""")
        parser.add_argument('--vm',
                            type=str,
                            default=None,
                            choices={'cached', 'var1', 'var2', 'random', 'tuple', 'size'},
                            help='define the field engine behavior')
        parser.add_argument('--flow',
                            type=str,
                            default="no-fs",
                            choices={'no-fs', 'fs', 'fsl'},
                            help='''Set to fs/fsl if you wants stats per stream.
                                    fs - create streams with flow stats.
                                    fsl - create streams with latency.
                                    no-fs - streams without flow stats''')
        parser.add_argument('--pg_id',
                            type=int,
                            default=7,
                            help='define the packet group ID')

        args = parser.parse_args(tunables)

        size, vm, flow = args.size, args.vm, args.flow
        if size != "imix":
            size = int(size)
        self.pg_id = args.pg_id + port_id
        if direction == 0:
            src, dst = self.ip_range['src'], self.ip_range['dst']
        else:
            src, dst = self.ip_range['dst'], self.ip_range['src']

        vm_var = STLVM()
        if not vm or vm == 'none':
            pass

        elif vm == 'var1':
            vm_var.var(name='src', min_value=src['start'], max_value=src['end'], size=4, op='inc')
            vm_var.write(fv_name='src', pkt_offset='IP.src')
            vm_var.fix_chksum()

        elif vm == 'var2':
            vm_var.var(name='src', min_value=src['start'], max_value=src['end'], size=4, op='inc')
            vm_var.var(name='dst', min_value=dst['start'], max_value=dst['end'], size=4, op='inc')
            vm_var.write(fv_name='src', pkt_offset='IP.src')
            vm_var.write(fv_name='dst', pkt_offset='IP.dst')
            vm_var.fix_chksum()

        elif vm == 'random':
            vm_var.var(name='src', min_value=src['start'], max_value=src['end'], size=4, op='random')
            vm_var.write(fv_name='src', pkt_offset='IP.src')
            vm_var.fix_chksum()

        elif vm == 'tuple':
            vm_var.tuple_var(ip_min=src['start'], ip_max=src['end'], port_min=self.ports['min'], port_max=self.ports['max'], name='tuple')
            vm_var.write(fv_name='tuple.ip', pkt_offset='IP.src')
            vm_var.write(fv_name='tuple.port', pkt_offset='UDP.sport')
            vm_var.fix_chksum()

        elif vm == 'size':
            if size == 'imix':
                raise STLError("Can't use VM of type 'size' with IMIX.")

            size = self.pkt_size['max']
            l3_len_fix = -len(Ether())
            l4_len_fix = l3_len_fix - len(IP())
            vm_var.var(name='fv_rand', min_value=(self.pkt_size['min'] - 4), max_value=(self.pkt_size['max'] - 4), size=2, op='random')
            vm_var.trim(fv_name='fv_rand')
            vm_var.write(fv_name='fv_rand', pkt_offset='IP.len', add_val=l3_len_fix)
            vm_var.write(fv_name='fv_rand', pkt_offset='UDP.len', add_val=l4_len_fix)
            vm_var.fix_chksum()

        elif vm == 'cached':
            vm_var.var(name='src', min_value=src['start'], max_value=src['end'], size=4, op='inc')
            vm_var.write(fv_name='src', pkt_offset='IP.src')
            vm_var.fix_chksum()
            # set VM as cached with 255 cache size of 255
            vm_var.set_cached(255)

        else:
            raise Exception("VM '%s' not available" % vm)


        if flow == 'no-fs':
            stl_flow = None

        elif flow == 'fs':
            stl_flow = STLFlowStats(pg_id=self.pg_id)

        elif flow == 'fsl':
             stl_flow = STLFlowLatencyStats(pg_id=self.pg_id)

        else:
            raise Exception("FLOW '%s' not available" % flow)


        if size == 'imix':
            return [self.create_stream(stl_flow, p['size'], vm_var, src=src['start'], dst=dst['start'], pps=p['pps'], isg=p['isg']) for p in self.imix_table]


        return [self.create_stream(stl_flow, size, vm_var, src=src['start'], dst=dst['start'])]


# dynamic load - used for trex console or simulator
def register():
    return STLBench()
```

## 4. Run TRex

```cmd
# ./t-rex-64 -i
```

## 5. Run the TRex console in another session

```cmd
# ./trex-console
```

## 6. Do ARP resolution

```cmd
# trex>service

Enabling service mode on port(s): [0]                        [SUCCESS]

13.76 [ms]

trex(service)>arp

Resolving destination on port(s) [0]:                        [SUCCESS]

Port 0 - Recieved ARP reply from: 2.2.2.2, hw: ec:f4:bb:c0:b6:28

125.13 [ms]

# trex(service)>service --off
```

## 7. Traffic control

```cmd
# trex>start -f stl/mt_bench.py -m 10% --port 0 -t --size 1200
```

To stop traffic

```cmd
# trex>stop
```

To see traffic stats:

```cmd
# trex>tui
```

This will create a view like the following.

```
Global Statistics

connection   : localhost, Port 4501                       total_tx_L2  : 0 bps
version      : STL @ v2.98                                total_tx_L1  : 0 bps
cpu_util.    : 0.0% @ 1 cores (1 per dual port)           total_rx     : 4.91 Kbps
rx_cpu_util. : 0.0% / 2.96 pps                            total_pps    : 0 pps
async_util.  : 0% / 32.92 bps                             drop_rate    : 0 bps
total_cps.   : 0 cps                                      queue_full   : 0 pkts

Port Statistics

   port    |         0
-----------+------------------
owner      |              root
link       |                UP
state      |              IDLE
speed      |           10 Gb/s
CPU util.  |              0.0%
--         |
Tx bps L2  |             0 bps
Tx bps L1  |             0 bps
Tx pps     |             0 pps
Line Util. |               0 %
---        |
Rx bps     |         4.91 Kbps
Rx pps     |          2.96 pps
----       |
opackets   |                 0
ipackets   |                14
obytes     |                 0
ibytes     |              2693
tx-pkts    |            0 pkts
rx-pkts    |           14 pkts
tx-bytes   |               0 B
rx-bytes   |           2.69 KB
-----      |
oerrors    |                 0
ierrors    |                 0

status:  \

Press 'ESC' for navigation panel...
status: [OK]
```

To exit tui mode type `quit` or `q`

## To configure high performance traffic generation

To configure high performance traffic generation please use DPDK with Trex

Setup DPDK as specified [here](https://doc.dpdk.org/guides/linux_gsg/index.html)

```bash
modprobe vfio-pci
```

Bind your interface to DPDK

```bash
./dpdk/usertools/dpdk-devbind.py -b vfio-pci 01:00.1
```

```bash
- port_limit      : 2
  version         : 2
  c               : 4
#List of interfaces. Change to suit your setup. Use ./dpdk_setup_ports.py -s to see available options
  interfaces    : ["01:00.1","dummy"]
  port_info       :  # Port IPs. Change to suit your needs. In case of loopback, you can leave as is.
          - ip         : 2.2.2.1
            default_gw : 2.2.2.2
```

> **_NOTE:_** multiple cores are enabled with the c : 4 option.

## References

- [TRex manual](https://trex-tgn.cisco.com/trex/doc/trex_manual.html)
- [Full set of TRex manuals](https://trex-tgn.cisco.com/trex/doc/)
- [TRex github](https://github.com/cisco-system-traffic-generator/trex-core)