# Latency testing with TRex

This doc will show how to setup a hairpin/RTT latency test using
Trex.

```bash
          +--------------------------------------------------+
          |   +------------------------------------------+   |
          |   |                                          |   |
          |   |          Simple Forwarding App           |   |
          |   |                                          |   |
          |   +------------------------------------------+   |
          |   |                 NIC                      |   |
          +---+------------------------------------------+---+
                     ^                           :
                     |                           |
                     :                           v
          +---+------------------------------------------+---+
          |   |                 NIC                      |   |
          |   +------------------------------------------+   |
          |   |                                          |   |
          |   |                 Trex                     |   |
          |   +------------------------------------------+   |
          +--------------------------------------------------+

            Figure 1: Latency Test Configuration
```

## Dependencies

This guide assumes you have already downloaded Trex as shown [here](https://github.com/maryamtahhan/xdp-progs/blob/main/trex/README.md#1-download-trex).

Configure hugepages for DPDK:

```bash
$ echo 4096 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

Some forwarding application running on the SUT.

## Trex configuration

For Latency testing Trex is configured to use DPDK and to pin the threads to certain cores (if a single host is used). An example configuration is shown below:

```bash
[host] $ cat /etc/trex_cfg.yaml
- port_limit      : 2
  version         : 2
  c               : 4

  interfaces    : ["01:00.1","dummy"]
  port_info       :
          - ip         : 2.2.2.1
            default_gw : 2.2.2.2
  platform :
        master_thread_id  : 1
        latency_thread_id : 3
        dual_if   :
             - socket   : 1
               threads  : [5, 7, 9, 11]
```

## Running Trex

The Trex server needs to run with the --astf flag:

```bash
[host] $ ./t-rex-64 -i --astf
```

## Run the Trex client

```bash
[host] $ ./trex-console
```

## Complete arp resolution

```bash
trex>service

Enabling service mode on port(s): [0]                        [SUCCESS]

3.39 [ms]

trex(service)>arp

Resolving destination on port(s) [0]:                        [SUCCESS]

Port 0 - Recieved ARP reply from: 2.2.2.2, hw: ec:f4:bb:c0:b6:28

137.78 [ms]

trex(service)>service --off

Disabling service mode on port(s): [0]                       [SUCCESS]

3.46 [ms]
```

## Start the latency test

```bash
trex>latency start --src 2.2.2.1 --dst 2.2.2.3 -p 0 -m 10000
```

## Show the latency results

```bash
trex>latency show
Latency Statistics

  Port ID:   |        0
-------------+----------------
TX pkts      |          641195
RX pkts      |          380000
Max latency  |             587
Avg latency  |              25
-- Window -- |
Last max     |              93
Last-1       |              38
Last-2       |              56
Last-3       |              45
Last-4       |             136
Last-5       |              93
Last-6       |             104
Last-7       |             148
Last-8       |             436
Last-9       |              43
Last-10      |             148
Last-11      |             257
Last-12      |             587
Last-13      |              39
Last-14      |              73
Last-15      |              69
Last-16      |             131
---          |
Jitter       |               4
----         |


trex>latency counters
Latency Counters

  Port ID:   |        0
-------------+----------------
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
m_jitter     |               1
m_pkt_ok     |          638770
m_seq_error  |               1
m_tx_pkt_ok  |          899966
--           |
cnt          |          638744
high_cnt     |          638744
max_usec     |             813
min_usec     |              10
s_avg        |            25.0
s_max        |      146.587143


trex>latency hist
Latency Histogram

  Port ID:   |        0
-------------+----------------
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
             |
700          |               1
300          |              16
200          |              18
100          |             147
90           |             107
80           |              29
70           |              43
60           |             148
50           |             151
40           |            1146
30           |           27217
20           |         1135210
10           |              16
```

## Stop the latency test

```bash
trex>latency stop
```
