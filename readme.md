# Verified data services 

## Goal 

Verified data services is a repository of architecture reference for having Kasten By Veeam and the main data services operator work together.

## Motivation 

Moving databases to kubernetes has a lot of benefits :
- Colocalisation and security 
- Agility 
- Ease of deployment
- Self healing
- Cost
- Microservices architecture
- Scalability
- Resiliency


With the operator pattern provided by kubernetes it is now possible to deploy High Availability data services with enterprise requirement very easily. 

Nearly all major databases vendor or database specialists provide an operator that capture their expertise.

We recommand using operators for your data services: 
- You'll benefit high availability and high security very easily. 
- You can upgrade to a vendor supported version even if in the first place you use the community or developper edition.

## Supported operators 

We provide an architecture reference for the Kasten integration with those data services/vendor :

- [MSSQL/DH2I](./dh2i/)
- Elasticsearch
- Mongo/Mongo-Enterprise 
- Mongo/Percona
- Postgres/EDB
- Postgres/Crunchy  
- Mysql/Oracle
- Mysql/Percona 
- Kafka/Strimzi
- Kafka/Confluence

## Key features covered 

In each architecture reference we cover those aspects :

### Operator deployment 
We define how the operator should be deployed in order to work with Kasten by Veeam. We specify the version of the operator and the version of the database.

### Operator validation 
Depending of the collaboration we developped with the operator we may ask the vendor to validate our architecture. 

### High Availability 
We always deploy the data service in High Availability mode and made sure that Kasten works well in this conditions. We Provide scenario to test failover.

### Unsafe backup & restore

An Unsafe backup and restore consist of capturing the namespace that contains the database without any extended behaviour 
from Kasten (freeze/flush or logical dump) by just backing up using the standard Kasten workflow. Then restore it to see if database : 
1. Restarts and can accept new read/write connections 
2. Is in a state consistent with the state of the database at the backup but this is very difficult to check 

Database are designed to restart after a power cut or a machine failure. Kasten take crash consistent backup hence when you 
restore your workload with Kasten most of the time they restarts. But database vendor recommand doing post an pre operation before taking a
backup for flushing the buffer on the disk and sometimes transaction are spanning on multiple machines and only vendor backup can capture 
a consistent state of the database. 

**With unsafe backup and restore your workload may restart but silent data loss can occur with no error message to let you know.**

If you don't have the time to implement a blueprint for your database, unsafe backup and restore is always better than nothing ... 
Actually it's far better than nothing. But this is not ideal and Kasten can solve this problem with Blueprint. 

### Blueprint example 

In order to do safe backup and restore of your database Kasten allow you to extends the backup/restore workflow with Blueprint. 

Blueprint capture data-service expertise in a sequence of function that you'll reapply to all your data-service in your cluster with blueprint binding.

Architecture reference always comes with a blueprint example.

### Performance consideration 

As much as possible we always do the backup on a replica to limit the impact of the backup for the rest of the applications.

### Point In Time (PIT) support 

If backup with PIT Restore is possible we make the maximum to enable it.


# This repository is not the Kanister example repository 

Blueprint are provided by the [Kanister project](https://docs.kanister.io/overview.html). Kanister provide [a great example repository](https://github.com/kanisterio/kanister/tree/master/examples) for all kind of data services.

However the verified data service repository leverage Kanister but mainly leverage Kasten and for this reason provide a lot of feature that you won't find with Kanister : 

- Autodiscovery and Capture of the metadata 
- Autodiscovery and Capture of the data with the Kasten data mover
- Migration 
- Automatic Blueprint Binding 
- Disaster recovery 
- Immutability 
- Fast restore from snapshots 
- GUI with authentication and authorization

And many others but that's the most important. 










