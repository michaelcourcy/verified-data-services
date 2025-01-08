
# Architecture Reference for using DH2I MSSQL with Kasten By Veeam

## Goal 

An architecture reference that explains how to use the DH2I MSSQL operator and Kasten by Veeam together.

D2H2I operator creates MSSQL Availability group and listeners on kubernetes for High Availability and failover MSSQL database.

## Architecture card


| Description                                   | Values                           | Comment                   |
|-----------------------------------------------|----------------------------------|---------------------------|
| Database                                      | MSSQL                            | MSSQL with Availability group | 
| Database version tested                       | Microsoft SQL Server 2022 (RTM-CU16) (KB5048033) - 16.0.4165.4 (X64) <br> Nov  6 2024 19:24:49 Developer Edition (64-bit) on Linux (Ubuntu 22.04.5 LTS) <X64> | docker image tag : mcr.microsoft.com/mssql/server:latest |
| Operator vendor                               | [DH2I](https://dh2i.com)         | License required          |
| Operator vendor validation                    | In progress                      |                           |
| Operator version tested                       | docker.io/dh2i/dxoperator:1.0    |                           |
| High Availability                             | Yes                              | The database must be added to the Availability group see install AdaventureWorks2022 example <br> a failover test scenario is proposed |
| Unsafe backup & restore without pods errors   | Yes                              | See unsafe backup and restore section |
| PIT (Point In Time) supported                 | Yes                              | See the limitation section |
| Blueprint and BlueprintBinding example        | Yes                              | See the limitation section |
| Blueprint actions                             | Backup & restore                 | Delete is done through restorepoint deletion as backup artifacts are living in a shared PVC |
| Backup performance impact on the database     | None                             | Backup done on secondary replica |


# Limitations 

Make sure you properly understand the limitations of this architecture reference.

- PIT restore is only possible between 2 backups not after the last backup
- PIT restore must be done manually after a regular restore complete, the blueprint does not support PIT restore but the procedure is fully described in this document.
- The Blueprint only backup and restore the database that are belonging to the availability group
- The blueprint does not support backup on the non-synchronous replica only on the synchronous replica 

Overcoming all this limitations is always possible. This is the duty of the customer or his partner to tradeoff functionalities against complexities. 


# Architecture diagrams

```
+---------------------------+       +---------------------------+       +---------------------------+
|                           |       |                           |       |                           |
|  SQL Server Instance 1    |       |  SQL Server Instance 2    |       |  SQL Server Instance 3    |
|  (Primary Replica)        |       |  (Secondary Replica)      |       |  (Secondary Replica)      |
|                           |       |                           |       |                           |
|  /var/opt/mssql           |       |  /var/opt/mssql           |       |  /var/opt/mssql           | 
|  (mssql-dxesqlag-0 pvc)   |       |  (mssql-dxesqlag-1 pvc)   |       |  (mssql-dxesqlag-2 pvc)   |
|  /etc/dh2i                |       |  /etc/dh2i                |       |  /etc/dh2i                |                             
|  (dxe-dxesqlag-0 pvc)     |       |  (dxe-dxesqlag-1 pvc)     |       |  (dxe-dxesqlag-2 pvc)     |
|  /backup                  |       |  /backup                  |       |  /backup                  |                              
|  (shared backup pvc)      |       |  (shared backup pvc)      |       |  (shared backup pvc)      |
+-------------+-------------+       +-------------+-------------+       +-------------+-------------+
              |                                   |                                   |           ^
              |                                   |                                   |           |
              +-----------------------------------+-----------------------------------+           |
                                                  |                                               |                                   
                                                  |                                               |                                   
                                                  v                                               |                                   
                                        +---------------------+                                   |
                                        | Availability Group  |                                   |
                                        +---------------------+                                   |
                                                  |                                               |
                                                  |                                               |
     +---------------------------+                v                            Backup databases   |  
     |    All PVCs in namespace  |      +---------------------+              on replica:/backup/  | 
     |                           |      |  Load Balancer      |                                   | 
     |  mssql-dxesqlag-0/1/2 pvc |      |  Listener 14033     |                                   |
     | dxe-dxesqlag-0/1/2 pvc    |      +---------------------+                                   |
     |    shared backup pvc      |                ^                                               |
     +---------------------------+                |                                               |
                    ^                             |                                               |
                    |                             | Restore databases                             |
   Backup All PVCs  |                             | on primary:/backup/                           |
   Restore All PVCs |                             |                                               |
                    |                   +---------------------+                                   |
                    +-------------------|  Kasten K10         |-------------------+---------------+
                                        |  Backup & Restore   |
                                        +---------------------+
                                                  |
                                                  | export 
                                                  v
                                        +---------------------+
                                        |  S3 Storage         |
                                        |  (Backup Storage)   |
                                        +---------------------+
```

- The DH2I operator deploy a MSSQL cluster by creating a group of instance with an availability group. The operatore will also create 2 PVCs per instance.
- We create a **single** shared pvc backup mounted on each instance on the path /backup
- When Kasten Backup it does 2 operations 
   1. It creates a backup of the databases on the backup pvc 
   2. It back up all the pvc including the backup pvc 
- When Kasten Restore it does 2 operations 
   1. It restore all the PVCs including the backup pvc
   2. It restore all the database from the backup pvc


# Install the operator

The steps described in the [dh2i documentation](https://support.dh2i.com/dxoperator/guides/dxoperator-qsg/) :

```
wget https://dxoperator.dh2i.com/dxesqlag/files/v1-cu2.yaml
kubectl apply -f v1-cu2.yaml
```

# Create a  DxEnterpriseSqlAg mssql cluster with an availability group 

> **Notice**: all the sql statement should be followed by a GO statement to be executed in `sqlcmd` mode.

Create the namespace for your installation
```
kubectl create ns mssql
kubectl config set-context --current --namespace=mssql
```

Create license and sa secret replace the LICENSE with your license number.
```
kubectl create secret generic mssql --from-literal=MSSQL_SA_PASSWORD='MyP@SSw0rd1!'
kubectl create secret generic dxe --from-literal=DX_PASSKEY='MyP@SSw0rd1!' --from-literal=DX_LICENSE=WPFC-Z36O-BGSP-ERRH
```

Create a custom mssql.conf
```
cat <<EOF | kubectl create -f - 
apiVersion: v1
kind: ConfigMap
metadata:
    name: mssql-config
data: 
    mssql.conf: |
     [EULA]
     accepteula = Y

     [network]
     tcpport = 1433

     [sqlagent]
     enabled = true
EOF
```

Create a shared pvc for the backup, this pvc must be Read Write many and must be snapshotable which is the case of azure file csi. 
I you don't have such storageclass available you can use this [solution](https://github.com/kubernetes-csi/csi-driver-nfs) based on a nfs share. 
```
cat<<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup  
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  storageClassName: azurefile
EOF
```

Create the dx mssql cluster
```
cat<<EOF | kubectl create -f -
apiVersion: dh2i.com/v1
kind:  DxEnterpriseSqlAg
metadata:
  name: dxesqlag
spec:
  synchronousReplicas: 3
  asynchronousReplicas: 0
  # ConfigurationOnlyReplicas are only allowed with availabilityGroupClusterType set to EXTERNAL
  configurationOnlyReplicas: 0            
  availabilityGroupName: AG1
  # Listener port for the availability group (uncomment to apply)
  availabilityGroupListenerPort: 14033
  # For a contained availability group, add the option CONTAINED
  availabilityGroupOptions: null
  # Valid options are EXTERNAL (automatic failover) and NONE (no automatic failover)
  availabilityGroupClusterType: EXTERNAL
  createLoadBalancers: true
  template:
    metadata:
      labels:
        label: example
      annotations:
        annotation: example
    spec:
      dxEnterpriseContainer:
        image: "docker.io/dh2i/dxe:latest"
        imagePullPolicy: Always
        acceptEula: true
        clusterSecret: dxe
        vhostName: VHOST1
        joinExistingCluster: false
        # QoS – guaranteed (uncomment to apply)
        #resources:
          #limits:
            #memory: 1Gi
            #cpu: '1'
        # Configuration options for the required persistent volume claim for DxEnterprise
        volumeMounts:
        - mountPath: /backup
          name: backup
        volumeClaimConfiguration:
          storageClassName: managed-premium
          resources:
            requests:
              storage: 1Gi
      mssqlServerContainer:
        image: "mcr.microsoft.com/mssql/server:latest"
        imagePullPolicy: Always
        mssqlSecret: mssql
        acceptEula: true
        mssqlPID: Developer
        # Only set this to a value if you created a ConfigMap
        mssqlConfigMap: mssql-config
        # QoS – guaranteed (uncomment to apply)
        #resources:
          #limits:
            #memory: 2Gi
            #cpu: '2'
        # Configuration options for the required persistent volume claim for SQL Server
        volumeMounts:
        - mountPath: /backup
          name: backup
        volumeClaimConfiguration:
          storageClassName: managed-premium
          resources:
            requests:
              storage: 2Gi
      # Additional side-car containers, such as mssql-tools (uncomment to apply)
      containers:
      - name: mssql-tools
        image: "mcr.microsoft.com/mssql-tools"
        command: [ "/bin/sh" ]
        args: [ "-c", "tail -f /dev/null" ]
        volumeMounts:
        - mountPath: /backup
          name: backup
      volumes:
      - name: backup
        persistentVolumeClaim:
          claimName: backup
EOF
```

Create the load balancer
```
cat<<EOF | kubectl create -f -
apiVersion: v1
kind: Service
metadata:
  name: dxemssql-cluster-lb
spec:
  type: LoadBalancer
  selector:
    dh2i.com/entity: dxesqlag
  ports:
  - name: sql
    protocol: TCP
    port: 1433
    targetPort: 1433
  - name: listener
    protocol: TCP
    port: 14033
    targetPort: 14033
  - name: dxe
    protocol: TCP
    port: 7979
    targetPort: 7979
EOF
```

Connect to the database from the first pod
```
kubectl exec -it dxesqlag-0 -c mssql-tools -- /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'MyP@SSw0rd1!'
```

List the databases 
```
# list the databases
SELECT name FROM sys.databases;
```

Check availability group informations
```
# Basic Information about Availability Groups:
SELECT name, resource_id, resource_group_id, failure_condition_level, automated_backup_preference FROM sys.availability_groups;
# Information about Availability Replicas:
SELECT replica_server_name, availability_mode_desc, failover_mode_desc, primary_role_allow_connections_desc, secondary_role_allow_connections_desc FROM sys.availability_replicas;
# Information about Availability Databases 
SELECT database_name FROM sys.availability_databases_cluster;
# Information about Availability Group Listeners:
SELECT listener_id, dns_name, port  FROM sys.availability_group_listeners;
# find the current database
SELECT DB_NAME() AS CurrentDatabase;
# what are the availability group available 
SELECT name, automated_backup_preference_desc FROM sys.availability_groups;
```

You should see `AG1` which is consistent with the configuration of the DxEnterpriseSqlAg custom resource 
```
name  automated_backup_preference_desc                            
---------------------------------------
 AG1  secondary 
```


Now you can exit by simply typing exit 
```
exit
```

# Create an alias to connect to the listener through the client 

For simplicity let's create an alias that will connect to the listner through the first pod.

```
alias dx="kubectl exec dxesqlag-0 -c mssql-tools -it -- /opt/mssql-tools/bin/sqlcmd -S dxemssql-cluster-lb,14033 -U sa -P 'MyP@SSw0rd1\!'"
```

The only thing you have to do now is to simply type `dx`
```
dx
1>
```

# Let's install the AdventureWorks2022 database in the availability group

first find who's the primary, because adding a database to the availability group require to be on the primary.
```
select @@servername
```
The listener always connect to the primary, hence `@@servername` will give you the primary.

Suppose you find DXESQLAG-0, then the rest of the operations should be done on the corresponding pod dxesqlag-0 

Let's install the AdventureWorks2022 database by entering the mssql tool
```
kubectl exec -it dxesqlag-0 -c mssql-tools -- /bin/bash
```

Let's download the backup on the shared backup pvc : 
```
curl -L -o /backup/AdventureWorks2022.bak https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorks2022.bak
sqlcmd -S localhost -U sa -P 'MyP@SSw0rd1!'
```

Discover the logical file name of the database 
```
RESTORE FILELISTONLY FROM DISK = '/backup/AdventureWorks2022.bak';
```
The 2 logical files we find from the output is AdventureWorks2022 and AdventureWorks2022_log.

Now you can restore the database using this 2 virtual filename.
```
RESTORE DATABASE AdventureWorks2022 FROM DISK = '/backup/AdventureWorks2022.bak' WITH MOVE 'AdventureWorks2022' TO '/var/opt/mssql/data/AdventureWorks2022.mdf', MOVE 'AdventureWorks2022_log' TO '/var/opt/mssql/data/AdventureWorks2022.ldf', RECOVERY;
```

You should obtain an output like this one 
```
Processed 25376 pages for database 'AdventureWorks2022', file 'AdventureWorks2022' on file 1.
Processed 2 pages for database 'AdventureWorks2022', file 'AdventureWorks2022_log' on file 1.
RESTORE DATABASE successfully processed 25378 pages in 10.375 seconds (19.109 MB/sec).
```

You can list the tables of AdventureWorks 
```
USE AdventureWorks2022;
SELECT TABLE_SCHEMA, TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';
SELECT COUNT(*) FROM Person.Person;
```


In order to add the AdventureWorks2022 database in availabilty group you need to take a backup first so that the database switch if full recovery mode. 

```
BACKUP DATABASE AdventureWorks2022 TO DISK = '/backup/AdventureWorks2022.bak';
BACKUP LOG AdventureWorks2022 TO DISK = '/backup/AdventureWorks2022.trn';
```

Now we can add the database to the availability group 
```
use master;
ALTER AVAILABILITY GROUP AG1 ADD DATABASE AdventureWorks2022;
```

> **Note**: `use master;` is mandatory to add a database on the availability group.

Now you can check that the database has been replicated on every nodes.
Exit from the sqlcmd and the container and execute: 
```
kubectl exec dxesqlag-1 -c mssql -- ls /var/opt/mssql/data/AdventureWorks2022.mdf
kubectl exec dxesqlag-2 -c mssql -- ls /var/opt/mssql/data/AdventureWorks2022.mdf
```

# Test failover 

In order to test failover we create a client that connect with the availability listener and constantly execute 
```
SELECT COUNT(*) FROM Person.Person;
```

Let's create the client and exec bash on it 
```
kubectl run client --image mcr.microsoft.com/mssql-tools  -- tail -f /dev/null
kubectl exec client -it -- /bin/bash
```

Now you can test this command using the listener `dxemssql-cluster-lb,14033`
```
sqlcmd -S dxemssql-cluster-lb,14033 -U sa -P 'MyP@SSw0rd1!' -d AdventureWorks2022 -Q 'SELECT COUNT(*) FROM Person.Person;'
```

If this command is successful let's do it in a loop
```
while true
do
  sqlcmd -S dxemssql-cluster-lb,14033 -U sa -P 'MyP@SSw0rd1!' -d AdventureWorks2022 -Q 'SELECT COUNT(*) FROM Person.Person;'
  sleep 1
done
```  

and in another shell kill the first pod 
```
kubectl delete po dxesqlag-0
```

You'll see this ouput in the client pod
```
-----------
      19972

(1 rows affected)
           
-----------
      19972

(1 rows affected)




Sqlcmd: Error: Microsoft ODBC Driver 13 for SQL Server : Unable to access availability database 'AdventureWorks2022' because the database replica is not in the PRIMARY or SECONDARY role. Connections to an availability database is permitted only when the database replica is in the PRIMARY or SECONDARY role. Try the operation again later..
Sqlcmd: Error: Microsoft ODBC Driver 13 for SQL Server : TCP Provider: Error code 0x2749.
Sqlcmd: Error: Microsoft ODBC Driver 13 for SQL Server : A network-related or instance-specific error has occurred while establishing a connection to SQL Server. Server is not found or not accessible. Check if instance name is correct and if SQL Server is configured to allow remote connections. For more information see SQL Server Books Online..
Sqlcmd: Error: Microsoft ODBC Driver 13 for SQL Server : Unable to access availability database 'AdventureWorks2022' because the database replica is not in the PRIMARY or SECONDARY role. Connections to an availability database is permitted only when the database replica is in the PRIMARY or SECONDARY role. Try the operation again later..
           
-----------
      19972

(1 rows affected)

           
-----------
      19972

(1 rows affected)
```

As you can see the database was unavailable for just few seconds and then the fail over kicks and we were able to read acgain.

Let's reuse the find-primary.sql script to find out who's the primary.
```
kubectl exec dxesqlag-0 -c mssql-tools -it -- /opt/mssql-tools/bin/sqlcmd -S dxemssql-cluster-lb,14033 -U sa -P 'MyP@SSw0rd1!' -i /backup/script/find-primary.sql
```

You can read that DXESQLAG-1 is now the primary instead of DXESQLAG-0.

# Unsafe backup and restore

An Unsafe backup and restore consist of capturing the namespace that contains the database without any extended behaviour 
from Kasten (freeze/flush or logical dump) by just backing up using the standard Kasten workflow. Then restore it to see if database : 
1. Restarts and can accept new read/write connections 
2. Is in a state consistent with the state of the database at the backup but this is very difficult to check 

## Should I rely on unsafe backup and restore ?

Short answer : No !!

Long answer : Database are designed to restart after a power cut or a machine failure. Kasten take crash consistent backup hence when you 
restore your workload with Kasten most of the time they restarts. But database vendor recommand doing post an pre operation before taking a
backup for flushing the buffer on the disk and sometimes transaction are spanning on multiple machines and only vendor backup can capture 
a consistent state of the database. 

**With unsafe backup and restore your workload may restart but silent data loss can occur with no error message to let you know.**

## So what's the point with unsafe backup and restore ? 

If you don't have the time to implement a blueprint for your database, unsafe backup and restore is always better than nothing ... 
Actually it's far better than nothing. But this is not ideal.

Also having an immediate successful unsafe backup and restore is a good sign of the operator and database robustness. In this matter DH2I is very robust.

## Testing Unsafe backup and restore 

Let's add a table to the AdventureWorks2022 database. 
```
use AdventureWorks2022;
CREATE TABLE Sales.MyTable (ID INT IDENTITY(1,1) PRIMARY KEY, Name NVARCHAR(50),CreatedAt DATETIME DEFAULT GETDATE());
INSERT INTO Sales.MyTable (Name) VALUES ('John Doe');
select * from Sales.MyTable;
```

Now use Kasten and create a policy with an export for the namespace mssql. 

When the backup is finished delete namespace mssql
```
kubectl delete ns mssql
```

Then go to the remote restore point and simply click restore. 

The restore should be successful and you should be able to retreive the data you just created.


# Use a Kasten blueprint to take full and log backup

The unsafe backup and restore we did above worked well because the database was not under heavy use. But on production 
SQL Server keeps active transactions and uncommitted data in memory. A snapshot taken without coordinating 
with SQL Server might lead to an inconsistent state. We need to quiet the disk and depending of your volumes
and your storage technology this operation can be long.

To guarantee consistency of our backup we're going to use the `BACKUP DATABASE` command.

Also backing up the transaction log in SQL Server is crucial for ensuring point-in-time recovery.

1. The first backup will be created in the `/backup/current` directory with the name of the database for instance `/backup/current/AdventureWorks2022.bak`
2. The second backup will : 
    - move the `.bak` file in the `/backup/previous` directory for instnace `/backup/previous/AdventureWorks2022.bak`
    - remove the old `.trn` log file with the name of the database if it exists, for instance `/backup/current/AdventureWorks2022.trn`
    - create a new `.trn` log file with the name of the database, for instance `/backup/current/AdventureWorks2022.trn`
    - create a `.bak` backup file with the name of the database for instance `/backup/current/AdventureWorks2022.bak`
3. The next backup will repeat the steps in 2 if a `.bak` file already exists in the `/backup` directory

When you'll need to restore you will restore as in the basic backup then you can use the backup file to restore in a 
point in time. 

For instance let's imagine you have a 2 hours frequency backup we create this succeeding backups 
```
- 08:00 
  - /backup/current/AdventureWorks2022.bak  (08:00)
- 10:00
  - /backup/current/AdventureWorks2022.trn  (10:00)
  - /backup/previous/AdventureWorks2022.bak (08:00)
  - /backup/current/AdventureWorks2022.bak  (10:00)
- 12:00
  - /backup/current/AdventureWorks2022.trn  (12:00)
  - /backup/previous/AdventureWorks2022.bak (10:00)
  - /backup/current/AdventureWorks2022.bak  (12:00)
- 14:00 
  - /backup/current/AdventureWorks2022.trn  (14:00)
  - /backup/previous/AdventureWorks2022.bak (12:00)
  - /backup/current/AdventureWorks2022.bak  (14:00)
```

For restoring the database at 12:00 you restore the 12:00 kasten `restorepoint` 
```
use master;
ALTER AVAILABILITY GROUP AG1 REMOVE DATABASE AdventureWorks2022;
RESTORE DATABASE AdventureWorks2022 FROM DISK = '/backup/current/AdventureWorks2022.bak' WITH REPLACE;
ALTER AVAILABILITY GROUP AG1 ADD DATABASE AdventureWorks2022;
```
This will be the default behaviour of the blueprint so you won't need to do it manually. 

For restoring a database at 11:32 you also restore the 12:00 kasten `restorepoint` but now you execute 
```
ALTER AVAILABILITY GROUP AG1 REMOVE DATABASE AdventureWorks2022;
RESTORE DATABASE AdventureWorks2022 FROM DISK = '/backup/previous/AdventureWorks2022.bak' WITH NORECOVERY, REPLACE;
RESTORE LOG AdventureWorks2022 FROM DISK = '/backup/current/AdventureWorks2022.trn' WITH NORECOVERY, STOPAT = '2025-01-05T20:32:00';
RESTORE DATABASE AdventureWorks2022 WITH RECOVERY;
ALTER AVAILABILITY GROUP AG1 ADD DATABASE AdventureWorks2022;
```

We did not automate this process in the blueprint because all the possible use cases can make the code of the blueprint too complex. 
Hence this operation must be done manually.

# Create the blueprint

## Important 

We provide an example blueprint implementing the strategy described above but the customer or the customer partner must 
adapt this strategy to its unique use case. It is very important to run extensive test of backup and restore before moving to production.

## Implementation 

The blueprint implement the stategy described in more details above. 

## Install

```
kubectl create -f mssql-bp.yaml
kubectl create -f mssql-bp-binding.yaml
```

The binding ensure that any time Kasten will backup a DxEnterpriseSqlAg object in the namespace it will apply the blueprint.

## test PIT (Point In Time) restore

### Create a client that insert data every 10 seconds 
For testing we are going to insert every 10 seconds a new entry in the `Sales.MyTable` table

Create the pod inserter 
```
kubectl create -f insterter.yaml 
```

if you logs the inserter pod 
```
kubectl logs inserter 
```

You should have an ouput like this one
```
+ true
+ /opt/mssql-tools/bin/sqlcmd -S dxemssql-cluster-lb,14033 -U sa -P 'MyP@SSw0rd1!' -d AdventureWorks2022 -Q 'INSERT INTO Sales.MyTable (Name) VALUES ('\''John Doe'\'');'

(1 rows affected)
+ sleep 1
+ true
+ /opt/mssql-tools/bin/sqlcmd -S dxemssql-cluster-lb,14033 -U sa -P 'MyP@SSw0rd1!' -d AdventureWorks2022 -Q 'INSERT INTO Sales.MyTable (Name) VALUES ('\''John Doe'\'');'

(1 rows affected)
+ sleep 1
+ true
+ /opt/mssql-tools/bin/sqlcmd -S dxemssql-cluster-lb,14033 -U sa -P 'MyP@SSw0rd1!' -d AdventureWorks2022 -Q 'INSERT INTO Sales.MyTable (Name) VALUES ('\''John Doe'\'');'
```

### Execute a PIT restore 

Ensure that you have an hourly policy and that 2 subsequent backup run successfully.

1. delete the namespace 
```
kubectl delete ns mssql 
```

2. restore the last remote hourly backup but exclude the inserter pod to not create extra data at restore.
Check the last entries in the Sales.MyTables with the `dx` alias described above
```
use AdventureWorks2022;
select * from Sales.MyTable;
```
The date of the last entries should match the date of your restorepoint.

3. enter the mssql-tools container with the `dx` alias command and execute the restore of the previous backup 
```
RESTORE DATABASE AdventureWorks2022 FROM DISK = '/backup/previous/AdventureWorks2022.bak' WITH NORECOVERY;
```

4. restore the PIT 
Adapt the date to your situation
```
RESTORE LOG AdventureWorks2022 FROM DISK = '/backup/current/AdventureWorks2022.trn' WITH RECOVERY, STOPAT = '2025-12-03T11:32:00';
```

5. Control the last entries in the  table Sales.MyTable
```
use AdventureWorks2022;
select * from Sales.MyTable;
```
And make sure they are consistent with your `STOPAT ` value.





Msg 3059, Level 16, State 2, Server dxesqlag-0, Line 1
This BACKUP or RESTORE command is not supported on a database mirror or secondary replica.
Msg 3013, Level 16, State 1, Server dxesqlag-0, Line 1
RESTORE DATABASE is terminating abnormally.


