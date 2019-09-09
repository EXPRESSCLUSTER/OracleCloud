Oracle Cloud Infrastructure (by internal load balancer)
===

About this guide
---
This guide describes how to setup EXPRESSCLUSTER X mirror disk type cluster on Oracle Cloud Infrastructure with internal load balancer.
For the detailed information of EXPRESSCLUSTER X, please refer to [this site](https://www.nec.com/en/global/prod/expresscluster/index.html).  

Configuration
---
### Overview
In this guide, create 2 nodes (Node1 Node2 as below) mirror disk type cluster.  
Each node has a block storage and date on both the block storages are synchronize by cluster mirroring feature.  
EXPRESSCLUSTER uses Oracle Cloud Infrastructure load balancer health check feature and it switches Active node and Standby node when it detects unhealthy status.  
Client can access to Applications on Active node in the virtual cloud network by specifying Load balancer IP address.  

<div align="center">
<img src="https://user-images.githubusercontent.com/52775132/62447428-4c0bc180-b7a0-11e9-92c6-a95233739474.png">
</div>

### Software versions
- In the case of Linux
  - Cent OS 6.10 (2.6.32-754.14.2.el6.x86_64)
    or
    Cent OS 7.6 (3.10.0-957.12.2.el7.x86_64)
  - EXPRESSCLUSTER X 4.1 for Linux (internal version：4.1.1-1)
- In the case of Windows
  - Windows Server 2016 Standard
  - EXPRESSCLUSTER X 4.1 for Windows (internal version：12.11)

### Cluster configurations
- Group resources
  - mirror disk resource
  - Azure probe port resource
- Monitor resources
     - mirror disk connect monitor resource
     - mirror disk monitor resource
     - Azure probe port monitor resource
     - Azure load balance monitor resource
- Network partition resolution resource
  - network partition resolution resource by PING method
- HeartBeat resources
  - kernel mode LAN heartbeat resource

Oracle Cloud setup
---
1. Create Instances
   - Create 2 Instances for cluster nodes with separating the fault domain by Advanced Options
     - Node1
        - availability domain：AD 1 (oIJw:AP-TOKYO-1-AD-1) 
        - fault domain：FAULT-DOMAIN-1
        - public IP address：10.0.0.8
        - private IP address：10.0.10.8
     - Node2
        - availability domain：AD 1 (oIJw:AP-TOKYO-1-AD-1)
        - fault domain：FAULT-DOMAIN-2
        - public IP address：10.0.0.9
        - private IP address：10.0.10.9
1. Create Block Volumes
   - Create 2 Block Volumes
1. Attach Block Volumes to each instance.
   - In the case of Linux (It will not be necessary for Windows)
     - Select DEVICE PATH(/dev/oracleoci/oraclevdb).
   - Attach by iscsi command.
1. Create a Load Balancer
   - Create 1 Load Balancer	
   - CHOOSE VISIBILITY TYPE : Private	
   - Skip "Choose Backends"
   - Configure the Update Health Check
     - PROTOCOL：TCP
     - PORT：26001
     - INTERNAL IN MS：5000
     - TIMEOUT IN MS：3000
     - NUMBUR OF RETRIES：2
   - Configure the Listener
     - PROTOCOL：TCP
     - PORT：80
1. Add the Backends to Load Balancer
   - Add the Backends which specifying the cluster node ip address before configuring the Load Balancer
     - Choose how to add backend servers by selecting compute instances or by entering IP addresses.：IP ADDRESSES
     - Node1
       - IPAddress：10.0.10.8
       - Port：8080
     - Node2
       - IPAddress：10.0.10.9
       - Port：8080

Setup EXPRESSCLUSTER X
---
Other parameters than the below, default values are set.

1. Configure the partitions for mirror disk.
   - In the case of Linux
     - /dev/oracleoci/oraclevdb1：no format (RAW)
     - /dev/oracleoci/oraclevdb2：format with ext4
   - In the case of Windows
     - D:\ ：no format
     - E:\ ：format with NTFS
1. Install EXPRESSCLUSTER and register licenses.
1. In Config mode of Cluster WebUI, execute Cluster generation wizard.
1. Configure Basic Settings and Interconnect.
   - interconnect1
     - Node1：10.0.0.8
     - Node2：10.0.0.9
     - MDC：do not use
   - interconnect2
     - Node1：10.0.10.8
     - Node2：10.0.10.9
     - MDC：mdc1
1. Configure NP Resolution.
   - Type：Ping
   - Ping Target：10.0.0.1
1. Add Failover Group and configure it.
1. Add mirror disk resource and configure it.
  - In the case of Linux
    - Details
      - Mirror Partition Device Name：/dev/NMP1
      - Mount Point：/mnt/md1
      - Data Partition Device Name：/dev/oracleoci/oraclevdb2
      - Cluster Partition Device Name：/dev/oracleoci/oraclevdb1
      - File System：ext4
  - In the case of Windows
    - Details
      - Date Partition Drive Letter：E:\
      - Cluster Partition Drive Letter：D:\
      - Mirror Disk Connect：mdc1
      - Servers that can run the group：Node1, Node2
1. Add Azure probe port resource and configure it.
   - Details
     - Probeport：26001
1. Confirm that monitor resources are added.
   - The following monitor resources are automatically added when setting failover group resources.
   - In the case of Linux
     - mirror disk connect monitor resource
     - mirror disk monitor resource
     - Azure probe port monitor resource
     - Azure load balance monitor resource
   - In the case of Windows
     - mirror connect monitor resource
     - mirror disk monitor resource
     - Azure probe port monitor resource
     - Azure load balance monitor resource
1. In Config mode of Cluster WebUI, execute Apply the Configuration File.

Check the operation for EXPRESSCLUSTER X
---
1. Please look up how to check the operation EXPRESSCLUSTER X in the URL below.

Reference
---
- EXPRESSCLUSTER X 4.1 HA Cluster Configuration Guide for Microsoft Azure (Linux)
   - https://www.nec.com/en/global/prod/expresscluster/en/support/setup/HOWTO_Azure_X41_Linux_EN_01.pdf
- EXPRESSCLUSTER X 4.1 HA Cluster Configuration Guide for Microsoft Azure (Windows)
   - https://www.nec.com/en/global/prod/expresscluster/en/support/setup/HOWTO_Azure_X41_Windows_EN_01.pdf
