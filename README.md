# What is Cisco TRex?
> This is an open-source traffic generator software designed to run on standard Intel processors based on DPDK. It supports both stateful and stateless modes, offering a relatively straightforward and fully scalable solution. <br>
> <br>
> Trex allows the generation of various types of traffic and provides data analysis upon reception. It operates at both MAC and IP levels, enabling users to specify packet sizes and quantities while controlling the data transfer rate. <br>
> <br>
> Interaction with the generator is organized within the Linux environment.  <br>
> <br>
> One of the notable features of the Trex generator is its utilization of DPDK technology, which helps bypass performance bottlenecks in the Linux network stack. DPDK, or Data Plane Development Kit, is a set of libraries and drivers for rapid packet processing. It eliminates the need for the Linux network stack in packet processing, allowing direct interaction with the network device. <br>
> <br>
> DPDK transforms a general-purpose processor into a packet forwarding server, eliminating the need for expensive switches and routers. However, DPDK imposes restrictions on the use of specific network adapters; the list of supported hardware is available on the link provided — Intel's popular platform is well-supported, ensuring compatibility with hardware that works with Linux drivers such as e1000, ixgbe, i40e, ice, fm10k, ipn3ke, ifc, igc. <br>
> <br>
> It's crucial to understand that for the TRex server to operate at 10 Gbps speeds, a multicore processor with at least 4 cores is required, preferably an Intel CPU with simultaneous multithreading (hyper-threading) support. <br>

## Preparing Trex for work
| Data TRex VM                                       |
|-----------------------------------------------------|
| Path to TRex: /opt/trex/v3.00/                      |
| TRex Version: 3.00                                 |
| DPDK version: 22.03.0                              |

### Intermediate Networks for TRex Virtual Router (TRVR)
| For client (VLAN_87):       | For Server (VLAN_88):      |
|---------------------------|---------------------------|
| Default gw: 172.16.9.1/30  | Default gw: 172.16.9.5/30  |
| TRVR: 172.16.9.2/30        | TRVR: 172.16.9.6/30        |

### Emulated networks behind the TRex Virtual Router (TRVR)
| Client Network           | Server Network              |
|--------------------------|-----------------------------|
| 172.16.7.0/24            | 172.16.12.0/24              |

**IMPORTANT: Before starting any work, disable Secure Boot on the TRex VM.**

**Also, before commencing operations, it is necessary to route the routes for emulated networks through the test zone.**

1) Connect to the VM and navigate to the TRex directory:
```
[trexlab@lab-ine-trex-01 v3.00]$ cd /opt/trex/v3.00/
[trexlab@lab-ine-trex-01 v3.00]$ ls
astf              daemon_server          ouvp_stl              _t-rex-64-debug-o
astf_schema.json  doc_process.py         __pycache__           t-rex-64-debug-o
astf-sim          dpdk_nic_bind.py       run_functional_tests  _t-rex-64-o
astf-sim-utl      dpdk_setup_ports.py    run_regression        t-rex-64-o
automation        emu                    so                    trex-cfg
avl               exp                    stl                   trex_client_v3.00.tar.gz
bird              external_libs          stl-sim               trex-console
b.pcap            find_python.sh         test.yaml             trex_daemon_server
bp-sim-64         general_daemon_server  _t-rex-64             trex-emu
bp-sim-64-debug   generated              t-rex-64              trex_emu
cap2              ko                     _t-rex-64-debug       x710_ddp
cap222.pcap       master_daemon.py       t-rex-64-debug        y
cfg               ndr                    t-rex-64-debug-gdb
```
TRex folder structure:
| Location                                               | Description                                       |
|--------------------------------------------------------|---------------------------------------------------|
| /stl                                                   | Stateless native (py) profiles                    |
| /stl/hlt                                               | Stateless HLT profiles                            |
| /ko                                                    | Kernel modules for DPDK                           |
| /external_libs                                         | Python external libs used by server/clients       |
| /exp                                                   | Golden PCAP file for unit-tests                   |
| /cfg                                                   | Examples of config files                          |
| /cap2                                                  | Stateful profiles                                 |
| /avl                                                   | Stateful profiles - EMIX profile                  |
| /automation                                            | Python client/server code for both Stateful and Stateless |
| /automation/regression                                 | Regression for Stateless and Stateful             |
| /automation/config                                     | Regression setups config files                    |
| /automation/trex_control_plane/interactive/trex        | Stateless lib and Console                         |
| /automation/trex_control_plane/interactive/trex/stl    | Stateless lib                                     |
| /automation/trex_control_plane/interactive/trex/examples/stl | Stateless examples                           |
| /astf                                                  | ASTF native (py) profiles                         |
| /automation/trex_control_plane/interactive/trex/examples/astf | Automation examples                     |
| /automation/trex_control_plane/interactive/trex/astf   | ASTF lib compiler (convert py to JSON), Interactive lib |
| /automation/trex_control_plane/stf                      | STF automation (used by ASTF mode)               |

2) Display the table of interfaces available for DPDK configuration:
```
[trexlab@lab-ine-trex-01 v3.00]$ sudo ./dpdk_setup_ports.py -t
+----+------+---------+-------------------+-----------------------------+---------+----------+----------+
| ID | NUMA |   PCI   |        MAC        |            Name             | Driver  | Linux IF |  Active  |
+====+======+=========+===================+=============================+=========+==========+==========+
| 0  | -1   | 0b:00.0 | 00:50:56:a0:1b:92 | VMXNET3 Ethernet Controller | vmxnet3 | ens192   | *Active* |
+----+------+---------+-------------------+-----------------------------+---------+----------+----------+
| 1  | -1   | 13:00.0 | 00:50:56:a0:01:33 | VMXNET3 Ethernet Controller | vmxnet3 | ens224   |          |
+----+------+---------+-------------------+-----------------------------+---------+----------+----------+
| 2  | -1   | 1b:00.0 | 00:50:56:a0:55:ac | VMXNET3 Ethernet Controller | vmxnet3 | ens256   |          |
+----+------+---------+-------------------+-----------------------------+---------+----------+----------+
```
ens192 - management interface for connecting to VM via SSH (cannot be used)

ens224 and ens256 - available for DPDK configuration

3) Let's configure the interfaces according to the TRex VM data specified above:
```
[trexlab@lab-ine-trex-01 v3.00]$ sudo ./dpdk_setup_ports.py -i
By default, IP based configuration file will be created. Do you want to use MAC based config? (y/N)n
+----+------+---------+-------------------+-----------------------------+---------+----------+----------+
| ID | NUMA |   PCI   |        MAC        |            Name             | Driver  | Linux IF |  Active  |
+====+======+=========+===================+=============================+=========+==========+==========+
| 0  | -1   | 0b:00.0 | 00:50:56:a0:1b:92 | VMXNET3 Ethernet Controller | vmxnet3 | ens192   | *Active* |
+----+------+---------+-------------------+-----------------------------+---------+----------+----------+
| 1  | -1   | 13:00.0 | 00:50:56:a0:01:33 | VMXNET3 Ethernet Controller | vmxnet3 | ens224   |          |
+----+------+---------+-------------------+-----------------------------+---------+----------+----------+
| 2  | -1   | 1b:00.0 | 00:50:56:a0:55:ac | VMXNET3 Ethernet Controller | vmxnet3 | ens256   |          |
+----+------+---------+-------------------+-----------------------------+---------+----------+----------+
Please choose an even number of interfaces from the list above, either by ID, PCI or Linux IF
Stateful will use order of interfaces: Client1 Server1 Client2 Server2 etc. for flows.
Stateless can be in any order.
Enter list of interfaces separated by space (for example: 1 3) : 1 2
 
For interface 1, assuming loopback to its dual interface 2.
Putting IP 1.1.1.1, default gw 2.2.2.2 Change it?(y/N).y
Please enter IP address for interface 1: 172.16.9.2
Please enter default gateway for interface 1: 172.16.9.1
For interface 2, assuming loopback to its dual interface 1.
Putting IP 2.2.2.2, default gw 1.1.1.1 Change it?(y/N).y
Please enter IP address for interface 2: 172.16.9.6
Please enter default gateway for interface 2: 172.16.9.5
Print preview of generated config? (Y/n)y
### Config file generated by dpdk_setup_ports.py ###
 
- version: 2
  interfaces: ['13:00.0', '1b:00.0']
  port_info:
      - ip: 172.16.9.2
        default_gw: 172.16.9.1
      - ip: 172.16.9.6
        default_gw: 172.16.9.5
 
  platform:
      master_thread_id: 0
      latency_thread_id: 1
      dual_if:
        - socket: 0
          threads: [2,3,4,5,6,7]
 
 
Save the config to file? (Y/n)y
Default filename is /etc/trex_cfg.yaml
Press ENTER to confirm or enter new file:
File /etc/trex_cfg.yaml already exist, overwrite? (y/N)y
Saved to /etc/trex_cfg.yaml.
```
As indicated in the output, the interface settings will be stored in the configuration file at the path /etc/trex_cfg.yaml.

Next, it is necessary to prepare TRex to run in one of the available modes:

| Stateless Mode Feature                                       | Stateful Mode Feature                                                              |
|-----------------------------------------------------------------|------------------------------------------------------------------------------------|
| Данный режим удобно использовать для расчета потерь при L3/L2 переключениях. | This mode is convenient for calculating losses during L3/L2 switches. |
| - Large scale — Supports about 10–30 million packets per second (Mpps) per core, scalable with the number of cores | - Emulate L7 applications, e.g. HTTP/HTTPS/Citrix                               |
| - Each profile can support multiple streams, scalable to 10K parallel streams | - Multi profile support and ability to group flows                          |
| - Each stream supports:                                       | - Performance and scale:                                                      |
|   - Packet template - ability to build any packet (including malformed) using Scapy (example: MPLS/IPv4/Ipv6/GRE/VXLAN/NSH) |   - High bandwidth - 200 Gb/sec                                             |
|     -- Ability to change any field inside the packet            |   - High connection rate - order of MCPS                                     |
|     -- Ability to change the packet size                        |   - Scale to millions of active established flows                            |
|     -- Mode - Continuous/Burst/Multi-burst                      |   - Benchmark and Stress features/devices like:                              |
|     -- Rate specification in pps, line rate percentage or L1/L2 bandwidth |     -- NAT                                                                   |
|     -- Action - stream can trigger a stream                     |     -- DPI                                                                   |
| - Interactive support - Fast Console, GUI                      |     -- Load Balancer                                                         |
| - Statistics per interface or per stream supported in hardware/software |     -- Network cache devices                                                 |
| - Latency and Jitter per stream                                 |     -- Firewalls                                                             |
| - Blazingly fast Python automation API                          |          **Stateful(STF) vs Advance Stateful (ASTF)**                                                                         |
| - Capture/Monitor traffic with BPF filters - no need for Wireshark |             - Same Flexible tuple generator                                                                    |
| - Capture network traffic by redirecting the traffic to Wireshark |                        - Same Clustering mode                                                         |
| - PCAP file import/export with huge pcap file transmission support (e.g. 1TB pcap file) for DPI |        - Same VLAN support                                                                           |
| - Multi-user support                                           |         - NAT - no need for complex learn mode. ASTF supports NAT64 out of the box. |                                                                        |
|                                          |        - Flow order. ASTF has inherent ordering verification using the TCP layer. It also checks IP/TCP/UDP checksum out of the box. | 
|                                          |       - Latency measurement is supported in both.  | 
|                                          |       - In ASTF mode, you can’t control the IPG, less predictable (concurrent flows are less deterministic) | 
|                                          |        - ASTF can be interactive (start, stop, stats) | 

**Advance Stateful mode will not be considered in this document**

## Stateless mode

In this mode, a Python script is used to launch the generator, specifying all the generation parameters.

## Testing Switching Methods

TRex allows displaying statistics in the console only for 4 streams simultaneously. Therefore, for conducting testing, it is recommended to use a minimum of 4 5-tuple streams with the following parameters:
| № | Description | ID | Source Parameters | Destination | Rate, pps | Frame Size, bytes | Stats |
|---|-------------|----|-------------------|-------------|-----------|---------------------|-------|
| 1 | Direct traffic (client --> server) | 10 | IP:172.16.7.1-172.16.7.60 | 172.16.12.10 | 10000 | 1400 | On |
| 2 | Direct traffic (client --> server) | 20 | IP:172.16.7.61-172.16.7.120 | 172.16.12.20 | 10000 | 1400 | On |
| 3 | Direct traffic (client --> server) | 30 | IP:172.16.7.121-172.16.7.180 | 172.16.12.30 | 10000 | 1400 | On |
| 4 | Direct traffic (client --> server) | 40 | IP:172.16.7.181-172.16.7.240 | 172.16.12.40 | 10000 | 1400 | On |

*Rate (pps, Packets Per Second): The frequency of packet generation per second.

*Frame size (bytes): The size of the generated packet. The recommended size is 1400, but this parameter can be lowered in the presence of packet losses.

*Stats: Determines whether the statistics of this stream will be displayed in the TRex console (specified in the Python script, parameter flow_stats).

Packet losses from all monitored streams at the moment of switching are recorded in the report. Then, the largest number of losses is selected and converted into milliseconds (1 ms = 10 packets at a frequency of 10000 pps).

## Setting up TRex in Stateless mode
1) Prepare the Python script (ouvp_stl/new_marker_streams_p0_p1.py):
```
from trex_stl_lib.api import *
 
def generate_payload(length):
      word = ''
      alphabet_size = len(string.ascii_letters)
      for i in range(length):
          word += string.ascii_letters[(i % alphabet_size)]
      return word
 
class STLS1(object):
 
    def __init__ (self):
        self.fsize       =200;
 
    def create_stream_to_1 (self):
        size = self.fsize - 4;
        #generating 5tuple flows with 802.1p =0 and dscp = be (00, tos = 0), should be dropped
        base_pkt =  Ether()/IP(src="172.16.7.1", dst = "172.16.12.10")/UDP(sport=1025,dport=12)
        vm = STLScVmRaw( [ STLVmTupleGen( ip_min = "172.16.7.1",
                                          ip_max = "172.16.7.60",
                                          port_min = 1025,
                                          port_max = 65535,
                                          name = "tuple"),
                           STLVmWrFlowVar(fv_name="tuple.ip", pkt_offset="IP.src"),
                           STLVmFixIpv4(offset = "IP"),
                           STLVmWrFlowVar(fv_name="tuple.port", pkt_offset="UDP.sport"),
                          ],
                        )
        pkt = STLPktBuilder(pkt=base_pkt/generate_payload(size-len(base_pkt)), vm=vm)
 
        return STLStream( name='1 stream to',
                          packet = pkt,
                          mode = STLTXCont( pps = 10000 ),
                          flow_stats = STLFlowStats(pg_id = 10)
                        )
 
    def create_stream_to_2 (self):
        size = self.fsize - 4;
        #generating 5tuple flows with 802.1p =0 and dscp = be (00, tos = 0), should be dropped
        base_pkt =  Ether()/IP(src="172.16.7.61", dst = "172.16.12.20")/UDP(sport=1025,dport=12)
        vm = STLScVmRaw( [ STLVmTupleGen( ip_min = "172.16.7.61",
                                          ip_max = "172.16.7.120",
                                          port_min = 1025,
                                          port_max = 65535,
                                          name = "tuple"),
                           STLVmWrFlowVar(fv_name="tuple.ip", pkt_offset="IP.src"),
                           STLVmFixIpv4(offset = "IP"),
                           STLVmWrFlowVar(fv_name="tuple.port", pkt_offset="UDP.sport"),
                          ],
                        )
        pkt = STLPktBuilder(pkt=base_pkt/generate_payload(size-len(base_pkt)), vm=vm)
 
        return STLStream( name='2 stream to',
                          packet = pkt,
                          mode = STLTXCont( pps = 10000 ),
                          flow_stats = STLFlowStats(pg_id = 20)
                        )
 
    def create_stream_to_3 (self):
        size = self.fsize - 4;
        #generating 5tuple flows with 802.1p =0 and dscp = be (00, tos = 0), should be dropped
        base_pkt =  Ether()/IP(src="172.16.7.121", dst = "172.16.12.30")/UDP(sport=1025,dport=12)
        vm = STLScVmRaw( [ STLVmTupleGen( ip_min = "172.16.7.121",
                                          ip_max = "172.16.7.180",
                                          port_min = 1025,
                                          port_max = 65535,
                                          name = "tuple"),
                           STLVmWrFlowVar(fv_name="tuple.ip", pkt_offset="IP.src"),
                           STLVmFixIpv4(offset = "IP"),
                           STLVmWrFlowVar(fv_name="tuple.port", pkt_offset="UDP.sport"),
                          ],
                        )
        pkt = STLPktBuilder(pkt=base_pkt/generate_payload(size-len(base_pkt)), vm=vm)
 
        return STLStream( name='3 stream to',
                          packet = pkt,
                          mode = STLTXCont( pps = 10000 ),
                          flow_stats = STLFlowStats(pg_id = 30)
                        )
 
    def create_stream_to_4 (self):
        size = self.fsize - 4;
        #generating 5tuple flows with 802.1p =0 and dscp = be (00, tos = 0), should be dropped
        base_pkt =  Ether()/IP(src="172.16.7.181", dst = "172.16.12.40")/UDP(sport=1025,dport=12)
        vm = STLScVmRaw( [ STLVmTupleGen( ip_min = "172.16.7.181",
                                          ip_max = "172.16.7.240",
                                          port_min = 1025,
                                          port_max = 65535,
                                          name = "tuple"),
                           STLVmWrFlowVar(fv_name="tuple.ip", pkt_offset="IP.src"),
                           STLVmFixIpv4(offset = "IP"),
                           STLVmWrFlowVar(fv_name="tuple.port", pkt_offset="UDP.sport"),
                          ],
                        )
        pkt = STLPktBuilder(pkt=base_pkt/generate_payload(size-len(base_pkt)), vm=vm)
 
        return STLStream( name='4 stream to',
                          packet = pkt,
                          mode = STLTXCont( pps = 10000 ),
                          flow_stats = STLFlowStats(pg_id = 40)
                        )
 
 
 
    def get_streams (self, direction = 0, **kwargs):
        # create 4 stream
        #self.streams = streams
        return [
          self.create_stream_to_1(),
          self.create_stream_to_2(),
          self.create_stream_to_3(),
          self.create_stream_to_4(),
          ]
 
# dynamic load - used for trex console or simulator
def register():
    return STLS1()
```
It is recommended to review the TRex documentation and scripts located in the /stl directory.

2) Open TRex in two windows. In the first window, launch TRex in stateless mode:
```
[trexlab@lab-ine-trex-01 v3.00]$ sudo ./t-rex-64 -i
 
-Per port stats table
      ports |               0 |               1
 -----------------------------------------------------------------------------------------
   opackets |               0 |               0
     obytes |               0 |               0
   ipackets |               0 |               0
     ibytes |               0 |               0
    ierrors |               0 |               0
    oerrors |               0 |               0
      Tx Bw |       0.00  bps |       0.00  bps
 
-Global stats enabled
 Cpu Utilization : 0.0  %
 Platform_factor : 1.0
 Total-Tx        :       0.00  bps
 Total-Rx        :       0.00  bps
 Total-PPS       :       0.00  pps
 Total-CPS       :       0.00  cps
 
 Expected-PPS    :       0.00  pps
 Expected-CPS    :       0.00  cps
 Expected-BPS    :       0.00  bps
 
 Active-flows    :        0  Clients :        0   Socket-util : 0.0000 %
 Open-flows      :        0  Servers :        0   Socket :        0 Socket/Clients :  -nan
 drop-rate       :       0.00  bps
 current time    : 3.6 sec
 test duration   : 0.0 sec
```
3) In the second window, access the TRex console:
```
[trexlab@lab-ine-trex-01 v3.00]$ sudo ./trex-console
 
Using 'python3' as Python interpeter
 
 
Connecting to RPC server on localhost:4501                   [SUCCESS]
 
 
Connecting to publisher server on localhost:4500             [SUCCESS]
 
 
Acquiring ports [0, 1]:                                      [SUCCESS]
 
 
Server Info:
 
Server version:   v3.00 @ STL
Server mode:      Stateless
Server CPU:       1 x Intel(R) Xeon(R) Gold 6252 CPU @ 2.10GHz
Ports count:      2 x 10Gbps @ VMXNET3 Ethernet Controller
 
-=TRex Console v3.0=-
 
Type 'help' or '?' for supported actions
 
trex>
```
4) In the second window, let's start the generator using the previously prepared script:

```
trex>start -f ouvp_stl/new_marker_streams_p0_p1.py --port 0 -d 180
```

   -f <file>: Specifies the path to the Python script.

   --port <num>: Indicates the outgoing port of the generator; without this key, generation will be performed on all available TRex ports.

   -d <num>: Sets the duration of generation in seconds (default value is 3600 seconds).


5) In the second window, open the graphical interface of TRex:
```
trex>tui
 
Global Statistics
 
connection   : localhost, Port 4501                       total_tx_L2  : 64.09 Mbps
version      : STL @ v3.00                                total_tx_L1  : 70.5 Mbps
cpu_util.    : 0.74% @ 1 cores (1 per dual port)          total_rx     : 64.09 Mbps
rx_cpu_util. : 0.34% / 40.06 Kpps                         total_pps    : 40.06 Kpps
async_util.  : 0% / 12.73 bps                             drop_rate    : 0 bps
total_cps.   : 0 cps                                      queue_full   : 0 pkts
 
Port Statistics
 
   port    |         0         |         1         |       total
-----------+-------------------+-------------------+------------------
owner      |              root |              root |
link       |                UP |                UP |
state      |      TRANSMITTING |              IDLE |
speed      |           10 Gb/s |           10 Gb/s |
CPU util.  |             0.74% |              0.0% |
--         |                   |                   |
Tx bps L2  |        64.09 Mbps |             0 bps |        64.09 Mbps
Tx bps L1  |         70.5 Mbps |             0 bps |         70.5 Mbps
Tx pps     |        40.06 Kpps |             0 pps |        40.06 Kpps
Line Util. |            0.71 % |               0 % |
---        |                   |                   |
Rx bps     |             0 bps |        64.09 Mbps |        64.09 Mbps
Rx pps     |             0 pps |        40.06 Kpps |        40.06 Kpps
----       |                   |                   |
opackets   |           1476516 |                 0 |           1476516
ipackets   |                 0 |           1476516 |           1476516
obytes     |         295303200 |                 0 |         295303200
ibytes     |                 0 |         295303200 |         295303200
tx-pkts    |        1.48 Mpkts |            0 pkts |        1.48 Mpkts
rx-pkts    |            0 pkts |        1.48 Mpkts |        1.48 Mpkts
tx-bytes   |          295.3 MB |               0 B |          295.3 MB
rx-bytes   |               0 B |          295.3 MB |          295.3 MB
-----      |                   |                   |
oerrors    |                 0 |                 0 |                 0
ierrors    |                 0 |                 0 |                 0
 
status:  /
 
Press 'ESC' for navigation panel...
status:
 
tui>
```
6) Enable the mode for displaying stream statistics by pressing the keys "Esc," "S," and then "Esc" again in sequence:

```
Global Statistics
 
connection   : localhost, Port 4501                       total_tx_L2  : 64.17 Mbps
version      : STL @ v3.00                                total_tx_L1  : 70.58 Mbps
cpu_util.    : 0.83% @ 1 cores (1 per dual port)          total_rx     : 64.18 Mbps
rx_cpu_util. : 0.31% / 40.11 Kpps                         total_pps    : 40.1 Kpps
async_util.  : 0% / 8.93 bps                              drop_rate    : 0 bps
total_cps.   : 0 cps                                      queue_full   : 0 pkts
 
Streams Statistics
 
  PG ID    |        10         |        20         |        30         |        40
-----------+-------------------+-------------------+-------------------+------------------
Tx pps     |           10 Kpps |           10 Kpps |           10 Kpps |           10 Kpps
Tx bps L2  |           16 Mbps |           16 Mbps |           16 Mbps |           16 Mbps
Tx bps L1  |         17.6 Mbps |         17.6 Mbps |         17.6 Mbps |         17.6 Mbps
---        |                   |                   |                   |
Rx pps     |           10 Kpps |           10 Kpps |           10 Kpps |           10 Kpps
Rx bps     |           16 Mbps |           16 Mbps |           16 Mbps |           16 Mbps
----       |                   |                   |                   |
opackets   |           2074074 |           2074074 |           2074074 |           2074074
ipackets   |           2074074 |           2074074 |           2074074 |           2074074
obytes     |         414814800 |         414814800 |         414814800 |         414814800
ibytes     |         414813800 |         414813800 |         414813800 |         414813800
-----      |                   |                   |                   |
opackets   |        2.07 Mpkts |        2.07 Mpkts |        2.07 Mpkts |        2.07 Mpkts
ipackets   |        2.07 Mpkts |        2.07 Mpkts |        2.07 Mpkts |        2.07 Mpkts
obytes     |         414.81 MB |         414.81 MB |         414.81 MB |         414.81 MB
ibytes     |         414.81 MB |         414.81 MB |         414.81 MB |         414.81 MB
 
status:  /
 
Press 'ESC' for navigation panel...
status:
 
tui>
```
We are interested in discrepancies in the values of "opackets" and "ipackets" in each stream.

You can stop the generator using the "stop" command.

Exit from the graphical interface and console can be done using the "q" command.

7) Perform the switch and record packet losses in each stream in the report.

8) For repeated testing, follow these steps:

* In the first window, restart TRex in stateless mode.

* In the second window, re-enter the TRex console and repeat all actions.

## Stateful mode
In this mode, two configuration files are used to launch the generator, specifying all the generation parameters:

1. A file indicating the primary network configuration of TRex, which is by default the file /etc/trex_cfg.yaml, as configured earlier.

2. A file describing the profile of the generated traffic. These files are typically stored in the directories /cap2 and /avl.

Configuration Process for TRex in Stateful Mode
1) Verify the first configuration file (/etc/trex_cfg.yaml):
```
### Config file generated by dpdk_setup_ports.py ###
 
- version: 2
  interfaces: ['13:00.0', '1b:00.0']
  port_info:
      - ip: 172.16.9.2
        default_gw: 172.16.9.1
      - ip: 172.16.9.6
        default_gw: 172.16.9.5
 
  platform:
      master_thread_id: 0
      latency_thread_id: 1
      dual_if:
        - socket: 0
          threads: [2,3,4,5,6,7]
```
2) Verify the second configuration file (/cap2/fw-int.yaml):
```
- duration : 40
  generator :
          distribution : "seq"
          clients_start : "172.16.7.1"
          clients_end   : "172.16.7.255"
          servers_start : "172.16.12.1"
          servers_end   : "172.16.12.255"
          clients_per_gb : 0
          min_clients    : 0
          dual_port_mask : "1.0.0.0"
          tcp_aging      : 0
          udp_aging      : 0
  cap_info :
     - name: cap2/Oracle.pcap
       cps : 1.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/Video_Calls.pcap
       cps : 11.4
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/rtp_160k.pcap
       cps : 3.6
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/rtp_250k_rtp_only_1.pcap
       cps : 4.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/rtp_250k_rtp_only_2.pcap
       cps : 4.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/smtp.pcap
       cps : 34.2
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/Voice_calls_rtp_only.pcap
       cps : 66.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/citrix.pcap
       cps : 105.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/dns.pcap
       cps : 240.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/exchange.pcap
       cps : 63.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/http_browsing.pcap
       cps : 267.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/http_get.pcap
       cps : 34.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/http_post.pcap
       cps : 345.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/https.pcap
       cps : 111.0
       ipg : 10000
       rtt : 10000
       w   : 4
     - name: cap2/mail_pop.pcap
       cps : 34.2
       ipg : 10000
       rtt : 10000
       w   : 4
```
*distribution : Currently, only sequential distribution is supported in IP allocation. This means the IP address is increased by one for each flow;

*clients_per_gb : Not used

*min_clients : Not used

*dual_port_mask : Currently, there is one global IP pool for clients and servers. It serves all templates. All templates will allocate IP from this global pool. Each TRex client/server "dual-port" (pair of ports, such as port 0 for client, port 1 for server) has its own generator offset, taken from the config file. The offset is called dual_port_mask

*tcp_aging : Time in sec to linger the deallocation of TCP flows (in particular return the src_port to the pool). Good for cases when there is a very high socket utilization (>50%) and there is a need to verify that socket source port are not wrapped and reuse. Default value is zero. Better to keep it like that from performance point of view. High value could create performance penalty

*udp_aging : Same as tcp_aging for UDP flows

*name : The name of the template pcap file. The pcap file should include only one flow. (Exception: in case of plug-ins)

*cps : Number of connections per second to generate. In the example, 1.0 means 1 connection per secod

*ipg : inter-packet gap in microseconds

*rtt : should be the same as ipg

*w : This indicates to the IP generator how to generate the flows. If w=2, two flows from the same template will be generated in a burst (more for HTTP that has burst of flows)
3) Launch the generator using the previously prepared configuration files:
```
[trexlab@lab-ine-trex-01 v3.00]$ sudo ./t-rex-64 --cfg /etc/trex_cfg.yaml -f cap2/fw-int.yaml -d 30 -m 1 --nc
```
--cfg <file> : Use file as TRex config file instead of the default /etc/trex_cfg.yaml

-f <file> : YAML file with traffic template configuration (Will run TRex in 'stateful' mode)

-d <num> : Duration of the test in sec (default is 3600)

-m <num> : Rate multiplier. Multiply basic rate of templates by this number

--nc : If set, will not wait for all flows to be closed, before terminating - see manual for more information
