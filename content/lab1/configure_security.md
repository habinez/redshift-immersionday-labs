---
title: "Configure Security"
date: 2019-12-15T15:39:05-05:00
draft: true
weight: 6
---


## Configure Security
### VPC
Create or identify a VPC where you will launch your Redshift cluster.  For our purposes we will create a **VPC** to isolate the traffic.
```
https://console.aws.amazon.com/vpc/home?#CreateVpc:
```
![](/images/VPC.png)

### InternetGateway
Create or identify an Internet Gateway.  For our purposes we will create an **Internet Gateway**. Once created, select the Internet Gateway and attach it to the VPC created earlier.  
```
https://console.aws.amazon.com/vpc/home?#Create%20Internet%20Gateway:
```
![](/images/InternetGateway.png)
![](/images/InternetGatewayAttach1.png)
![](/images/InternetGatewayAttach2.png)

### Subnets
Create or identify a subnet with a default route to an Internet Gateway.  For our purposes we will create a **Subnet** attached to the previously created VPC in two different Availability Zones to improve fault tolerance.
```
https://console.aws.amazon.com/vpc/home?#CreateSubnet:
```
![](/images/Subnet1.png)
![](/images/Subnet2.png)
### Route Table
If you have created new subnets, you must create a **Route Table** with the default route pointed to the internet gateway and with the new subnets added.
```
https://console.aws.amazon.com/vpc/home?#CreateRouteTable:
```
![](/images/Route.png)
![](/images/EditRoute.png)
![](/images/EditSubnet.png)
### Subnet Group
Create a Redshift **Cluster Subnet Group** containing the two subnets you created earlier.
```
https://console.aws.amazon.com/redshift/home?#subnet-groups:
```
![](/images/SubnetGroup.png)
### Security Group
Create a **Security Group** associated to the VPC you created earlier.  Edit the Security Group to create a rule which allows incoming connections from your IP Address.
```
https://console.aws.amazon.com/vpc/home#SecurityGroups:sort=groupId
```
![](/images/SecurityGroup.png)
### S3 Access
Create an **IAM Role** with the type "Redshift" and the use-case of "Redshift - Customizable" and attach the AmazonS3ReadOnlyAccess and AWSGlueConsoleFullAccess policies to the role.
```
https://console.aws.amazon.com/iam/home?#/roles$new?step=type
```
![](/images/Role.png)
