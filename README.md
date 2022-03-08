### SQL Server 2019 Active / Active Cluster on Windows Server 2019 Cluster


# Intruduction
In this article we are going to see how to configure SQL Server 2019 Active / Active Cluster. Following are the pre-requisit for that

## Softwares needed
1. Windows Server 2019 Evaluation Edition - https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019

3. Vritualization Software - https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html


# Configuring Windows Server Cluster
Very first step in configuring SQL Server 2019 cluster is to have Windows Cluster first. Now there are many steps involved while creating Windows Cluster itself. Following are the steps which you can follow in-order to configure Windows Server Cluster

## 1. Installation of Windows Server 2019 Server
1. Install Windows Server 2019 by creating a Virtual Machine
2. Change machine name 
3. Allocate IP Address 
   #### Computer Name 
        1. This PC --> Properties --> Advanced System Settings --> Computer Name (GOSI-DC)
   #### IP Addresses
        1. Control Panel --> Network & Internet --> Network and Sharing Center --> Ethernet0 --> Properties --> Internet Protocol Verstion 4 --> (TCP/IPv4)
           - Static IP Address    : 192.168.80.10
           - Subnet Mask          : 255.255.255.0
           - Default Gateway      : 192.168.80.2  [Same for all VMs] 
           - Preferred DNS Server : 192.168.80.10 [Same for all VMs]
           - Alternet DNS Server  : 192.168.80.2 
   #### Disable Firewall
        1. Control Panel --> System and Security --> Windows Defender Firewall --> Turn Off Windows Defender Firewall
## 2. Domain Controller Configurations   
### Installation of Active Directory
    1. Open Server Manager
    2. Local Server --> Manage --> Add Roles & Features
    3. Installation Type --> Role Based Installation
    4. Select local server - this is a default option
    5. Server Roles --> Select - Active Directory Domain Services 
    6. Install
### Configuring Active Directory Domain Service
    1. Click on AD-DS Role at Server Manager
    2. Click More
    3. Select Action - Promote server to domain controller
    4. Add a new forest (Since we don't have any existing domain)
    5. Give domain name as - gosi.gov.sa [you can give any name which you want eg. abc.xyz.pqr]
    6. Click Next
    7. Specify DSRM (directory service recovery mode) password
    8. No need to create DNS deligation
    9. NetBIOS domain name -- keep it default 
    10. Paths - Keep it default
    11. Install
    12. Restart & Login with Doamin User this time
### Validating Active Directory 
    1. Open "Active Directory Users and Computers"

## 3. Installation of 2 Nodes
### Installation of node-1
1. Install Windows Server 2019 by creating a Virtual Machine
2. Change machine name 
3. Allocate IP Address
   #### Computer Name 
        1. This PC --> Properties --> Advanced System Settings --> Computer Name (GOSI-NODE1)
   #### IP Addresses
        1. Control Panel --> Network & Internet --> Network and Sharing Center --> Ethernet0 --> Properties --> Internet Protocol Verstion 4 --> (TCP/IPv4)
           - Static IP Address    : 192.168.80.15 
           - Subnet Mask          : 255.255.255.0 
           - Default Gateway      : 192.168.80.2  [Same for all VMs]
           - Preferred DNS Server : 192.168.80.10 [Same for all VMs] 
           - Alternet DNS Server  : Keep blank 
   #### Disable Firewall
        1. Control Panel --> System and Security --> Windows Defender Firewall --> Turn Off Windows Defender Firewall
 ### Installation of node-2
1. Install Windows Server 2019 by creating a Virtual Machine
2. Change machine name 
3. Allocate IP Address
   #### Computer Name 
        1. This PC --> Properties --> Advanced System Settings --> Computer Name (GOSI-NODE2)
   #### IP Addresses
        1. Control Panel --> Network & Internet --> Network and Sharing Center --> Ethernet0 --> Properties --> Internet Protocol Verstion 4 --> (TCP/IPv4)
           - Static IP Address    : 192.168.80.20 
           - Subnet Mask          : 255.255.255.0 
           - Default Gateway      : 192.168.80.2 [Same for all VMs] 
           - Preferred DNS Server : 192.168.80.10 [Same for all VMs] 
           - Alternet DNS Server  : Keep blank
   #### Disable Firewall
        1. Control Panel --> System and Security --> Windows Defender Firewall --> Turn Off Windows Defender Firewall

## 4. Adding nodes to Domain
1. Validate if you can ping to Domain from both the nodes - ping gosi.gov.sa
2. Assign domain name for both nodes 
   - This PC --> Properties --> Advanced System Settings --> Member Of Domain - gosi.gov.sa
   - Specify credentials for Domain Admin - GOSI\Administrator & P@ssword1
   - Restart server
   - Follow same steps for both nodes
   - While logging you should be able to login as Domain Administrator user instead of local Administrator
   - Validate nodes in Domain Controller using "Active Directory Users and Computers"

## 5. Installation of iScassi Target
DC Server / Node will act as Storage or SAN. We will create 6 drives there as below:
1. 2 Data Drives (5 GB Each) - Data01 (G) & Data02 (H)
2. 2 Log Drives (3 GB Each) - Log01 (L) & Log02 (M)
3. 2 Temp Drives (2 GB Each) - Temp01 (T) & Temp02 (U)
4. Quorum Drive (1 GB)

These drives will share with other nodes, wherein we will actually install & configure SQL Server 2019 Active \ Active Cluster.

Now, in order to create & share \ present these drives (iscasi luns) to other nodes, we need to follow following steps
### a. Installation of iSCSI Target Server
        1. Open Server Manager
        2. Local Server --> Manage --> Add Roles & Features
        3. Installation Type --> Role Based Installation
        4. Select local server - this is a default option
        5. Server Roles --> Select - File & Storage Services --> iSCSI Target Server
        6. Install        
### b. Add NIC Cards to all 3 nodes in network for iSCSI communication
        1. Player --> Manage --> Virtual Machine Settings 
        2. Click Add --> Select Network Adapter
        3. Assign static IP Address for all 3 nodes as below
           - for DC          -- 10.0.0.10
           - for GOSI-NODE1  -- 10.0.0.15
           - for GOSI-NODE2  -- 10.0.0.20
        4. Disable Windows Firewall
        5. Check the connectivity from DC
    
### c. Creating Shared Disks on Domain Controller
        1. Open Server Manager
        2. Select "File & Storage Services"
        3. Select "iSCSI"
        4. Tasks --> "New iSCSI Virtual Disk"
        5. Specify Disk Name
        6. Specify Disk Size
        7. Add Target IP Address 10.0.0.15 & 10.0.0.20
        8. Repeat process for all drives which needs to be added

### d. Configuring Shared Drives on GOSI-NODE1 & GOSI-NODE2
        1. Open "iSCSI initiator"
        2. Specify target - 10.0.0.10
        3. Click on Volumes & Devices --> Auto Configure
        4. To format disk - Open "Disk Management"
        5. Select disks and make online (Right click on each disk)
        6. Once its online -- right click --> Initialize Disk
        7. Select "GPT - GUID Partition Table" --> 
        8. Click on Drive (once its online) & click "New Simple Volume"
        9. Specify Drive Letter
        10. Specify following settings
           - File System - NTFS
           - Allocation Unit Size -- 64 KB 
           - Volume Label
           - Select "Perform Quick Format"
           - Finish
        11. Repeat same for all available drives
        12. Once node-1 configuration is finished, make all drives offline
        13. Open "iSCSI Initiator" on 2nd node
        14. Specify target - 10.0.0.10
        15. Click on Volumes & Devices --> Auto Configure
        16. Open "Disk Management"
        17. Select disks and make online (Right click on each disk)
        18. Right click on each drive --> Select "Change Drive Letter" & match it exactly what we had given for Node1


## 6. Installing & Configuring Windows Server 2019 Cluster
In order to configure SQL Server 2019 Cluster we should first have Windows Server 2019 cluster. This Windows Cluster 2019 will be between GOSI-NODE1 & GOSI-NODE2, as Domain Controller node is not required now.
### a. Add new Network Adapter Cards
    1. First step of this process is to add 2 separate NIC Cards to each node
    2. Assign IP Address to each of them as below
       - GOSI-NODE1 - 172.16.0.15
       - GOSI-NODE2 - 172.16.0.20
    3. Disable Firewall
    4. Restart both nodes
### b. Install fail-over cluster feature on both nodes
    1. Open Server Manager
    2. Local Server --> Manage --> Add Roles & Features
    3. Features --> Select "Failover Clustering"
    4. Click "Add Features"
    5. Install
    6. Restart the machine
    7. Do it for both the nodes
### c. Configure Windows Cluster
    1. Turn on DC
    2. Open "Failover Cluster Manager" in node1
    3. Select "Create Cluster"
    4. Add both nodes GOSI-NODE1 & GOSI-NODE2
    5. Perform validation by selecting all required tests
    6. Validate the report and check the status 
    7. Specify cluser name - WinServerCluster
    8. Specify IP Address - 192.168.80.25
    9. Select "Add all eligible storege to cluster"
    10. Click Finish
### d. Validate witness / quorum disk
    1. Right click on "WinServerCluster" --> More Actions --> Configure Cluster Quorum Settings
    2. Next --> select quorum witness
    3. Configure a disk witness
    4. Select witness disk (Q) from available disk
### e. Validate Cluster Failover    

## 7. SQL Server 2019 Installation - Prerequisite

### a. Creating Service Accounts
    1. Open "Active Directory Users and Computers" in DC
    2. Create AD Group "SQL ADMIN" for all Admins, and give Domain Admin or Local Admin access to this group.
    3. Add individual Domain Accounts to "SQL ADMIN". e.g. Add GOSI\3114 or GOSI\3115 to group "SQL ADMIN"
    4. Make AD Group "SQL ADMIN" as a Local Admin to each node
       - Open "Edit Local Users and Groups"
       - Under Groups section, select "Administrator" group and add "SQL ADMIN" in it.
    5. Create separate Service Accounts for each SQL Service as below
       - SQL.SERVER
       - SQL.AGENT   
       
### b. Grant Windows Rights to SQL Server Service account on both nodes
    1. Open "Local Group Policy Editor"
       Computer configuration --> Windows settings --> Security settings --> Local policies --> User rights assignment-->
    2. Add above listed SA accounts to following listing:
       - Perform volume maintenance tasks
       - Lock pages in memory

### c. Changing to CSVFS (Cluster Shared Volume File System) format
    1. Sharing Disks between 2 nodes
       - Open Failover Cluster Manager
       - Create 2 Empty Roles - Node1 & Node2
       - Assign Preferred Owner as Node1, Node2 & vice versa       
       - Assign Storage to each node
       

## 8. SQL Server 2019 Installation 
1. SQL Server Editions & Licencing 
    https://download.microsoft.com/download/6/6/0/66078040-86d8-4f6e-b0c5-e9919bbcb537/SQL%20Server%202019%20Licensing%20guide.pdf <br>
    
2. Download SQL Server Developer Edition from https://www.microsoft.com/en-us/sql-server/sql-server-downloads
    1. Once download, run the installer and choose "Download Media" option
    2. Select following options on next screen
       - Choose ISO image for download
       - Location as G:\Softwares\SQLServer2019
       - Click download
    3. Once download finishes, copy ISO image to respective node where you want to install SQL Server
    4. Click on ISO, and you can start SQL Server Installation by running Setup.exe
3. Installation Steps

    2. Re-Name Network in failover
       - Cluster Network 1 --> Cluster Nodes
       - Cluster Network 2 --> Shared Drives
       - Cluster Network 3 --> Heartbeat [This will verify if both nodes are up & running; if one of the node is down, it will start failover]
    3. Choose "Installation" --> "New SQL Server Failover Cluster Installation"
    4. Specify Edition --> (Developer)
    6. Accept Lincese Aggregement
    7. Feature Selection - We are going to install only "Database Engine Service".
    8. Select installation directories as follows
       - Instance Root Directory - C:\Program Files\Microsoft SQL Server\
       - Shared Feature Directory - C:\Program Files\Microsoft SQL Server\
       - Shared Feature Directory (x86) - C:\Program Files (x86)\Microsoft SQL Server\
    9. Specify SQL Server Network Name - GOSISERVER 
    10. This will be used to connect to SQL Server, you can skip named instance and keep default instance
    11. Specify Disks which you want to allocate to that SQL Server instance.
    12. Specify IP Address for SQL failover cluster - this has to be of same range \ network of your node - which is 192.168.80.30
    13. Specify Service Accounts & Change startup type to Automatic
       - SQL Server Agent - SQL.AGENT & Specify Password
       - SQL Server Database Engine - SQL.SERVER & Specify Password
       - Check on "Grant Perform Volume Maintenance Tasks privileges to SQL Server Database Engine Services"
    14. Select "Mixed Mode" Authentication
       - Specify password for SA account as "P@ssword1"
       - Add group "SQL ADMIN" as SQL Administrator
    15. Specify Data Directories     
       - Select data root directory as "C:\Data01\Data"
       - Select User database directory as "C:\Data01\Data"
       - Select User database log directory as "C:\Log01\Log"
       - Select Backup directory as "C:\Temp01\Temp"
    16. Temp DB      
        - Initial Size 1024 MB
        - Select data directory as - C:\Temp01\Temp
        - Select log directory as - C:\Temp01\Temp
        - Keep "Temp log file Size configuration" as default
    17. MaxDOP
        - Total no. of logical processors we have are 2 
    18. Memory
        - Click on "Recommended"
        - Since we have total 2 GB avilable for VM, change max memory to 1024 MB / 1 GB
        - Check on "Click here to accept the recommended memory configurations for the SQL Server Database Engine"
        
    19. FileStream
        - ignore this
        
    20. Click Next & the Install
    21. Installation will fail with error as below if you specify Temp directory directly as C:\ClusterStorage\Volume3. So, make sure you will create directory inside a drive before installation
        "Updating permission settings for folder "C:\ClusterStorage\Volume3" failed. Please check blog - https://blog.sqlauthority.com/2017/11/11/sql-server-installation-error-updating-permission-setting-file-failed/
    17. Cancel installation, create a new directory under Volume3 and re-start installation
       
4. Download SSMS from https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15    

5. Installing failover cluster
    1. Copy installer to 2nd node
    2. Choose "Installation" --> "Add a node to a SQL Server Failover Cluster"
    3. Select "Developer" Edition
    4. Accept "License Aggregement"
    5. All validations should be Ok
    6. Specify SQL Server Instance - in our case its a default instance, name of node2, Cluster Network Name (dgogate) & name of node1. This is all default and no need to modify anything here.
    7. IP Address of SQL Instance will appear automatically
    8. SQL Server & Agent Services and corresponding Service Accounts will appear; specify password. Select check box for volume maintenance task
    9. Install
    
5. Validation of SQL Server failover cluster
    1. Validate the ownership of LUNs / Disks - its not mandatory    
    2. Create a Dummy database and create a table in that. Run a sql query to validate the data
    3. Turn off one node
    4. You should still be able to connect to SQL Server from 2nd node and execute same query

6. Use Local SSD instead of Shared Drives / SAN for TempDB
    1.  Create directories under C:\MSSQL\Temp01
    2.  Move existing TempDB from Shared Drive to Local Drive
        - How to move DBs from one drive to another? https://docs.microsoft.com/en-us/sql/relational-databases/databases/move-system-databases?view=sql-server-ver15
    2.  Bring one of the node and validate
