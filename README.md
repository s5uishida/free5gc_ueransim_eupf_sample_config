# free5GC 5GC & UERANSIM UE / RAN Sample Configuration - eUPF(eBPF/XDP UPF)
This describes a simple configuration for working free5GC and eUPF(eBPF/XDP UPF).
In particular, see [here](https://github.com/s5uishida/install_eupf) for eUPF.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of free5GC 5GC Simulation Mobile Network](#overview)
- [Changes in configuration files of free5GC 5GC, eUPF and UERANSIM UE / RAN](#changes)
  - [Changes in configuration files of free5GC 5GC C-Plane](#changes_cp)
  - [Changes in configuration files of eUPF](#changes_up)
  - [Changes in configuration files of UERANSIM UE / RAN](#changes_ueransim)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE (IMSI-001010000000000)](#changes_ue)
- [Network settings of free5GC 5GC, eUPF and UERANSIM UE / RAN](#network_settings)
  - [Network settings of eUPF and Data Network Gateway](#network_settings_up)
- [Build free5GC, eUPF and UERANSIM](#build)
- [Run free5GC 5GC, eUPF and UERANSIM UE / RAN](#run)
  - [Run eUPF](#run_up)
  - [Run free5GC 5GC C-Plane](#run_cp)
  - [Run UERANSIM](#run_ueran)
    - [Start gNB](#start_gnb)
    - [Start UE](#start_ue)
- [Ping google.com](#ping)
  - [Case for going through DN 10.60.0.0/16](#ping_1)
- [Changelog (summary)](#changelog)

---

<a id="overview"></a>

## Overview of free5GC 5GC Simulation Mobile Network

This describes a simple configuration of C-Plane, eBPF/XDP UPF and Data Network Gateway for free5GC.
**Note that this configuration is implemented with Proxmox VE VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / eBPF/XDP UPF / UE / RAN used are as follows.
- 5GC - free5GC v4.0.1 (2025.04.27) - https://github.com/free5gc/free5gc
- eBPF/XDP UPF - eUPF v0.7.1 (2025.04.16) - https://github.com/edgecomllc/eupf
- UE / RAN - UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Mem<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | free5GC 5GC C-Plane | 192.168.0.141/24 | Ubuntu 24.04 | 1 | 2GB | 20GB |
| VM-UP | eUPF U-Plane | 192.168.0.151/24 | Ubuntu 24.04 | 1 | 2GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
| VM | Device | Model | Linux Bridge | IP address | Interface | XDP |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | ens18 | VirtIO | vmbr1 | 10.0.0.141/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.141/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr4 | 192.168.14.141/24 | N4 | -- |
| VM-UP | ~~ens18~~ | ~~VirtIO~~ | ~~vmbr1~~ | ~~10.0.0.151/24~~ | ~~(NAPT NW)~~ ***down*** | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.151/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.151/24 | N3 | x |
| | ens21 | VirtIO | vmbr4 | 192.168.14.151/24 | N4 | -- |
| | ens22 | VirtIO | vmbr6 | 192.168.16.151/24 | N6 | x |
| VM-DN | ens18 | VirtIO | vmbr1 | 10.0.0.152/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.152/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr6 | 192.168.16.152/24 | N6,<br>***default GW for VM-UP*** | -- |
| VM2 | ens18 | VirtIO | vmbr1 | 10.0.0.131/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.131/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.131/24 | N3 | -- |
| VM3 | ens18 | VirtIO | vmbr1 | 10.0.0.132/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.132/24 | (Mgmt NW) | -- |


Linux Bridges of Proxmox VE are as follows.
| Linux Bridge | Network CIDR | Interface |
| --- | --- | --- |
| vmbr1 | 10.0.0.0/24 | NAPT NW |
| mgbr0 | 192.168.0.0/24 | Mgmt NW |
| vmbr3 | 192.168.13.0/24 | N3 |
| vmbr4 | 192.168.14.0/24 | N4 |
| vmbr6 | 192.168.16.0/24 | N6 |

Subscriber Information (other information is the same) is as follows.  
**Note. Please select OP or OPc according to the setting of UERANSIM UE configuration file.**
| UE | IMSI | DNN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000000 | internet | OPc |

I registered these information with the free5GC WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

The DN is as follows.
| DN | DNN | TUNnel interface of UE |
| --- | --- | --- |
| 10.60.0.0/16 | internet | uesimtun0 |

<a id="changes"></a>

## Changes in configuration files of free5GC 5GC, eUPF and UERANSIM UE / RAN

Please refer to the following for building free5GC, eUPF and UERANSIM respectively.
- free5GC v4.0.1 (2025.04.27) - https://free5gc.org/guide/
- eUPF v0.7.1 (2025.04.16) - https://github.com/s5uishida/install_eupf
- UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of free5GC 5GC C-Plane

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2024-10-13 05:09:24.000000000 +0900
+++ amfcfg.yaml 2025-05-04 19:42:17.462006265 +0900
@@ -5,7 +5,7 @@
 configuration:
   amfName: AMF # the name of this AMF
   ngapIpList:  # the IP list of N2 interfaces on this AMF
-    - 127.0.0.18
+    - 192.168.0.141
   ngapPort: 38412 # the SCTP port listened by NGAP
 
   # Service-based Interface (SBI) Configuration
@@ -30,22 +30,22 @@
   servedGuamiList:
     # <GUAMI> = <MCC><MNC><AMF ID>
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       amfId: cafe00 # AMF identifier (3 bytes hex string, range: 000000~FFFFFF)
 
   # the TAI (Tracking Area Identifier) list supported by this AMF
   supportTaiList:
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       tac: 000001 # Tracking Area Code (3 bytes hex string, range: 000000~FFFFFF)
 
   # the PLMNs (Public land mobile network) list supported by this AMF
   plmnSupportList:
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       snssaiList: # the S-NSSAI (Single Network Slice Selection Assistance Information) list supported by this AMF
         - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
           sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
```
- `free5gc/config/ausfcfg.yaml`
```diff
--- ausfcfg.yaml.orig   2024-09-01 09:47:28.000000000 +0900
+++ ausfcfg.yaml        2024-09-01 09:55:54.000000000 +0900
@@ -16,10 +16,8 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
   plmnSupportList: # the PLMNs (Public Land Mobile Network) list supported by this AUSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
-    - mcc: 123 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 45  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   groupId: ausfGroup001 # ID for the group of the AUSF
   eapAkaSupiImsiPrefix: false # including "imsi-" prefix or not when using the SUPI to do EAP-AKA' authentication

```
- `free5gc/config/nrfcfg.yaml`
```diff
--- nrfcfg.yaml.orig    2024-09-01 09:47:28.000000000 +0900
+++ nrfcfg.yaml 2024-09-01 09:56:10.000000000 +0900
@@ -18,8 +18,8 @@
       key: cert/root.key
     oauth: true
   DefaultPlmnId:
-    mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-    mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+    mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   serviceNameList: # the SBI services provided by this NRF, refer to TS 29.510
     - nnrf-nfm # Nnrf_NFManagement service
     - nnrf-disc # Nnrf_NFDiscovery service
```
- `free5gc/config/nssfcfg.yaml`
```diff
--- nssfcfg.yaml.orig   2024-09-01 09:47:28.000000000 +0900
+++ nssfcfg.yaml        2024-09-01 09:56:46.000000000 +0900
@@ -18,12 +18,12 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
   supportedPlmnList: # the PLMNs (Public land mobile network) list supported by this NSSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   supportedNssaiInPlmnList: # Supported S-NSSAI List for each PLMN
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       supportedSnssaiList: # Supported S-NSSAIs of the PLMN
         - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
           sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
```
- `free5gc/config/smfcfg.yaml`
```diff
--- smfcfg.yaml.orig    2024-10-13 05:09:24.000000000 +0900
+++ smfcfg.yaml 2025-05-04 21:00:46.538990509 +0900
@@ -42,16 +42,16 @@
 
   # Optional: PLMN IDs configuration.
   plmnList:
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   locality: area1 # Name of the location where a set of AMF, SMF, PCF and UPFs are located
 
   # PFCP (Packet Forwarding Control Protocol) configuration for N4 interface.
   pfcp:
     # addr config is deprecated in smf config v1.0.3, please use the following config
-    nodeID: 127.0.0.1 # the Node ID of this SMF
-    listenAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
-    externalAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    nodeID: 192.168.14.141 # the Node ID of this SMF
+    listenAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    externalAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
     assocFailAlertInterval: 10s
     assocFailRetryInterval: 30s
     heartbeatInterval: 10s
@@ -63,8 +63,8 @@
         type: AN # the type of the node (AN or UPF)
       UPF: # the name of the node
         type: UPF # the type of the node (AN or UPF)
-        nodeID: 127.0.0.8 # the Node ID of this UPF
-        addr: 127.0.0.8 # the IP/FQDN of N4 interface on this UPF (PFCP)
+        nodeID: 192.168.14.151 # the Node ID of this UPF
+        addr: 192.168.14.151 # the IP/FQDN of N4 interface on this UPF (PFCP)
         sNssaiUpfInfos: # S-NSSAI information list for this UPF
           - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
               sst: 1 # Slice/Service Type (uinteger, range: 0~255)
@@ -91,7 +91,7 @@
         interfaces: # Interface list for this UPF
           - interfaceType: N3 # the type of the interface (N3 or N9)
             endpoints: # the IP address of this N3/N9 interface on this UPF
-              - 127.0.0.8
+              - 192.168.13.151
             networkInstances: # Data Network Name (DNN)
               - internet
 
@@ -99,7 +99,7 @@
     links:
       - A: gNB1
         B: UPF
-
+  ulcl: false
   # retransmission timer for PDU session modification command
   t3591:
     enable: true # true or false
```

<a id="changes_up"></a>

### Changes in configuration files of eUPF

See [here](https://github.com/s5uishida/install_eupf#create-configuration-file) for the original file.

- `eupf/config.yml`  
```diff
--- config.yml.orig     2025-03-16 14:23:10.971766289 +0900
+++ config.yml  2025-05-04 20:01:22.764630769 +0900
@@ -3,7 +3,7 @@
 api_address: :8080
 pfcp_address: 192.168.14.151:8805
 pfcp_node_id: 192.168.14.151
-pfcp_remote_node: 192.168.14.111
+pfcp_remote_node: 192.168.14.141
 association_setup_timeout: 5
 metrics_address: :9090
 n3_address: 192.168.13.151
```

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `UERANSIM/config/free5gc-gnb.yaml`
```diff
--- free5gc-gnb.yaml.orig       2024-12-11 20:31:30.000000000 +0900
+++ free5gc-gnb.yaml    2025-05-04 19:47:48.421731012 +0900
@@ -1,17 +1,17 @@
-mcc: '208'          # Mobile Country Code value
-mnc: '93'           # Mobile Network Code value (2 or 3 digits)
+mcc: '001'          # Mobile Country Code value
+mnc: '01'           # Mobile Network Code value (2 or 3 digits)
 
 nci: '0x000000010'  # NR Cell Identity (36-bit)
 idLength: 32        # NR gNB ID length in bits [22...32]
 tac: 1              # Tracking Area Code
 
-linkIp: 127.0.0.1   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
-ngapIp: 127.0.0.1   # gNB's local IP address for N2 Interface (Usually same with local IP)
-gtpIp: 127.0.0.1    # gNB's local IP address for N3 Interface (Usually same with local IP)
+linkIp: 192.168.0.131   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
+ngapIp: 192.168.0.131   # gNB's local IP address for N2 Interface (Usually same with local IP)
+gtpIp: 192.168.13.131    # gNB's local IP address for N3 Interface (Usually same with local IP)
 
 # List of AMF address information
 amfConfigs:
-  - address: 127.0.0.1
+  - address: 192.168.0.141
     port: 38412
 
 # List of supported S-NSSAIs by this gNB
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE (IMSI-001010000000000)

- `UERANSIM/config/free5gc-ue.yaml`
```diff
--- free5gc-ue.yaml.orig        2025-03-16 15:49:12.000000000 +0900
+++ free5gc-ue.yaml     2025-05-04 19:49:27.180000029 +0900
@@ -1,9 +1,9 @@
 # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
-supi: 'imsi-208930000000001'
+supi: 'imsi-001010000000000'
 # Mobile Country Code value of HPLMN
-mcc: '208'
+mcc: '001'
 # Mobile Network Code value of HPLMN (2 or 3 digits)
-mnc: '93'
+mnc: '01'
 # SUCI Protection Scheme : 0 for Null-scheme, 1 for Profile A and 2 for Profile B
 protectionScheme: 0
 # Home Network Public Key for protecting with SUCI Profile A
@@ -31,7 +31,7 @@
 
 # List of gNB IP addresses for Radio Link Simulation
 gnbSearchList:
-  - 127.0.0.1
+  - 192.168.0.131
 
 # UAC Access Identities Configuration
 uacAic:
```

<a id="network_settings"></a>

## Network settings of free5GC 5GC, eUPF and UERANSIM UE / RAN

<a id="network_settings_up"></a>

### Network settings of eUPF and Data Network Gateway

See [this1](https://github.com/s5uishida/install_eupf#setup-eupf-on-vm-up) and [this2](https://github.com/s5uishida/install_eupf#setup-data-network-gateway-on-vm-dn).

<a id="build"></a>

## Build free5GC, eUPF and UERANSIM

Please refer to the following for building free5GC, eUPF and UERANSIM respectively.
- free5GC v4.0.1 (2025.04.27) - https://free5gc.org/guide/
- eUPF v0.7.1 (2025.04.16) - https://github.com/s5uishida/install_eupf
- UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on free5GC 5GC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

**Note. The installation guide also includes instructions on building the latest committed version.**

<a id="run"></a>

## Run free5GC 5GC, eUPF and UERANSIM UE / RAN

First run eUPF, then the 5GC and UERANSIM (UE & RAN implementation).

<a id="run_up"></a>

### Run eUPF

See [this](https://github.com/s5uishida/install_eupf#run-eupf-on-vm-up).

<a id="run_cp"></a>

### Run free5GC 5GC C-Plane

Next, run free5GC 5GC C-Plane.
Create the following shell script and run it.
```bash
#!/usr/bin/env bash

PID_LIST=()

NF_LIST="nrf amf smf udr pcf udm nssf ausf chf nef"

export GIN_MODE=release

for NF in ${NF_LIST}; do
    ./bin/${NF} &
    PID_LIST+=($!)
    sleep 1
done

function terminate()
{
    sudo kill -SIGTERM ${PID_LIST[${#PID_LIST[@]}-2]} ${PID_LIST[${#PID_LIST[@]}-1]}
    sleep 2
}

trap terminate SIGINT
wait ${PID_LIST}
```
The PFCP association log between eUPF and free5GC SMF is as follows.
```
2025/05/04 21:05:53 INF Got Association Setup Request from: 192.168.14.141
2025/05/04 21:05:53 INF 
Association Setup Request:
  Node ID: 192.168.14.141
  Recovery Time: 2025-05-04 21:05:52 +0900 JST

2025/05/04 21:05:53 INF Saving new association: &{ID:192.168.14.141 Addr:192.168.14.141 NextSessionID:1 NextSequenceID:1 Sessions:map[] HeartbeatChannel:0xc0001fa540 HeartbeatsActive:false Mutex:{_:{} mu:{state:0 sema:0}}}
```

<a id="run_ueran"></a>

### Run UERANSIM

Here, the case of UE (IMSI-001010000000000) & RAN is described.
First, do an NG Setup between gNodeB and 5GC, then register the UE with 5GC and establish a PDU session.

Please refer to the following for usage of UERANSIM.

https://github.com/aligungr/UERANSIM/wiki/Usage


<a id="start_gnb"></a>

#### Start gNB

Start gNB as follows.
```
# ./nr-gnb -c ../config/free5gc-gnb.yaml
UERANSIM v3.2.7
[2025-05-04 21:06:50.754] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2025-05-04 21:06:50.757] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2025-05-04 21:06:50.757] [sctp] [debug] SCTP association setup ascId[9]
[2025-05-04 21:06:50.757] [ngap] [debug] Sending NG Setup Request
[2025-05-04 21:06:50.759] [ngap] [debug] NG Setup Response received
[2025-05-04 21:06:50.759] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2025-05-04T21:06:50.774396882+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:56315
2025-05-04T21:06:50.775037955+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:56315
2025-05-04T21:06:50.775478564+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Handle NGSetupRequest
2025-05-04T21:06:50.775585927+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Send NG-Setup response
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.7
[2025-05-04 21:07:29.833] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2025-05-04 21:07:29.834] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2025-05-04 21:07:29.834] [nas] [info] Selected plmn[001/01]
[2025-05-04 21:07:29.834] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2025-05-04 21:07:29.834] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2025-05-04 21:07:29.834] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2025-05-04 21:07:29.834] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2025-05-04 21:07:29.836] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2025-05-04 21:07:29.836] [nas] [debug] Sending Initial Registration
[2025-05-04 21:07:29.836] [rrc] [debug] Sending RRC Setup Request
[2025-05-04 21:07:29.836] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2025-05-04 21:07:29.836] [rrc] [info] RRC connection established
[2025-05-04 21:07:29.836] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2025-05-04 21:07:29.837] [nas] [info] UE switches to state [CM-CONNECTED]
[2025-05-04 21:07:29.890] [nas] [debug] Authentication Request received
[2025-05-04 21:07:29.890] [nas] [debug] Received SQN [00000000002C]
[2025-05-04 21:07:29.890] [nas] [debug] SQN-MS [000000000000]
[2025-05-04 21:07:29.913] [nas] [debug] Security Mode Command received
[2025-05-04 21:07:29.913] [nas] [debug] Selected integrity[2] ciphering[0]
[2025-05-04 21:07:30.068] [nas] [debug] Registration accept received
[2025-05-04 21:07:30.068] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2025-05-04 21:07:30.068] [nas] [debug] Sending Registration Complete
[2025-05-04 21:07:30.068] [nas] [info] Initial Registration is successful
[2025-05-04 21:07:30.068] [nas] [debug] Sending PDU Session Establishment Request
[2025-05-04 21:07:30.069] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2025-05-04 21:07:30.269] [nas] [debug] Configuration Update Command received
[2025-05-04 21:07:30.426] [nas] [debug] PDU Session Establishment Accept received
[2025-05-04 21:07:30.426] [nas] [info] PDU Session establishment is successful PSI[1]
[2025-05-04 21:07:30.445] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2025-05-04T21:07:29.865285871+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Handle InitialUEMessage
2025-05-04T21:07:29.865330346+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2025-05-04T21:07:29.865364558+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2025-05-04T21:07:29.865409129+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2025-05-04T21:07:29.865476972+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2025-05-04T21:07:29.865483607+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2025-05-04T21:07:29.865490559+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2025-05-04T21:07:29.865497755+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2025-05-04T21:07:29.865507939+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2025-05-04T21:07:29.865514047+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2025-05-04T21:07:29.866405843+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.867997245+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.870302443+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.871250539+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:29.872473438+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |
2025-05-04T21:07:29.873472728+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.874882873+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.877562044+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.878983108+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2025-05-04T21:07:29.879316283+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2025-05-04T21:07:29.880034825+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.881269460+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.883465320+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.884349764+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:29.885652405+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |
2025-05-04T21:07:29.888809692+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.889856670+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.892891955+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.894737500+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2025-05-04T21:07:29.895515666+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.896913307+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.899537740+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.899929205+09:00 [INFO][UDM][Suci] scheme 0
2025-05-04T21:07:29.900028074+09:00 [INFO][UDM][Suci] SUPI type is IMSI
2025-05-04T21:07:29.900312798+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.901415162+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.903554552+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.904358812+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:29.905144460+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |
2025-05-04T21:07:29.909753397+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2025-05-04T21:07:29.910485280+09:00 [INFO][UDM][Proc] ModifyAuthenticationSubscriptionRequest:  [{replace /sequenceNumber  { 00000000002d map[] 0 }}]
2025-05-04T21:07:29.914355520+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2025-05-04T21:07:29.914784363+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |
2025-05-04T21:07:29.915349827+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2025-05-04T21:07:29.915519478+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2025-05-04T21:07:29.915796272+09:00 [INFO][AUSF][5gAka] XresStar = 3638383432393835343537383962373738323962326565366532383838616434
2025-05-04T21:07:29.916054756+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |
2025-05-04T21:07:29.916677678+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2025-05-04T21:07:29.916936241+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Send Downlink Nas Transport
2025-05-04T21:07:29.917183922+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2025-05-04T21:07:29.918195994+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Handle UplinkNASTransport
2025-05-04T21:07:29.918355471+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2025-05-04T21:07:29.918398088+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2025-05-04T21:07:29.918404439+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2025-05-04T21:07:29.918411847+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2025-05-04T21:07:29.919223185+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.920732540+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.923391053+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.924855353+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2025-05-04T21:07:29.924900326+09:00 [INFO][AUSF][5gAka] res*: 3638383432393835343537383962373738323962326565366532383838616434
Xres*: 3638383432393835343537383962373738323962326565366532383838616434
2025-05-04T21:07:29.925024068+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2025-05-04T21:07:29.925725629+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.926701680+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.929577076+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.931240188+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2025-05-04T21:07:29.932024464+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.933364340+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.936096629+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.938586210+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-status |
2025-05-04T21:07:29.938890861+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |
2025-05-04T21:07:29.939375793+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |
2025-05-04T21:07:29.939788879+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2025-05-04T21:07:29.939955136+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2025-05-04T21:07:29.940239437+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Send Downlink Nas Transport
2025-05-04T21:07:29.940539697+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2025-05-04T21:07:29.941462289+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Handle UplinkNASTransport
2025-05-04T21:07:29.941478826+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2025-05-04T21:07:29.941533918+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2025-05-04T21:07:29.941542640+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2025-05-04T21:07:29.941548029+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2025-05-04T21:07:29.941576542+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2025-05-04T21:07:29.941583557+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2025-05-04T21:07:29.942435953+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.943822525+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.946012105+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.946891825+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:29.948021500+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2025-05-04T21:07:29.948685358+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.949817175+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.954369478+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.956372390+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 nssai
2025-05-04T21:07:29.956681883+09:00 [INFO][UDM][SDM] Handle GetNssai
2025-05-04T21:07:29.957496548+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.958587580+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.960961499+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.962405148+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2025-05-04T21:07:29.963006516+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features= |
2025-05-04T21:07:29.964344312+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2025-05-04T21:07:29.964903221+09:00 [INFO][AMF][Gmm] RequestedNssai: &{Iei:47 Len:5 Buffer:[4 1 1 2 3]}
2025-05-04T21:07:29.964959464+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2025-05-04T21:07:29.965625735+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.966974921+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.968850882+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.969548816+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:29.970497692+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2025-05-04T21:07:29.970974251+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.972021840+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.974592318+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.976173573+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2025-05-04T21:07:29.976330897+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2025-05-04T21:07:29.977028912+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.977966576+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.980347253+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.983946932+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |
2025-05-04T21:07:29.984423584+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |
2025-05-04T21:07:29.986979468+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.989018830+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.991839177+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.993202656+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 am-data
2025-05-04T21:07:29.993342462+09:00 [INFO][UDM][SDM] Handle GetAmData
2025-05-04T21:07:29.994104795+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:29.995306119+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:29.997924215+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:29.999255299+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2025-05-04T21:07:29.999806063+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2025-05-04T21:07:30.001339193+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2025-05-04T21:07:30.003825882+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.006591673+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.010325504+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.011770439+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 smf-select-data
2025-05-04T21:07:30.011908000+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2025-05-04T21:07:30.012742254+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.015393054+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.019100152+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.020943863+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data?supported-features= |
2025-05-04T21:07:30.021450801+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2025-05-04T21:07:30.022658361+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.024231418+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.027040701+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.028321313+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 ue-context-in-smf-data
2025-05-04T21:07:30.028622879+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2025-05-04T21:07:30.029289327+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.030468299+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.033102790+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.034770635+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/smf-registrations?supported-features= |
2025-05-04T21:07:30.035276086+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/ue-context-in-smf-data |
2025-05-04T21:07:30.036109761+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.037482757+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.040399236+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.042267631+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sdm-subscriptions
2025-05-04T21:07:30.043190074+09:00 [INFO][UDM][SDM] Handle Subscribe
2025-05-04T21:07:30.043896281+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.044739163+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.047579040+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.050597023+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2025-05-04T21:07:30.051059504+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v2/imsi-001010000000000/sdm-subscriptions |
2025-05-04T21:07:30.052306537+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.053813107+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.055790451+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.056576064+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.057699004+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |
2025-05-04T21:07:30.058268718+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.059641910+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.062679819+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.065005160+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2025-05-04T21:07:30.065857057+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.068471692+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.070539720+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.072837259+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.073631716+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |
2025-05-04T21:07:30.074623327+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.076015602+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.078636643+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.080450676+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/policy-data/ues/imsi-001010000000000/am-data |
2025-05-04T21:07:30.081532180+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.083171989+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.085265206+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.086106089+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.087344683+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |
2025-05-04T21:07:30.087922498+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.089174027+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.092144748+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.093589251+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2025-05-04T21:07:30.093665946+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2025-05-04T21:07:30.093767498+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |
2025-05-04T21:07:30.094370053+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |
2025-05-04T21:07:30.094955455+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2025-05-04T21:07:30.095203745+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Send Initial Context Setup Request
2025-05-04T21:07:30.095420778+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2025-05-04T21:07:30.095825413+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Handle InitialContextSetupResponse
2025-05-04T21:07:30.095995053+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Handle InitialContextSetupResponse (RAN UE NGAP ID: 1)
2025-05-04T21:07:30.296550820+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Handle UplinkNASTransport
2025-05-04T21:07:30.296582752+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2025-05-04T21:07:30.296645007+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2025-05-04T21:07:30.296654753+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2025-05-04T21:07:30.296660566+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2025-05-04T21:07:30.296679341+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Configuration Update Command
2025-05-04T21:07:30.296686810+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Send Downlink Nas Transport
2025-05-04T21:07:30.296742495+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2025-05-04T21:07:30.296900208+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Handle UplinkNASTransport
2025-05-04T21:07:30.296908428+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2025-05-04T21:07:30.296938429+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2025-05-04T21:07:30.296944334+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2025-05-04T21:07:30.296949700+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2025-05-04T21:07:30.296958747+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2025-05-04T21:07:30.297897548+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.299237972+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.301264127+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.302070835+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.302914888+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |
2025-05-04T21:07:30.303475932+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.304779298+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.307434920+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.308925135+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2025-05-04T21:07:30.309354953+09:00 [WARN][NSSF][Util] No TA {"plmnId":{"mcc":"001","mnc":"01"},"tac":"000001"} in NSSF configuration
2025-05-04T21:07:30.309596922+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v2/network-slice-information?nf-id=1fbf84cc-5cd9-4a5a-9588-1da312f05839&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D&tai=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22tac%22%3A%22000001%22%7D |
2025-05-04T21:07:30.310035854+09:00 [WARN][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] nsiInformation is still nil, use default NRF[http://127.0.0.10:8000]
2025-05-04T21:07:30.310854571+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.312081817+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.314279684+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.315113936+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.316435971+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%5B%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%5D&target-nf-type=SMF&target-plmn-list=%5B%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%5D |
2025-05-04T21:07:30.317115789+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.318709977+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.323863003+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.326978429+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2025-05-04T21:07:30.328883927+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2025-05-04T21:07:30.329142902+09:00 [INFO][SMF][CTX] UrrPeriod: 30s
2025-05-04T21:07:30.330112755+09:00 [INFO][SMF][CTX] UrrThreshold: 500000
2025-05-04T21:07:30.331408514+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.333486155+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.335701537+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.336543217+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.337545117+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |
2025-05-04T21:07:30.338475596+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2025-05-04T21:07:30.338846051+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.340013915+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.343060489+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.344653006+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sm-data
2025-05-04T21:07:30.344852777+09:00 [INFO][UDM][SDM] Handle GetSmData
2025-05-04T21:07:30.345591343+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.346785418+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.349397899+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.349801645+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2025-05-04T21:07:30.351465326+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2025-05-04T21:07:30.351925923+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2025-05-04T21:07:30.353040475+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2025-05-04T21:07:30+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc000255600 0xc000255620]
2025-05-04T21:07:30.353337694+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2025-05-04T21:07:30.353461135+09:00 [INFO][SMF][GSM] &{[0xc000255600 0xc000255620]}
2025-05-04T21:07:30.353498226+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2025-05-04T21:07:30.354376599+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.355583643+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.357733157+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.358436380+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.359516193+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=1fbf84cc-5cd9-4a5a-9588-1da312f05839&target-nf-type=AMF |
2025-05-04T21:07:30.360026076+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2025-05-04T21:07:30.360292771+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2025-05-04T21:07:30.360485255+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2025-05-04T21:07:30.360503734+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Allocated PDUAdress[10.60.0.1]
2025-05-04T21:07:30.360826994+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.361857274+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.364035924+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.364783319+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.365691747+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |
2025-05-04T21:07:30.366501176+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.367679750+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.370618729+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.372040489+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2025-05-04T21:07:30.373115226+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.374305745+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.377003211+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.379405271+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2025-05-04T21:07:30.383125009+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.385395307+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.389228584+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.392330350+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/application-data/influenceData?dnns=internet&snssais=%5B%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%5D&supis=imsi-001010000000000 |
2025-05-04T21:07:30.392715163+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2025-05-04T21:07:30.393522182+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.394883394+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.397534025+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.399114014+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/application-data/influenceData/subs-to-notify |
2025-05-04T21:07:30.400078531+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.401710796+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.403789367+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.404496194+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.404962162+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |
2025-05-04T21:07:30.406105572+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |
2025-05-04T21:07:30.407602279+09:00 [INFO][SMF][PduSess] CHF Selection for SMContext SUPI[imsi-001010000000000] PDUSessionID[1]
2025-05-04T21:07:30.408576361+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.409707292+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.411767398+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.412539078+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T21:07:30.413415550+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=CHF |
2025-05-04T21:07:30.413792233+09:00 [INFO][SMF][Charging] Handle SendConvergedChargingRequest
2025-05-04T21:07:30.414024094+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.415269410+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.417975609+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.423391277+09:00 [INFO][CHF][ChargingPost] HandleChargingdataInitial
2025-05-04T21:07:30.423650093+09:00 [INFO][CHF][ChargingPost] SMF charging event
2025-05-04T21:07:30.423898718+09:00 [ERRO][CHF][ChargingPost] Charging gateway fail to send CDR to billing domain dial tcp 127.0.0.1:2121: connect: connection refused
2025-05-04T21:07:30.424026819+09:00 [INFO][CHF][ChargingPost] Open CDR for UE imsi-001010000000000
2025-05-04T21:07:30.424245737+09:00 [INFO][CHF][ChargingPost] NewChfUe imsi-001010000000000
2025-05-04T21:07:30.424826003+09:00 [INFO][CHF][GIN] | 201 |       127.0.0.1 | POST    | /nchf-convergedcharging/v3/chargingdata |
2025-05-04T21:07:30.425524102+09:00 [INFO][SMF][Charging] Send Charging Data Request[Init] successfully
2025-05-04T21:07:30.425789643+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-1]
2025-05-04T21:07:30.425980616+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2025-05-04T21:07:30.426157121+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has default path
2025-05-04T21:07:30.427752490+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2025-05-04T21:07:30.429520011+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sdm-subscriptions
2025-05-04T21:07:30.430004350+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2025-05-04T21:07:30.430257346+09:00 [INFO][UDM][SDM] Handle Subscribe
2025-05-04T21:07:30.431506885+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.435168359+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.438683166+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.438991287+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.440707718+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.441812728+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2025-05-04T21:07:30.443535701+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v2/imsi-001010000000000/sdm-subscriptions |
2025-05-04T21:07:30.443875422+09:00 [INFO][SMF][PduSess] SDM Subscription Successful UE: imsi-001010000000000 SubscriptionId: 2
2025-05-04T21:07:30.444062935+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |
2025-05-04T21:07:30.445034209+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2025-05-04T21:07:30.449827857+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.451763717+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2025-05-04T21:07:30.451947865+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Send PDU Session Resource Setup Request
2025-05-04T21:07:30.452262672+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |
2025-05-04T21:07:30.453993509+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Handle PDUSessionResourceSetupResponse
2025-05-04T21:07:30.454189775+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:56315] Not comprehended IE ID 0x0079 (criticality: ignore)
2025-05-04T21:07:30.454213118+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:56315] Handle PDUSessionResourceSetupResponse (RAN UE NGAP ID: 1)
2025-05-04T21:07:30.454997624+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T21:07:30.456535888+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T21:07:30.459407353+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T21:07:30.461188349+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2025-05-04T21:07:30.462625918+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2025-05-04T21:07:30.462920925+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:cd832468-60cf-4ed8-92c4-d3e29f8b8c27/modify |
```
The PDU session establishment log of eUPF is as follows.
```
2025/05/04 21:07:30 INF Got Session Establishment Request from: 192.168.14.141.
2025/05/04 21:07:30 INF 
Session Establishment Request:
  CreatePDR ID: 1 
    Outer Header Removal: 0 
    FAR ID: 1 
    QER ID: 2 
    QER ID: 1 
    URR ID: 1 
    Source Interface: 0 
    TEID: 2 
    Ipv4: 192.168.13.151 
    Ipv6: <nil> 
    UE IPv4 Address: 10.60.0.1 
    SDF Filter: permit out ip from any to assigned 
  CreatePDR ID: 2 
    FAR ID: 2 
    QER ID: 2 
    QER ID: 1 
    URR ID: 1 
    Source Interface: 1 
    UE IPv4 Address: 10.60.0.1 
    SDF Filter: permit out ip from any to assigned 
  CreateFAR ID: 1 
    Apply Action: [2] 
    Forwarding Parameters:
      Network Instance: internet 
  CreateFAR ID: 2 
    Apply Action: [2] 
    Forwarding Parameters:
  CreateQER ID: 2 
    Gate Status DL: 0 
    Gate Status UL: 0 
    QFI: 1 
  CreateQER ID: 1 
    Gate Status DL: 0 
    Gate Status UL: 0 
    Max Bitrate DL: 1000000 
    Max Bitrate UL: 1000000 
    QFI: 1 
  CreateURR ID: 2 
    Measurement Method: 2 
    Volume Threshold: &{Flags:6 TotalVolume:0 UplinkVolume:500000 DownlinkVolume:500000} 
  CreateURR ID: 7 
    Measurement Method: 2 
    Volume Threshold: &{Flags:6 TotalVolume:0 UplinkVolume:500000 DownlinkVolume:500000} 
  CreateURR ID: 1 
    Measurement Method: 2 
    Volume Threshold: &{Flags:6 TotalVolume:0 UplinkVolume:500000 DownlinkVolume:500000} 

2025/05/04 21:07:30 WRN No OuterHeaderCreation
2025/05/04 21:07:30 INF Saving FAR info to session: 1, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:2534254784 TransportLevelMarking:0}
2025/05/04 21:07:30 WRN No OuterHeaderCreation
2025/05/04 21:07:30 INF Saving FAR info to session: 2, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:2534254784 TransportLevelMarking:0}
2025/05/04 21:07:30 INF Saving QER info to session: 2, {GateStatusUL:0 GateStatusDL:0 Qfi:1 MaxBitrateUL:0 MaxBitrateDL:0 StartUL:0 StartDL:0}
2025/05/04 21:07:30 INF Saving QER info to session: 1, {GateStatusUL:0 GateStatusDL:0 Qfi:1 MaxBitrateUL:1000000000 MaxBitrateDL:1000000000 StartUL:0 StartDL:0}
2025/05/04 21:07:30 INF Saving URR info to session: 2, {UplinkVolume:0 DownlinkVolume:0}
2025/05/04 21:07:30 INF Saving URR info to session: 7, {UplinkVolume:0 DownlinkVolume:0}
2025/05/04 21:07:30 INF Saving URR info to session: 1, {UplinkVolume:0 DownlinkVolume:0}
2025/05/04 21:07:30 Matched groups: ["permit out ip from any to assigned" "ip" "any" "" "" "assigned" "" ""]
2025/05/04 21:07:30 Matched groups: ["permit out ip from any to assigned" "ip" "any" "" "" "assigned" "" ""]
2025/05/04 21:07:30 INF Session Establishment Request from 192.168.14.141 accepted.
2025/05/04 21:07:30 INF Got Session Modification Request from: 192.168.14.141. 

2025/05/04 21:07:30 INF Finding association for 192.168.14.141
2025/05/04 21:07:30 INF Finding session 2
2025/05/04 21:07:30 INF 
Session Modification Request:
  UpdatePDR ID: 2 
    FAR ID: 2 
    URR ID: 1 
    Source Interface: 1 
    UE IPv4 Address: 10.60.0.1 
    SDF Filter: permit out ip from any to assigned 
  UpdateFAR ID: 2 
    Apply Action: [2] 
    Update forwarding Parameters:
      Network Instance: internet 
      Outer Header Creation: &{OuterHeaderCreationDescription:256 TEID:1 IPv4Address:192.168.13.131 IPv6Address:<nil> PortNumber:0 CTag:0 STag:0} 

2025/05/04 21:07:30 INF Updating FAR info: 2, {FarInfo:{Action:2 OuterHeaderCreation:1 Teid:1 RemoteIP:2198710464 LocalIP:2534254784 TransportLevelMarking:0} GlobalId:1}
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.60.0.1` from free5GC 5GC.
```
[2025-05-04 21:07:30.445] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
10: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/24 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::fb12:2e91:afc2:ffe7/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
...
```

<a id="ping"></a>

## Ping google.com

Specify the UE's TUNnel interface and try ping.

Please refer to the following for usage of TUNnel interface.

https://github.com/aligungr/UERANSIM/wiki/Usage


<a id="ping_1"></a>

### Case for going through DN 10.60.0.0/16

Run `tcpdump` on VM-DN and check that the packet goes through N6 (ens20).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (172.217.31.142) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 172.217.31.142: icmp_seq=1 ttl=111 time=19.3 ms
64 bytes from 172.217.31.142: icmp_seq=2 ttl=111 time=18.0 ms
64 bytes from 172.217.31.142: icmp_seq=3 ttl=111 time=17.9 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i ens20 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:12:55.825797 IP 10.60.0.1 > 172.217.31.142: ICMP echo request, id 1481, seq 1, length 64
21:12:55.844089 IP 172.217.31.142 > 10.60.0.1: ICMP echo reply, id 1481, seq 1, length 64
21:12:56.827327 IP 10.60.0.1 > 172.217.31.142: ICMP echo request, id 1481, seq 2, length 64
21:12:56.844309 IP 172.217.31.142 > 10.60.0.1: ICMP echo reply, id 1481, seq 2, length 64
21:12:57.828373 IP 10.60.0.1 > 172.217.31.142: ICMP echo request, id 1481, seq 3, length 64
21:12:57.845341 IP 172.217.31.142 > 10.60.0.1: ICMP echo reply, id 1481, seq 3, length 64
```
If you have built eUPF to output the kernel log for debugging as described [here](https://github.com/s5uishida/install_eupf#generate_codes), you can see the kernel log as follows.
```
# cat /sys/kernel/debug/tracing/trace_pipe
```
You could specify the IP address assigned to the TUNnel interface to run almost any applications (iperf3 etc.) as in the following example using `nr-binder` tool.

- `curl google.com` on VM3 (UE)
```
# sh nr-binder 10.60.0.1 curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- Run `tcpdump` on VM-DN
```
21:15:28.239828 IP 10.60.0.1.43781 > 172.217.31.142.80: Flags [S], seq 467769662, win 65280, options [mss 1360,sackOK,TS val 3747309310 ecr 0,nop,wscale 7], length 0
21:15:28.255199 IP 172.217.31.142.80 > 10.60.0.1.43781: Flags [S.], seq 1099740655, ack 467769663, win 65535, options [mss 1412,sackOK,TS val 636095263 ecr 3747309310,nop,wscale 8], length 0
21:15:28.256230 IP 10.60.0.1.43781 > 172.217.31.142.80: Flags [.], ack 1, win 510, options [nop,nop,TS val 3747309326 ecr 636095263], length 0
21:15:28.256230 IP 10.60.0.1.43781 > 172.217.31.142.80: Flags [P.], seq 1:74, ack 1, win 510, options [nop,nop,TS val 3747309326 ecr 636095263], length 73: HTTP: GET / HTTP/1.1
21:15:28.273374 IP 172.217.31.142.80 > 10.60.0.1.43781: Flags [.], ack 74, win 1050, options [nop,nop,TS val 636095281 ecr 3747309326], length 0
21:15:28.370186 IP 172.217.31.142.80 > 10.60.0.1.43781: Flags [P.], seq 1:774, ack 74, win 1050, options [nop,nop,TS val 636095378 ecr 3747309326], length 773: HTTP: HTTP/1.1 301 Moved Permanently
21:15:28.371074 IP 10.60.0.1.43781 > 172.217.31.142.80: Flags [.], ack 774, win 504, options [nop,nop,TS val 3747309441 ecr 636095378], length 0
21:15:28.371892 IP 10.60.0.1.43781 > 172.217.31.142.80: Flags [F.], seq 74, ack 774, win 504, options [nop,nop,TS val 3747309441 ecr 636095378], length 0
21:15:28.387147 IP 172.217.31.142.80 > 10.60.0.1.43781: Flags [F.], seq 774, ack 75, win 1050, options [nop,nop,TS val 636095395 ecr 3747309441], length 0
21:15:28.388001 IP 10.60.0.1.43781 > 172.217.31.142.80: Flags [.], ack 775, win 504, options [nop,nop,TS val 3747309458 ecr 636095395], length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using eUPF.

---

Now you could work free5GC with eUPF.
I would like to thank the excellent developers and all the contributors of free5GC, eUPF and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2025.05.04] Updated to eUPF `v0.7.1 (2025.04.16)`, free5GC `v4.0.1 (2025.04.27)` and UERANSIM `v3.2.7 (2025.04.28)`. Changed the VM environment from Virtualbox to Proxmox VE.
- [2024.03.24] Updated to eUPF `v0.6.1`.
- [2023.12.05] The eUPF version confirmed to work in the changelog on 2023.12.04 has been tagged as `v0.6.0`.
- [2023.12.04] Updated as eUPF FTUP feature has been merged into `main` branch.
- [2023.11.26] Updated to eUPF `120-upf-ftup-fteid` branch that supports FTUP. Although free5GC does not support FTUP in PDU Session Establishment, I confirmed its operation on `120-upf-ftup-fteid` branch in order to compare it with the logs of Open5GS which supports FTUP.
- [2023.10.29] Initial release.
