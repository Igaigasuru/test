Oracle Cloud Infrastructure (by public load balancer)
===

About this guide
---
This guide describes how to setup EXPRESSCLUSTER X of the mirror disk type cluster on Oracle Cloud Infrastructure.  
The following describes the cluster configuration by public load balancer.  
For the detailed information of EXPRESSCLUSTER X, please refer to [this site](https://www.nec.com/en/global/prod/expresscluster/index.html).  

Configuration
---
### Overview
In the configuration of this guide, create 2-server(Node1 Node2 as below) cluster of mirror disk type.  
And the date on block storage synchronize between nodes.  
And active and standby servers of the cluster are swiched by controlling the Oracle Cloud Infrastructure load balancer from EXPRESSCLUSTER.  
Client Applications will be accessible instance in the virtual cloud network if you specify Load balancer IP address.  

<div align="center">
<img src="https://user-images.githubusercontent.com/52775132/62447475-68a7f980-b7a0-11e9-9683-19133c4eabdd.png">
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
- Monitor resource
   - In the case of Linux
     - mirror disk connect monitor resource
     - mirror disk monitor resource
     - Azure probe port monitor resource
     - Azure load balance monitor resource
     - custom monitor resource (by network partition resource)
     - IP monitor resources (by network partition resource)
     - multi target monitor resource (by network partition resource)
   - In the case of Windows
     - mirror connect monitor resource
     - mirror disk monitor resource
     - Azure probe port monitor resource
     - Azure load balance monitor resource
     - custom monitor resource (by network partition resource)
     - IP monitor resources (by network partition resource)
     - multi target monitor resource (by network partition resource)

Oracle Cloud setup
---
1. Configure the Instances
   - Separate the fault domain by Advanced Options
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
1. Configure the Block Volumes
   - Configure the Block Volumes of 2 nodes
1. Attach Block Volumes to instance.
   - Select DEVICE PATH(/dev/oracleoci/oraclevdb).
   - Attach by iscsi command.
1. Configure the Load Balancer
   - CHOOSE VISIBILITY TYPE : Public
   - CHOOSE NETWORKING : by public subnet
   - Configure the Choose Backends
     - SELECT BACKEND SERVERS : Add 2-server(Node1, Node2)
       - Port : 8080 ( provide application's port in instance side )
   - Configure the Update Health Check
     - PROTOCOL：TCP
     - PORT：26001
     - INTERNAL IN MS：5000
     - TIMEOUT IN MS：3000
     - NUMBUR OF RETRIES：2
   - Configure the Listener
     - PROTOCOL：TCP
     - PORT：80 ( provide application's port in internet side )
1. if you need security list, configure the security list.

Setup EXPRESSCLUSTER X
---
Other parameters than below, default value is setting.

1. Configure the partition for mirror disk
   - In the case of Linux
     - /dev/oracleoci/oraclevdb1：no format (RAW)
     - /dev/oracleoci/oraclevdb2：format ext4
   - In the case of Windows
     - D:\ ：no format
     - E:\ ：format NTFS
1. Install EXPRESSCLUSTER and register license.
1. In Config mode of the Cluster WebUI, executing Cluster generation wizard.
1. Configure the Basic Settings and Interconnect.
   - interconnect1
     - Node1：10.0.0.8
     - Node2：10.0.0.9
     - MDC：do not use
   - interconnect2
     - Node1：10.0.10.8
     - Node2：10.0.10.9
     - MDC：mdc1
1. Configure the NP Resolution
   - Type：Ping
   - Ping Target：10.0.0.1
1. Configure the Failover Group
1. Configure the mirror disk resource
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
1. Configure the Azure probe port resource
   - Details
     - Probeport：26001
1. Configure the custom monitor resource
   - Info
      - Name : genw1
   - Monitor(special)
      - select the "Script create with this product" and push the "Edit"
          - In the case of Linux
            ```
               ! /bin/sh
               /opt/nec/clusterpro/bin/clpazure_port_checker -h iaas.ap-tokyo-1.oraclecloud.com -p 443
               exit $?
            ```
          - In the case of Windows
            ```
               "C:\Program Files\EXPRESSCLUSTER\bin\clpazure_port_checker" -h iaas.ap-tokyo-1.oraclecloud.com -p 443
               EXIT %ERRORLEVEL%
            ```
   - Recovery Action
      - Recovery Action : Execute only the final action
      - Recevery Target : LocalServer
      - Final Action: No operation
1. Configure the IP monitor resource (1st)
   - Info
      - Name : ipw1
   - Monitor(common)
      - Choose servers that execute monitoring
         - select "Select" and add the Node1
   - Monitor(special)
      - push the "Add" and enter the Node2's IP address (10.0.10.9)
   - Recovery Action
      - Recovery Action : Execute only the final action
      - Recevery Target : LocalServer
      - Final Action: No operation
1. Configure the IP monitor resource (2nd)
   - Info
      - Name : ipw2
   - Monitor(common)
      - Choose servers that execute monitoring
         - select "Select" and add the Node2
   - Monitor(special)
      - push the "Add" and enter the Node1's IP address (10.0.10.8)
   - Recovery Action
      - Recovery Action : Execute only the final action
      - Recevery Target : LocalServer
      - Final Action: No operation
1. Configure the multi target monitor resource
   - Monitor(common)
      - add genw1, ipw1, and ipw2
   - Monitor(special)
   - Recovery Action
      - Recovery Action : Execute only the final action
      - Recevery Target : LocalServer
      - Execute Script before Final Action : On
      - Final Action: No operation
      - Script Settings
         - select the "Script create with this product" and push the "Edit"
            - In the case of Linux
              ```#! /bin/sh
                 /opt/nec/clusterpro/bin/clpazure_port_checker -h 127.0.0.1 -p 8080
                 if [ $? -ne 0 ]
                 then
                 clpdown
                 exit 0
                 fi
     
                 /opt/nec/clusterpro/bin/clpazure_port_checker -h <IP address set to load balancer> -p 80
                 if [ $? -ne 0 ]
                 then
                 clpdown
                 exit 0
                 fi
                 ```
            - In the case of Windows
              ```
                  rem ********************
                  rem Check Active Node
                  rem ********************
                  "C:\Program Files\EXPRESSCLUSTER\bin\clpazure_port_checker" -h 127.0.0.1 -p 8080
                  IF NOT "%ERRORLEVEL%" == "0" (
                  GOTO CLUSTER_SHUTDOWN
                  )
                  rem ********************
                  rem Check DNS
                  rem ********************
                  "C:\Program Files\EXPRESSCLUSTER\bin\clpazure_port_checker" -h <IP address set to load balancer> -p 80
                  IF "%ERRORLEVEL%" == "0" (
                  GOTO EXIT
                  )
                  rem ********************
                  rem Cluster Shutdown
                  rem ********************
                  :CLUSTER_SHUTDOWN
                  clpdown
                  rem ********************
                  rem EXIT
                  rem ********************
                  :EXIT
                  EXIT 0
               ```
         - Timeout : 15 sec
1. The following that monitor resource is automatically registered when setting group resouces.
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

1. Configure the Cluster Properties
   - Timeout
       - HeartBeat
            - Timeout : 120 sec ( over as below sum(A, B, C) value )
               - A : ipw1, ipw2, mtw1's Interval * (Retry Count - 1)
                     (select the most largest value)  
               - B : mtw's Interval * (Retry Count - 1)
               - C : 30 sec
1. In Config mode of the Cluster WebUI, executing Apply the Configuration File.

Check the operation for EXPRESSCLUSTER X
---
1. Please look up how to check the operation EXPRESSCLUSTER X in the URL below.

Reference
---
- EXPRESSCLUSTER X 4.1 HA Cluster Configuration Guide for Microsoft Azure (Linux)
   - https://www.nec.com/en/global/prod/expresscluster/en/support/setup/HOWTO_Azure_X41_Linux_EN_01.pdf
- EXPRESSCLUSTER X 4.1 HA Cluster Configuration Guide for Microsoft Azure (Windows)
   - https://www.nec.com/en/global/prod/expresscluster/en/support/setup/HOWTO_Azure_X41_Windows_EN_01.pdf
