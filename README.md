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
**Note that this configuration is implemented with Virtualbox VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / eBPF/XDP UPF / UE / RAN used are as follows.
- 5GC - free5GC v3.3.0 (2023.11.25) - https://github.com/free5gc/free5gc
- eBPF/XDP UPF - eUPF `120-upf-ftup-fteid` branch (2023.11.25) - https://github.com/edgecomllc/eupf
- UE / RAN - UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | free5GC 5GC C-Plane | 192.168.0.141/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM-UP | eUPF U-Plane | 192.168.0.151/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
| VM | Device | Network Adapter | IP address | Interface | XDP |
| --- | --- | --- | --- | --- | --- |
| VM1 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.141/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.14.141/24 | N4 | -- |
| VM-UP | ~~enp0s3~~ | ~~NAT(default)~~ | ~~10.0.2.15/24~~ | ~~(VM default NW)~~ ***down*** | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.151/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.151/24 | N3 | x |
| | enp0s10 | NAT Network | 192.168.14.151/24 | N4 | -- |
| | enp0s16 | NAT Network | 192.168.16.151/24 | N6 | x |
| VM-DN | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.152/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.16.152/24 | N6, ***default GW for VM-UP*** | -- |
| VM2 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.131/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.131/24 | N3 | -- |
| VM3 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.132/24 | (Mgmt NW) | -- |

NAT networks of Virtualbox  are as follows.
| Network Name | Network CIDR |
| --- | --- |
| N3 | 192.168.13.0/24 |
| N4 | 192.168.14.0/24 |
| N6 | 192.168.16.0/24 |


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
- free5GC v3.3.0 (2023.11.25) - https://free5gc.org/guide/
- eUPF `120-upf-ftup-fteid` branch (2023.11.25) - https://github.com/s5uishida/install_eupf
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of free5GC 5GC C-Plane

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2023-11-10 23:13:08.456735624 +0900
+++ amfcfg.yaml 2023-11-10 23:26:10.862955424 +0900
@@ -5,7 +5,7 @@
 configuration:
   amfName: AMF # the name of this AMF
   ngapIpList:  # the IP list of N2 interfaces on this AMF
-    - 127.0.0.18
+    - 192.168.0.141
   ngapPort: 38412 # the SCTP port listened by NGAP
   sbi: # Service-based interface information
     scheme: http # the protocol for sbi (http or https)
@@ -24,18 +24,18 @@
   servedGuamiList: # Guami (Globally Unique AMF ID) list supported by this AMF
     # <GUAMI> = <MCC><MNC><AMF ID>
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       amfId: cafe00 # AMF identifier (3 bytes hex string, range: 000000~FFFFFF)
   supportTaiList:  # the TAI (Tracking Area Identifier) list supported by this AMF
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       tac: 000001 # Tracking Area Code (3 bytes hex string, range: 000000~FFFFFF)
   plmnSupportList: # the PLMNs (Public land mobile network) list supported by this AMF
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
--- ausfcfg.yaml.orig   2023-10-28 22:09:20.479542560 +0900
+++ ausfcfg.yaml        2023-10-28 22:16:35.947533214 +0900
@@ -15,8 +15,8 @@
     - nausf-auth # Nausf_UEAuthentication service
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   plmnSupportList: # the PLMNs (Public Land Mobile Network) list supported by this AUSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
     - mcc: 123 # Mobile Country Code (3 digits string, digit: 0~9)
       mnc: 45  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   groupId: ausfGroup001 # ID for the group of the AUSF
```
- `free5gc/config/nrfcfg.yaml`
```diff
--- nrfcfg.yaml.orig    2023-10-28 22:09:20.480542536 +0900
+++ nrfcfg.yaml 2023-10-28 22:16:42.051592773 +0900
@@ -14,8 +14,8 @@
       pem: cert/nrf.pem # NRF TLS Certificate
       key: cert/nrf.key # NRF TLS Private key
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
--- nssfcfg.yaml.orig   2023-10-28 22:09:20.480542536 +0900
+++ nssfcfg.yaml        2023-10-28 22:16:46.708638917 +0900
@@ -17,12 +17,12 @@
     - nnssf-nssaiavailability # Nnssf_NSSAIAvailability service
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
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
--- smfcfg.yaml.orig    2023-10-28 22:09:20.480542536 +0900
+++ smfcfg.yaml 2023-10-28 22:16:51.996692030 +0900
@@ -34,22 +34,22 @@
             ipv4: 8.8.8.8
             ipv6: 2001:4860:4860::8888
   plmnList: # the list of PLMN IDs that this SMF belongs to (optional, remove this key when unnecessary)
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   locality: area1 # Name of the location where a set of AMF, SMF, PCF and UPFs are located
   pfcp: # the IP address of N4 interface on this SMF (PFCP)
     # addr config is deprecated in smf config v1.0.3, please use the following config
-    nodeID: 127.0.0.1 # the Node ID of this SMF
-    listenAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
-    externalAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    nodeID: 192.168.14.141 # the Node ID of this SMF
+    listenAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    externalAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
   userplaneInformation: # list of userplane information
     upNodes: # information of userplane node (AN or UPF)
       gNB1: # the name of the node
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
@@ -72,7 +72,7 @@
         interfaces: # Interface list for this UPF
           - interfaceType: N3 # the type of the interface (N3 or N9)
             endpoints: # the IP address of this N3/N9 interface on this UPF
-              - 127.0.0.8
+              - 192.168.13.151
             networkInstances:  # Data Network Name (DNN)
               - internet
     links: # the topology graph of userplane, A and B represent the two nodes of each link
@@ -89,8 +89,10 @@
     expireTime: 16s   # default is 6 seconds
     maxRetryTimes: 3 # the max number of retransmission
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
-  #urrPeriod: 10 # default usage report period in seconds
-  #urrThreshold: 1000 # default usage report threshold in bytes
+  urrPeriod: 10 # default usage report period in seconds
+  urrThreshold: 1000 # default usage report threshold in bytes
+  ulcl: false
+  nwInstFqdnEncoding: true
 
 logger: # log output setting
   enable: true # true or false
```


<a id="changes_up"></a>

### Changes in configuration files of eUPF

See [here](https://github.com/s5uishida/install_eupf#create-configuration-file) for the original file.

- `eupf/config.yml`  
There is no change.

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `UERANSIM/config/free5gc-gnb.yaml`
```diff
--- free5gc-gnb.yaml.orig       2021-02-12 09:47:56.000000000 +0900
+++ free5gc-gnb.yaml    2023-06-15 22:24:00.297158446 +0900
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
--- free5gc-ue.yaml.orig        2023-05-10 14:51:54.000000000 +0900
+++ free5gc-ue.yaml     2023-06-15 22:24:48.016816988 +0900
@@ -1,9 +1,9 @@
 # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
-supi: 'imsi-208930000000003'
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
@@ -28,7 +28,7 @@
 
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
- free5GC v3.3.0 (2023.11.25) - https://free5gc.org/guide/
- eUPF `120-upf-ftup-fteid` branch (2023.11.25) - https://github.com/s5uishida/install_eupf
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

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

NF_LIST="nrf amf smf udr pcf udm nssf ausf chf"

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
2023/11/26 08:43:21 INF Got Association Setup Request from: 192.168.14.141. 

2023/11/26 08:43:21 INF 
Association Setup Request:
  Node ID: 192.168.14.141
  Recovery Time: 2023-11-26 08:43:21 +0900 JST

2023/11/26 08:43:21 INF Saving new association: &{ID:192.168.14.141 Addr:192.168.14.141 NextSessionID:1 NextSequenceID:1 Sessions:map[] HeartbeatRetries:0 cancelRetries:<nil>}
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
UERANSIM v3.2.6
[2023-11-26 08:44:22.460] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2023-11-26 08:44:22.462] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2023-11-26 08:44:22.463] [sctp] [debug] SCTP association setup ascId[7]
[2023-11-26 08:44:22.463] [ngap] [debug] Sending NG Setup Request
[2023-11-26 08:44:22.465] [ngap] [debug] NG Setup Response received
[2023-11-26 08:44:22.465] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2023-11-26T08:44:22.450438250+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:55891
2023-11-26T08:44:22.451270013+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:55891
2023-11-26T08:44:22.451939243+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Handle NGSetupRequest
2023-11-26T08:44:22.451976539+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Send NG-Setup response
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.6
[2023-11-26 08:45:10.629] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2023-11-26 08:45:10.630] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2023-11-26 08:45:10.631] [nas] [info] Selected plmn[001/01]
[2023-11-26 08:45:10.631] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2023-11-26 08:45:10.631] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2023-11-26 08:45:10.631] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2023-11-26 08:45:10.632] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2023-11-26 08:45:10.635] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-11-26 08:45:10.635] [nas] [debug] Sending Initial Registration
[2023-11-26 08:45:10.636] [rrc] [debug] Sending RRC Setup Request
[2023-11-26 08:45:10.636] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2023-11-26 08:45:10.636] [rrc] [info] RRC connection established
[2023-11-26 08:45:10.636] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2023-11-26 08:45:10.636] [nas] [info] UE switches to state [CM-CONNECTED]
[2023-11-26 08:45:10.658] [nas] [debug] Authentication Request received
[2023-11-26 08:45:10.665] [nas] [debug] Security Mode Command received
[2023-11-26 08:45:10.665] [nas] [debug] Selected integrity[2] ciphering[0]
[2023-11-26 08:45:10.705] [nas] [debug] Registration accept received
[2023-11-26 08:45:10.706] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2023-11-26 08:45:10.706] [nas] [debug] Sending Registration Complete
[2023-11-26 08:45:10.706] [nas] [info] Initial Registration is successful
[2023-11-26 08:45:10.706] [nas] [debug] Sending PDU Session Establishment Request
[2023-11-26 08:45:10.706] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-11-26 08:45:10.914] [nas] [debug] Configuration Update Command received
[2023-11-26 08:45:11.023] [nas] [debug] PDU Session Establishment Accept received
[2023-11-26 08:45:11.029] [nas] [info] PDU Session establishment is successful PSI[1]
[2023-11-26 08:45:11.054] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2023-11-26T08:45:10.632218127+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Handle InitialUEMessage
2023-11-26T08:45:10.632269531+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2023-11-26T08:45:10.632306619+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2023-11-26T08:45:10.632787649+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2023-11-26T08:45:10.633068299+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2023-11-26T08:45:10.633247788+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2023-11-26T08:45:10.633415930+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2023-11-26T08:45:10.633557713+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2023-11-26T08:45:10.633734961+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2023-11-26T08:45:10.633887351+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2023-11-26T08:45:10.634574869+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.635731385+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |
2023-11-26T08:45:10.637048493+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2023-11-26T08:45:10.637232183+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2023-11-26T08:45:10.637923007+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.639174634+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |
2023-11-26T08:45:10.640205882+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2023-11-26T08:45:10.640361618+09:00 [INFO][UDM][Suci] suciPart: [suci 0 001 01 0000 0 0 0000000000]
2023-11-26T08:45:10.640508134+09:00 [INFO][UDM][Suci] scheme 0
2023-11-26T08:45:10.640590138+09:00 [INFO][UDM][Suci] SUPI type is IMSI
http://127.0.0.10:8000
2023-11-26T08:45:10.641592490+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.643166426+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |
2023-11-26T08:45:10.644769238+09:00 [INFO][UDR][DataRepo] Handle QueryAuthSubsData
2023-11-26T08:45:10.647102454+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2023-11-26T08:45:10.647981953+09:00 [INFO][UDM][UEAU] Nil Op
2023-11-26T08:45:10.648520036+09:00 [INFO][UDR][DataRepo] Handle ModifyAuthentication
2023-11-26T08:45:10.650343173+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2023-11-26T08:45:10.650689244+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |
2023-11-26T08:45:10.651187842+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2023-11-26T08:45:10.651239408+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2023-11-26T08:45:10.651271633+09:00 [INFO][AUSF][5gAka] XresStar = 3830646232613139356165383761326262656262636132626562633538363236
2023-11-26T08:45:10.651375789+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |
2023-11-26T08:45:10.651609675+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2023-11-26T08:45:10.651685828+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Send Downlink Nas Transport
2023-11-26T08:45:10.652174590+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2023-11-26T08:45:10.653532645+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Handle UplinkNASTransport
2023-11-26T08:45:10.653694193+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-11-26T08:45:10.653892965+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2023-11-26T08:45:10.654044980+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2023-11-26T08:45:10.654208265+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2023-11-26T08:45:10.655059054+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2023-11-26T08:45:10.655114326+09:00 [INFO][AUSF][5gAka] res*: 3830646232613139356165383761326262656262636132626562633538363236
Xres*: 3830646232613139356165383761326262656262636132626562633538363236
2023-11-26T08:45:10.655176280+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2023-11-26T08:45:10.656117093+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2023-11-26T08:45:10.656789924+09:00 [INFO][UDR][DataRepo] Handle CreateAuthenticationStatus
2023-11-26T08:45:10.658052465+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-status |
2023-11-26T08:45:10.658368466+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |
2023-11-26T08:45:10.658683577+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |
2023-11-26T08:45:10.658943460+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2023-11-26T08:45:10.659048634+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2023-11-26T08:45:10.659103073+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Send Downlink Nas Transport
2023-11-26T08:45:10.659517739+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2023-11-26T08:45:10.660920903+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Handle UplinkNASTransport
2023-11-26T08:45:10.661160396+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-11-26T08:45:10.661350288+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2023-11-26T08:45:10.661493528+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2023-11-26T08:45:10.661690725+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2023-11-26T08:45:10.661876769+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2023-11-26T08:45:10.662031100+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2023-11-26T08:45:10.662723698+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.664042185+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2023-11-26T08:45:10.665064856+09:00 [INFO][UDM][SDM] Handle GetNssai
2023-11-26T08:45:10.665989234+09:00 [INFO][UDR][DataRepo] Handle QueryAmData
2023-11-26T08:45:10.666638696+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data |
2023-11-26T08:45:10.667040208+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-26T08:45:10.667375812+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2023-11-26T08:45:10.668927670+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.670141919+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2023-11-26T08:45:10.672313324+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2023-11-26T08:45:10.672489890+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2023-11-26T08:45:10.673537294+09:00 [INFO][UDR][DataRepo] Handle CreateAmfContext3gpp
2023-11-26T08:45:10.674752582+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |
2023-11-26T08:45:10.675053953+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |
2023-11-26T08:45:10.675969818+09:00 [INFO][UDM][SDM] Handle GetAmData
2023-11-26T08:45:10.676376090+09:00 [INFO][UDR][DataRepo] Handle QueryAmData
2023-11-26T08:45:10.677025577+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-26T08:45:10.677532772+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-26T08:45:10.678227811+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2023-11-26T08:45:10.678940826+09:00 [INFO][UDR][DataRepo] Handle QuerySmfSelectData
2023-11-26T08:45:10.679455106+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data |
2023-11-26T08:45:10.679818073+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-26T08:45:10.680439979+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2023-11-26T08:45:10.680738811+09:00 [INFO][UDR][DataRepo] Handle QuerySmfRegList
2023-11-26T08:45:10.681300602+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/smf-registrations |
2023-11-26T08:45:10.681655154+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/ue-context-in-smf-data |
2023-11-26T08:45:10.682304965+09:00 [INFO][UDM][SDM] Handle Subscribe
2023-11-26T08:45:10.683217943+09:00 [INFO][UDR][DataRepo] Handle CreateSdmSubscriptions
2023-11-26T08:45:10.683430382+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2023-11-26T08:45:10.683800747+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v1/imsi-001010000000000/sdm-subscriptions |
2023-11-26T08:45:10.684668609+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.686665029+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |
2023-11-26T08:45:10.688569548+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2023-11-26T08:45:10.689254331+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.690405959+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |
2023-11-26T08:45:10.691245475+09:00 [INFO][UDR][DataRepo] Handle PolicyDataUesUeIdAmDataGet
2023-11-26T08:45:10.692075051+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/am-data |
2023-11-26T08:45:10.693028789+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.694489670+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |
2023-11-26T08:45:10.695483853+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2023-11-26T08:45:10.695689841+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2023-11-26T08:45:10.695937933+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |
2023-11-26T08:45:10.696917064+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |
2023-11-26T08:45:10.697542990+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2023-11-26T08:45:10.697854132+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Send Initial Context Setup Request
2023-11-26T08:45:10.699402583+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2023-11-26T08:45:10.700345961+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Handle InitialContextSetupResponse
2023-11-26T08:45:10.700543608+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Handle InitialContextSetupResponse (RAN UE NGAP ID: 1)
2023-11-26T08:45:10.903937625+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Handle UplinkNASTransport
2023-11-26T08:45:10.904576733+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-11-26T08:45:10.905005231+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2023-11-26T08:45:10.905507108+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2023-11-26T08:45:10.905800355+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2023-11-26T08:45:10.906331449+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Configuration Update Command
2023-11-26T08:45:10.906697467+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Send Downlink Nas Transport
2023-11-26T08:45:10.908057115+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2023-11-26T08:45:10.909987371+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Handle UplinkNASTransport
2023-11-26T08:45:10.910301353+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-11-26T08:45:10.910607379+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2023-11-26T08:45:10.913474778+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2023-11-26T08:45:10.913729458+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2023-11-26T08:45:10.914081134+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2023-11-26T08:45:10.915365691+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.917159377+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |
2023-11-26T08:45:10.918351419+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2023-11-26T08:45:10.918795690+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v1/network-slice-information?nf-id=830cc6c2-7c0c-4535-b72d-62b6cbc31825&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D |
2023-11-26T08:45:10.919916424+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.921253179+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&target-nf-type=SMF&target-plmn-list=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-26T08:45:10.922448940+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2023-11-26T08:45:10.923289545+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2023-11-26T08:45:10.923554230+09:00 [INFO][SMF][CTX] UrrPeriod: 10s
2023-11-26T08:45:10.923730400+09:00 [INFO][SMF][CTX] UrrThreshold: 1000
2023-11-26T08:45:10.924416905+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.926135204+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |
2023-11-26T08:45:10.927283646+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2023-11-26T08:45:10.929333299+09:00 [INFO][UDM][SDM] Handle GetSmData
2023-11-26T08:45:10.929531142+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2023-11-26T08:45:10.930338640+09:00 [INFO][UDR][DataRepo] Handle QuerySmData
2023-11-26T08:45:10.931370766+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-11-26T08:45:10.931887268+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-11-26T08:45:10.932600355+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2023-11-26T08:45:10+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc000308d00 0xc000308d20]
2023-11-26T08:45:10.933057212+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2023-11-26T08:45:10.933298697+09:00 [INFO][SMF][GSM] &{[0xc000308d00 0xc000308d20]}
2023-11-26T08:45:10.933450572+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2023-11-26T08:45:10.934401508+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.935748922+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=830cc6c2-7c0c-4535-b72d-62b6cbc31825&target-nf-type=AMF |
2023-11-26T08:45:10.936345916+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2023-11-26T08:45:10.936626479+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2023-11-26T08:45:10.936814930+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2023-11-26T08:45:10.937028382+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Allocated PDUAdress[10.60.0.1]
2023-11-26T08:45:10.937402363+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.938599367+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |
2023-11-26T08:45:10.939988715+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2023-11-26T08:45:10.940939121+09:00 [INFO][UDR][DataRepo] Handle PolicyDataUesUeIdSmDataGet
2023-11-26T08:45:10.942166673+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-11-26T08:45:10.945220026+09:00 [INFO][UDR][DataRepo] Handle ApplicationDataInfluenceDataGet
2023-11-26T08:45:10.945733646+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/application-data/influenceData?dnns=internet&internal-Group-Ids=&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&supis=imsi-001010000000000 |
2023-11-26T08:45:10.946717497+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2023-11-26T08:45:10.947911964+09:00 [INFO][UDR][DataRepo] Handle ApplicationDataInfluenceDataSubsToNotifyPost
2023-11-26T08:45:10.948105383+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/application-data/influenceData/subs-to-notify |
2023-11-26T08:45:10.950196554+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-26T08:45:10.951015789+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |
2023-11-26T08:45:10.951788104+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |
2023-11-26T08:45:10.953193517+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has no pre-config route
2023-11-26T08:45:10.955329242+09:00 [WARN][SMF][PduSess] Create URR
2023-11-26T08:45:10.955530007+09:00 [WARN][SMF][PduSess] Create URR
2023-11-26T08:45:10.955695410+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-1]
2023-11-26T08:45:10.956136479+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2023-11-26T08:45:10.956428444+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |
2023-11-26T08:45:10.956821495+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2023-11-26T08:45:10.957598717+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2023-11-26T08:45:10.972950034+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2023-11-26T08:45:10.975306927+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2023-11-26T08:45:10.975525581+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Send PDU Session Resource Setup Request
2023-11-26T08:45:10.976374496+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |
2023-11-26T08:45:11.019146568+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:55891] Handle PDUSessionResourceSetupResponse
2023-11-26T08:45:11.019712335+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:55891] Handle PDUSessionResourceSetupResponse (RAN UE NGAP ID: 1)
2023-11-26T08:45:11.022654184+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2023-11-26T08:45:11.032001079+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2023-11-26T08:45:11.032538766+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:a227e17a-8e4f-4dc0-9398-239cfa399638/modify |
```
The PDU session establishment log of eUPF is as follows.
```
2023/11/26 08:45:10 INF Got Session Establishment Request from: 192.168.14.141.
2023/11/26 08:45:10 INF 
Session Establishment Request:
  CreatePDR ID: 1 
    Outer Header Removal: 0 
    FAR ID: 1 
    QER ID: 2 
    QER ID: 1 
    URR ID: 1 
    Source Interface: 0 
    TEID: 1 
    Ipv4: 192.168.13.151 
    Ipv6: <nil> 
    UE IPv4 Address: 10.60.0.1 
  CreatePDR ID: 2 
    FAR ID: 2 
    QER ID: 2 
    QER ID: 1 
    URR ID: 1 
    Source Interface: 2 
    UE IPv4 Address: 10.60.0.1 
  CreatePDR ID: 3 
    Outer Header Removal: 0 
    FAR ID: 3 
    QER ID: 1 
    QER ID: 3 
    URR ID: 1 
    Source Interface: 0 
    TEID: 1 
    Ipv4: 192.168.13.151 
    Ipv6: <nil> 
    UE IPv4 Address: 10.60.0.1 
    SDF Filter: permit out ip from 10.60.0.0/16 to any 
  CreatePDR ID: 4 
    FAR ID: 4 
    QER ID: 1 
    QER ID: 3 
    URR ID: 1 
    Source Interface: 2 
    UE IPv4 Address: 10.60.0.1 
    SDF Filter: permit out ip from any to 10.60.0.0/16 
  CreateFAR ID: 1 
    Apply Action: [2] 
    Forwarding Parameters:
      Network Instance:internet 
  CreateFAR ID: 2 
    Apply Action: [2] 
    Forwarding Parameters:
  CreateFAR ID: 3 
    Apply Action: [2] 
    Forwarding Parameters:
      Network Instance:internet 
  CreateFAR ID: 4 
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
    QFI: 0 
  CreateQER ID: 3 
    Gate Status DL: 0 
    Gate Status UL: 0 
    QFI: 2 
  CreateURR ID: 1 
    Measurement Method: 2 
    Volume Threshold: &{Flags:6 TotalVolume:0 UplinkVolume:1000 DownlinkVolume:1000} 
  CreateURR ID: 2 
    Measurement Method: 2 
    Volume Threshold: &{Flags:6 TotalVolume:0 UplinkVolume:1000 DownlinkVolume:1000} 

2023/11/26 08:45:10 INF WARN: No OuterHeaderCreation
2023/11/26 08:45:10 INF Saving FAR info to session: 1, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:2534254784 TransportLevelMarking:0}
2023/11/26 08:45:10 INF WARN: No OuterHeaderCreation
2023/11/26 08:45:10 INF Saving FAR info to session: 2, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:2534254784 TransportLevelMarking:0}
2023/11/26 08:45:10 INF WARN: No OuterHeaderCreation
2023/11/26 08:45:10 INF Saving FAR info to session: 3, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:2534254784 TransportLevelMarking:0}
2023/11/26 08:45:10 INF WARN: No OuterHeaderCreation
2023/11/26 08:45:10 INF Saving FAR info to session: 4, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:2534254784 TransportLevelMarking:0}
2023/11/26 08:45:10 INF Saving QER info to session: 2, {GateStatusUL:0 GateStatusDL:0 Qfi:1 MaxBitrateUL:0 MaxBitrateDL:0 StartUL:0 StartDL:0}
2023/11/26 08:45:10 INF Saving QER info to session: 1, {GateStatusUL:0 GateStatusDL:0 Qfi:0 MaxBitrateUL:1000000000 MaxBitrateDL:1000000000 StartUL:0 StartDL:0}
2023/11/26 08:45:10 INF Saving QER info to session: 3, {GateStatusUL:0 GateStatusDL:0 Qfi:2 MaxBitrateUL:0 MaxBitrateDL:0 StartUL:0 StartDL:0}
2023/11/26 08:45:10 Matched groups: [permit out ip from 10.60.0.0/16 to any ip 10.60.0.0 16  any  ]
2023/11/26 08:45:10 Matched groups: [permit out ip from any to 10.60.0.0/16 ip any   10.60.0.0 16 ]
2023/11/26 08:45:10 INF Session Establishment Request from 192.168.14.141 accepted.
2023/11/26 08:45:11 INF Got Session Modification Request from: 192.168.14.141. 

2023/11/26 08:45:11 INF Finding association for 192.168.14.141
2023/11/26 08:45:11 INF Finding session 2
2023/11/26 08:45:11 INF 
Session Modification Request:
  UpdatePDR ID: 2 
    FAR ID: 2 
    Source Interface: 2 
    UE IPv4 Address: 10.60.0.1 
  UpdatePDR ID: 4 
    FAR ID: 4 
    Source Interface: 2 
    UE IPv4 Address: 10.60.0.1 
    SDF Filter: permit out ip from any to 10.60.0.0/16 
  UpdateFAR ID: 2 
    Apply Action: [2] 
    Update forwarding Parameters:
      Network Instance:internet 
      Outer Header Creation: &{OuterHeaderCreationDescription:256 TEID:1 IPv4Address:192.168.13.131 IPv6Address:<nil> PortNumber:0 CTag:0 STag:0} 
  UpdateFAR ID: 4 
    Apply Action: [2] 
    Update forwarding Parameters:
      Network Instance:internet 
      Outer Header Creation: &{OuterHeaderCreationDescription:256 TEID:1 IPv4Address:192.168.13.131 IPv6Address:<nil> PortNumber:0 CTag:0 STag:0} 

2023/11/26 08:45:11 INF Updating FAR info: 2, {FarInfo:{Action:2 OuterHeaderCreation:1 Teid:1 RemoteIP:2198710464 LocalIP:2534254784 TransportLevelMarking:0} GlobalId:1}
2023/11/26 08:45:11 INF Updating FAR info: 4, {FarInfo:{Action:2 OuterHeaderCreation:1 Teid:1 RemoteIP:2198710464 LocalIP:2534254784 TransportLevelMarking:0} GlobalId:3}
2023/11/26 08:45:11 INF Both F-TEID IE and UE IP Address IE are missing
2023/11/26 08:45:11 INF Both F-TEID IE and UE IP Address IE are missing
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.60.0.1` from free5GC 5GC.
```
[2023-11-26 08:45:11.054] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
8: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/32 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::cfa:4c2e:b68e:c9c7/64 scope link stable-privacy 
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

Run `tcpdump` on VM-DN and check that the packet goes through N6 (enp0s9).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (172.217.26.238) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 172.217.26.238: icmp_seq=1 ttl=61 time=18.8 ms
64 bytes from 172.217.26.238: icmp_seq=2 ttl=61 time=17.5 ms
64 bytes from 172.217.26.238: icmp_seq=3 ttl=61 time=18.9 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
08:48:34.404197 IP 10.60.0.1 > 172.217.26.238: ICMP echo request, id 5, seq 1, length 64
08:48:34.420756 IP 172.217.26.238 > 10.60.0.1: ICMP echo reply, id 5, seq 1, length 64
08:48:35.404355 IP 10.60.0.1 > 172.217.26.238: ICMP echo request, id 5, seq 2, length 64
08:48:35.419801 IP 172.217.26.238 > 10.60.0.1: ICMP echo reply, id 5, seq 2, length 64
08:48:36.405598 IP 10.60.0.1 > 172.217.26.238: ICMP echo request, id 5, seq 3, length 64
08:48:36.422188 IP 172.217.26.238 > 10.60.0.1: ICMP echo reply, id 5, seq 3, length 64
```
- See `/sys/kernel/debug/tracing/trace_pipe` on VM-UP
```
# cat /sys/kernel/debug/tracing/trace_pipe
...
          <idle>-0       [000] d.s31  5014.547520: bpf_trace_printk: upf: gtp-u received
          <idle>-0       [000] d.s31  5014.547524: bpf_trace_printk: SDF: filter protocol: 1
          <idle>-0       [000] d.s31  5014.547529: bpf_trace_printk: SDF: filter source ip: 10.60.0.0, destination ip: 0.0.0.0
          <idle>-0       [000] d.s31  5014.547531: bpf_trace_printk: SDF: filter source ip mask: 255.255.0.0, destination ip mask: 0.0.0.0
          <idle>-0       [000] d.s31  5014.547532: bpf_trace_printk: SDF: filter source port lower bound: 0, source port upper bound: 65535
          <idle>-0       [000] d.s31  5014.547534: bpf_trace_printk: SDF: filter destination port lower bound: 0, destination port upper bound: 65535
          <idle>-0       [000] d.s31  5014.547535: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] d.s31  5014.547536: bpf_trace_printk: SDF: packet source ip: 10.60.0.1, destination ip: 172.217.26.238
          <idle>-0       [000] d.s31  5014.547538: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] d.s31  5014.547575: bpf_trace_printk: Packet with source ip: 10.60.0.1, destination ip: 172.217.26.238 matches SDF filter
          <idle>-0       [000] d.s31  5014.547577: bpf_trace_printk: upf: sdf filter matches teid:1
          <idle>-0       [000] d.s31  5014.547578: bpf_trace_printk: upf: far:2 action:2 outer_header_creation:0
          <idle>-0       [000] d.s31  5014.547580: bpf_trace_printk: upf: qer:1 gate_status:0 mbr:1000000000
          <idle>-0       [000] d.s31  5014.547582: bpf_trace_printk: upf: session for teid:1 far:0 outer_header_removal:0
          <idle>-0       [000] d.s31  5014.547592: bpf_trace_printk: upf: bpf_fib_lookup 10.60.0.1 -> 172.217.26.238: nexthop: 192.168.16.152
          <idle>-0       [000] d.s31  5014.564475: bpf_trace_printk: SDF: filter protocol: 1
          <idle>-0       [000] d.s31  5014.564481: bpf_trace_printk: SDF: filter source ip: 0.0.0.0, destination ip: 10.60.0.0
          <idle>-0       [000] d.s31  5014.564518: bpf_trace_printk: SDF: filter source ip mask: 0.0.0.0, destination ip mask: 255.255.0.0
          <idle>-0       [000] d.s31  5014.564520: bpf_trace_printk: SDF: filter source port lower bound: 0, source port upper bound: 65535
          <idle>-0       [000] d.s31  5014.564522: bpf_trace_printk: SDF: filter destination port lower bound: 0, destination port upper bound: 65535
          <idle>-0       [000] d.s31  5014.564522: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] d.s31  5014.564525: bpf_trace_printk: SDF: packet source ip: 172.217.26.238, destination ip: 10.60.0.1
          <idle>-0       [000] d.s31  5014.564526: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] d.s31  5014.564528: bpf_trace_printk: Packet with source ip: 172.217.26.238, destination ip: 10.60.0.1 matches SDF filter
          <idle>-0       [000] d.s31  5014.564530: bpf_trace_printk: Packet with source ip:172.217.26.238 and destination ip:10.60.0.1 matches SDF filter
          <idle>-0       [000] d.s31  5014.564533: bpf_trace_printk: upf: downlink session for ip:10.60.0.1  far:3 action:2
          <idle>-0       [000] d.s31  5014.564535: bpf_trace_printk: upf: qer:1 gate_status:0 mbr:1000000000
          <idle>-0       [000] d.s31  5014.564537: bpf_trace_printk: upf: use mapping 10.60.0.1 -> TEID:1
          <idle>-0       [000] d.s31  5014.564539: bpf_trace_printk: upf: send gtp pdu 192.168.13.151 -> 192.168.13.131
          <idle>-0       [000] d.s31  5014.564548: bpf_trace_printk: upf: bpf_fib_lookup 192.168.13.151 -> 192.168.13.131: nexthop: 192.168.13.131
          <idle>-0       [000] d.s31  5015.547664: bpf_trace_printk: upf: gtp-u received
          <idle>-0       [000] d.s31  5015.547670: bpf_trace_printk: SDF: filter protocol: 1
          <idle>-0       [000] d.s31  5015.547674: bpf_trace_printk: SDF: filter source ip: 10.60.0.0, destination ip: 0.0.0.0
          <idle>-0       [000] d.s31  5015.547676: bpf_trace_printk: SDF: filter source ip mask: 255.255.0.0, destination ip mask: 0.0.0.0
          <idle>-0       [000] d.s31  5015.547678: bpf_trace_printk: SDF: filter source port lower bound: 0, source port upper bound: 65535
          <idle>-0       [000] d.s31  5015.547680: bpf_trace_printk: SDF: filter destination port lower bound: 0, destination port upper bound: 65535
          <idle>-0       [000] d.s31  5015.547681: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] d.s31  5015.547683: bpf_trace_printk: SDF: packet source ip: 10.60.0.1, destination ip: 172.217.26.238
          <idle>-0       [000] d.s31  5015.547685: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] d.s31  5015.547687: bpf_trace_printk: Packet with source ip: 10.60.0.1, destination ip: 172.217.26.238 matches SDF filter
          <idle>-0       [000] d.s31  5015.547689: bpf_trace_printk: upf: sdf filter matches teid:1
          <idle>-0       [000] d.s31  5015.547691: bpf_trace_printk: upf: far:2 action:2 outer_header_creation:0
          <idle>-0       [000] d.s31  5015.547692: bpf_trace_printk: upf: qer:1 gate_status:0 mbr:1000000000
          <idle>-0       [000] d.s31  5015.547694: bpf_trace_printk: upf: session for teid:1 far:0 outer_header_removal:0
          <idle>-0       [000] d.s31  5015.547705: bpf_trace_printk: upf: bpf_fib_lookup 10.60.0.1 -> 172.217.26.238: nexthop: 192.168.16.152
          <idle>-0       [000] d.s31  5015.563566: bpf_trace_printk: SDF: filter protocol: 1
          <idle>-0       [000] d.s31  5015.563571: bpf_trace_printk: SDF: filter source ip: 0.0.0.0, destination ip: 10.60.0.0
          <idle>-0       [000] d.s31  5015.563574: bpf_trace_printk: SDF: filter source ip mask: 0.0.0.0, destination ip mask: 255.255.0.0
          <idle>-0       [000] d.s31  5015.563576: bpf_trace_printk: SDF: filter source port lower bound: 0, source port upper bound: 65535
          <idle>-0       [000] d.s31  5015.563579: bpf_trace_printk: SDF: filter destination port lower bound: 0, destination port upper bound: 65535
          <idle>-0       [000] d.s31  5015.563580: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] d.s31  5015.563582: bpf_trace_printk: SDF: packet source ip: 172.217.26.238, destination ip: 10.60.0.1
          <idle>-0       [000] d.s31  5015.563584: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] d.s31  5015.563587: bpf_trace_printk: Packet with source ip: 172.217.26.238, destination ip: 10.60.0.1 matches SDF filter
          <idle>-0       [000] d.s31  5015.563591: bpf_trace_printk: Packet with source ip:172.217.26.238 and destination ip:10.60.0.1 matches SDF filter
          <idle>-0       [000] d.s31  5015.563593: bpf_trace_printk: upf: downlink session for ip:10.60.0.1  far:3 action:2
          <idle>-0       [000] d.s31  5015.563596: bpf_trace_printk: upf: qer:1 gate_status:0 mbr:1000000000
          <idle>-0       [000] d.s31  5015.563598: bpf_trace_printk: upf: use mapping 10.60.0.1 -> TEID:1
          <idle>-0       [000] d.s31  5015.563601: bpf_trace_printk: upf: send gtp pdu 192.168.13.151 -> 192.168.13.131
          <idle>-0       [000] d.s31  5015.563611: bpf_trace_printk: upf: bpf_fib_lookup 192.168.13.151 -> 192.168.13.131: nexthop: 192.168.13.131
          <idle>-0       [000] d.s31  5016.548877: bpf_trace_printk: upf: gtp-u received
          <idle>-0       [000] d.s31  5016.548883: bpf_trace_printk: SDF: filter protocol: 1
          <idle>-0       [000] d.s31  5016.548889: bpf_trace_printk: SDF: filter source ip: 10.60.0.0, destination ip: 0.0.0.0
          <idle>-0       [000] d.s31  5016.548892: bpf_trace_printk: SDF: filter source ip mask: 255.255.0.0, destination ip mask: 0.0.0.0
          <idle>-0       [000] d.s31  5016.548894: bpf_trace_printk: SDF: filter source port lower bound: 0, source port upper bound: 65535
          <idle>-0       [000] d.s31  5016.548897: bpf_trace_printk: SDF: filter destination port lower bound: 0, destination port upper bound: 65535
          <idle>-0       [000] d.s31  5016.548898: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] d.s31  5016.548901: bpf_trace_printk: SDF: packet source ip: 10.60.0.1, destination ip: 172.217.26.238
          <idle>-0       [000] d.s31  5016.548902: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] d.s31  5016.548905: bpf_trace_printk: Packet with source ip: 10.60.0.1, destination ip: 172.217.26.238 matches SDF filter
          <idle>-0       [000] d.s31  5016.548907: bpf_trace_printk: upf: sdf filter matches teid:1
          <idle>-0       [000] d.s31  5016.548909: bpf_trace_printk: upf: far:2 action:2 outer_header_creation:0
          <idle>-0       [000] d.s31  5016.548911: bpf_trace_printk: upf: qer:1 gate_status:0 mbr:1000000000
          <idle>-0       [000] d.s31  5016.548914: bpf_trace_printk: upf: session for teid:1 far:0 outer_header_removal:0
          <idle>-0       [000] d.s31  5016.548926: bpf_trace_printk: upf: bpf_fib_lookup 10.60.0.1 -> 172.217.26.238: nexthop: 192.168.16.152
          <idle>-0       [000] d.s31  5016.565941: bpf_trace_printk: SDF: filter protocol: 1
          <idle>-0       [000] d.s31  5016.565948: bpf_trace_printk: SDF: filter source ip: 0.0.0.0, destination ip: 10.60.0.0
          <idle>-0       [000] d.s31  5016.565951: bpf_trace_printk: SDF: filter source ip mask: 0.0.0.0, destination ip mask: 255.255.0.0
          <idle>-0       [000] d.s31  5016.565953: bpf_trace_printk: SDF: filter source port lower bound: 0, source port upper bound: 65535
          <idle>-0       [000] d.s31  5016.565955: bpf_trace_printk: SDF: filter destination port lower bound: 0, destination port upper bound: 65535
          <idle>-0       [000] d.s31  5016.565956: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] d.s31  5016.565959: bpf_trace_printk: SDF: packet source ip: 172.217.26.238, destination ip: 10.60.0.1
          <idle>-0       [000] d.s31  5016.565960: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] d.s31  5016.565963: bpf_trace_printk: Packet with source ip: 172.217.26.238, destination ip: 10.60.0.1 matches SDF filter
          <idle>-0       [000] d.s31  5016.565966: bpf_trace_printk: Packet with source ip:172.217.26.238 and destination ip:10.60.0.1 matches SDF filter
          <idle>-0       [000] d.s31  5016.565969: bpf_trace_printk: upf: downlink session for ip:10.60.0.1  far:3 action:2
          <idle>-0       [000] d.s31  5016.565971: bpf_trace_printk: upf: qer:1 gate_status:0 mbr:1000000000
          <idle>-0       [000] d.s31  5016.565973: bpf_trace_printk: upf: use mapping 10.60.0.1 -> TEID:1
          <idle>-0       [000] d.s31  5016.565977: bpf_trace_printk: upf: send gtp pdu 192.168.13.151 -> 192.168.13.131
          <idle>-0       [000] d.s31  5016.565986: bpf_trace_printk: upf: bpf_fib_lookup 192.168.13.151 -> 192.168.13.131: nexthop: 192.168.13.131
...
```
You could specify the IP address assigned to the TUNnel interface to run almost any applications as in the following example using `nr-binder` tool.

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
08:51:05.081459 IP 10.60.0.1.44263 > 172.217.26.238.80: Flags [S], seq 787533438, win 65280, options [mss 1360,sackOK,TS val 3494135696 ecr 0,nop,wscale 7], length 0
08:51:05.097140 IP 172.217.26.238.80 > 10.60.0.1.44263: Flags [S.], seq 2560001, ack 787533439, win 65535, options [mss 1460], length 0
08:51:05.099124 IP 10.60.0.1.44263 > 172.217.26.238.80: Flags [.], ack 1, win 65280, length 0
08:51:05.099424 IP 10.60.0.1.44263 > 172.217.26.238.80: Flags [P.], seq 1:75, ack 1, win 65280, length 74: HTTP: GET / HTTP/1.1
08:51:05.099575 IP 172.217.26.238.80 > 10.60.0.1.44263: Flags [.], ack 75, win 65535, length 0
08:51:05.154834 IP 172.217.26.238.80 > 10.60.0.1.44263: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
08:51:05.156657 IP 10.60.0.1.44263 > 172.217.26.238.80: Flags [.], ack 774, win 64507, length 0
08:51:05.160215 IP 10.60.0.1.44263 > 172.217.26.238.80: Flags [F.], seq 75, ack 774, win 64507, length 0
08:51:05.160386 IP 172.217.26.238.80 > 10.60.0.1.44263: Flags [.], ack 76, win 65535, length 0
08:51:05.175600 IP 172.217.26.238.80 > 10.60.0.1.44263: Flags [F.], seq 774, ack 76, win 65535, length 0
08:51:05.177037 IP 10.60.0.1.44263 > 172.217.26.238.80: Flags [.], ack 775, win 64507, length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using eUPF.

---

Now you could work free5GC with eUPF.
I would like to thank the excellent developers and all the contributors of free5GC, eUPF and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2023.11.26] Updated to eUPF `120-upf-ftup-fteid` branch that supports FTUP. Although free5GC does not support FTUP in PDU Session Establishment, I confirmed its operation on `120-upf-ftup-fteid` branch in order to compare it with the logs of Open5GS which supports FTUP.
- [2023.10.29] Initial release.
