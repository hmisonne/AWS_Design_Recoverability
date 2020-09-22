# Data durability and recovery

This project is part of the Udacity Cloud Architect Nanodegree.

The goal of this project is to :
- Build a Multi-AvailabilityZone, Multi-Region database and show how to use it in multiple geographically separate AWS regions.  
- Build a website hosting solution that is versioned so that any data destruction and accidents can be quickly and easily undone.

Here is a visualisation of the Cloud Architecture I set up for this project using [Lucid Chart](https://www.lucidchart.com):
![AWS Diagram](screenshots/diagram.png "AWS Diagram")

The instructions for this project are available [here](https://github.com/udacity/nd063-c2-design-for-availability-resilience-reliability-replacement-project-starter-template).

## Project Setup
### Cloud formation
In this project, I used the AWS CloudFormation to create Virtual Private Clouds. CloudFormation is an AWS service that allows you to create "infrastructure as code". 

A configuration file written in a YAML file was provided to automate the creation of the VPCs with public and private subnets and Security Groups. The YAML file is available in this GitHub repo: https://github.com/udacity/nd063-c2-design-for-availability-resilience-reliability-replacement-project-starter-template/blob/master/cloudformation/vpc.yaml

### Part 1

### Data durability and recovery
In order to achieve the highest levels of durability and availability in AWS, I deployed a Cloud Formation in 2 AWS regions: An active region and a standby region with different CIDR address ranges for the VPCs.

**Primary VPC:**
![Primary VPC](screenshots/primary_Vpc.png "Primary VPC")

**Secondary VPC:**
![Secondary VPC](screenshots/secondary_Vpc.png "Secondary VPC")


### Highly durable RDS Database

To secure the access of the database, I created a new RDS **private Subnet group** in the active and standby region. This means that traffic coming directly from the Internet will not be allowed. Only traffic coming from the VPC.

**Subnet groups in the active region:**
![Primary DB subnetgroup](screenshots/primaryDB_subnetgroup.png "Primary DB subnetgroup")

**Subnet groups in the secondary region:**
![Secondary DB subnetgroup](screenshots/secondaryDB_subnetgroup.png "Secondary DB subnetgroup")

As shown below, public subnets allow traffic incoming from the internet (while private subnet only allows incoming traffic from the VPC)

**Route tables in subnet of the active region:**
![Primary subnet routing](screenshots/primary_subnet_routing.png "Primary subnet routing")

**Route tables in subnet of the secondary region:**
![Secondary subnet routing](screenshots/secondary_subnet_routing.png "Secondary subnet routing")

I created a new MySQL with the following parameters:
- Multi-AZ database to ensure a smooth failover in case of a single AZ outage
- Have only the “UDARR-Database” **security group**. This security group was defined in the Cloud Formation to only allow traffic from the EC2 instance (not yet created). More specifically, this “UDARR-Database” security group allows traffic from "UDARR-Application" security group inside the VPC using this specific port: 3306 (mysql port).
- Have an initial database called “udacity.” 

**Configuration of the database in the active region:**
![Primary DB config](screenshots/primaryDB_config.png "Primary DB config")

After the primary database has been set up, I created a read replica database in the standby region. This database has the same requirements as the database in the active region.

**Configuration of the database in the secondary region:**
![Secondary DB config](screenshots/secondaryDB_config.png "Secondary DB config")


### Estimate availability of this configuration

Description of Achievable Recovery Time Objective (RTO) and Recovery Point Objective (RPO) for this Multi-AZ, multi-region database in terms of:

1. Minimum RTO for a single AZ outage 
If a multi-AZ configuration is set up, the fail over to another AZ will happen automatically which can take a few minutes.
    
2. Minimum RTO for a single region outage
- 00:00 - Problem happens (0 minutes) 
- 00:05 - An amount of time passes before an alert triggers (5 minutes) 
- 00:06 - Alert triggers on-all staff (1 minute) 
- 00:16 - On-call staff may need to get out of bed, get to computer, log in, log onto VPN (10 minutes) 
- 00:26 - On-call staff starts diagnosing issue (10 minutes) 
- 00:41 - Root cause is discovered (15 minutes) 
- 00:46 - Remediation started (5 minutes) :  Promote secondary instance to be the new master and then route the traffic to the new endpoint
- 00:56 - Remediation completed (10 minutes) 
Total time: 56 minutes 

3. Minimum RPO for a single AZ outage
As it only takes a few minutes to fail over to another AZ, a few minutes of data will be lost.   
       
4. Minimum RPO for a single region outage 
If we set up an RDS database with automatic backups enabled, the RPO will be based on how often data is backed up. If we set up a backup every 4 hours, the minimun RPO will be 4 hours.


### Demonstrate normal usage

In the active region:
I created an EC2 keypair and launched an Amazon Linux EC2 instance in the active region with the following configuration:
- VPC's public subnet
- Security group ("UDARR-Application") that allows incoming traffic (SSH) from the Internet.

I established a SSH connection to the instance by running this command:
```
ssh -i {absolute/path/to/PrimaryKeyPair.pem} {EC2 identifiers provided on the EC2 console while clicking on connect}
```
After being connected and having mysql installed, I connected to my database by running this command:
```
mysql -u admin -p -h {PRIMARY_DATABASE_ENDPOINT}
```

Log of connecting to the database, creating the table, writing to and reading from the table:
![Log Primary](screenshots/log_primary.png "Log Primary")


### Monitor database

**DB Connections:**
![Monitoring connection](screenshots/monitoring_connections.png "Monitoring connection")

**DB Replication:**
![Monitoring replication](screenshots/monitoring_replication.png "Monitoring replication")

### Part 2
### Failover And Recovery
In the standby region:

I created an EC2 keypair in the region and launched an Amazon Linux EC2 instance in the standby region with the same configuration as before.
Since the database in the standby region is a read replica, we can only read from the database but we cannot insert data.

Database configuration **before the database promotion:**
![DB before promotion](screenshots/rr_before_promotion.png "DB before promotion")

Log of connecting to the database, writing to and reading from the database **before the database promotion:**
![Log before promotion](screenshots/log_rr_before_promotion.png "Log before promotion")

After promoting the read replica, we can now insert data into and read from the read replica database.

Database configuration **after the database promotion:**
![DB after promotion](screenshots/rr_after_promotion.png "DB after promotion")

Log of connecting to the database, writing to and reading from the database **after the database promotion:**
![Log after promotion](screenshots/log_rr_after_promotion.png "Log after promotion")


### Part 3
### Website Resiliency

Build a resilient static web hosting solution in AWS. Create a versioned S3 bucket and configure it as a static website.

1. Enter “index.html” for both Index document and Error document
2. Upload the files from the GitHub repo (under `/project/s3/`)
3. Paste URL into a web browser to see your website. 


**Original webpage:**
![Original Webpage](screenshots/s3_original.png "Original Webpage")

You will now “accidentally” change the contents of the website such that it is no longer serving the correct content

1. Change `index.html` to refer to a different “season”
2. Re-upload `index.html`
3. Refresh web page

**Modified webpage:**
![Accidental Change Webpage](screenshots/s3_season.png "Accidental Change Webpage")

You will now need to “recover” the website by rolling the content back to a previous version.

1. Recover the `index.html` object back to the original version
2. Refresh web page

**Modified webpage (reverted):**
![Reverted Change Webpage](screenshots/s3_season_revert.png "Reverted Change Webpage")

You will now “accidentally” delete contents from the S3 bucket. Delete “winter.jpg”

**Webpage after content deleted:**
![Deleted Content Webpage](screenshots/s3_deletion.png "Deleted Content Webpage")

**Existing versions of the file showing the "Deletion marker".**
![Delete Marker](screenshots/s3_delete_market.png "Delete Marker")

You will now need to “recover” the object:

1. Recover the deleted object
2. Refresh web page

**Recovered Website:**
![Recovered Website](screenshots/s3_delete_revert.png "Recovered Website")

## License
