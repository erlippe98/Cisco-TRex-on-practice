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
