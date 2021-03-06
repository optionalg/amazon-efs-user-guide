# Security<a name="security-considerations"></a>

Following, you can find a description of security considerations for working with Amazon EFS\. There are four levels of access control to consider for Amazon EFS file systems, with different mechanisms used for each\.


+ [AWS Identity and Access Management \(IAM\) Permissions for API Calls](#iam-permissions-for-api-calls)
+ [Security Groups for Amazon EC2 Instances and Mount Targets](#network-access)
+ [Read, Write, and Execute Permissions for EFS Files and Directories](#user-and-group-permissions)
+ [Encrypting Data and Metadata at Rest in EFS](encryption.md)

## AWS Identity and Access Management \(IAM\) Permissions for API Calls<a name="iam-permissions-for-api-calls"></a>

You create, manage, and delete file systems with calls to the Amazon EFS API\. If the caller is using credentials for an AWS Identity and Access Management \(IAM\) user or assumed role, each API call requires that the caller have permissions for the action being called in its IAM policy\. Some API actions support policy permissions specific to the file system that is the object of the call \(that is, resource\-level permissions\)\. API calls made with an account's root credentials have permissions for all API actions on file systems owned by the account\. 

As an example of IAM permissions, IAM user Alice might have permissions to retrieve descriptions of all file systems in her parent AWS account\. However, she might be allowed to manage the security groups for only one of them, file system ID `fs-12345678`\.

For more information about IAM permissions with the Amazon EFS API, see [Authentication and Access Control for Amazon EFS](auth-and-access-control.md)\.

## Security Groups for Amazon EC2 Instances and Mount Targets<a name="network-access"></a>

When using Amazon EFS, you specify Amazon EC2 security groups for your EC2 instances and security groups for the EFS mount targets associated with the file system\. Security groups act as a firewall, and the rules you add define the traffic flow\. In the Getting Started exercise, you created one security group when you launched the EC2 instance\. You then associated another with the EFS mount target \(that is, the default security group for your default VPC\)\. That approach works for the Getting Started exercise\. However, for a production system, you should set up security groups with minimal permissions for use with EFS\.

You can authorize inbound and outbound access to your EFS file system\. To do so, you add rules that allow your EC2 instance to connect to your Amazon EFS file system through the mount target using the Network File System \(NFS\) port\. Take the following steps to create and update your security groups\.

**To create security groups for EC2 instances and mount targets**

1. Create two security groups in your VPC\.

   For instructions, see the procedure "To create a security group" in [Creating a Security Group](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html#CreatingSecurityGroups) in the *Amazon VPC User Guide*\. 

1. Open the Amazon VPC Management Console at [https://console\.aws\.amazon\.com/vpc/](https://console.aws.amazon.com/vpc/), and verify the default rules for these security groups\. Both security groups should have only an outbound rule that allows traffic to leave\. 

**To update the necessary access for your security groups**

1. Open the Amazon VPC console at [https://console\.aws\.amazon\.com/vpc/](https://console.aws.amazon.com/vpc/)\.

1. Add a rule for your EC2 security group to allow inbound access using Secure Shell \(SSH\) from any host\. Optionally, restrict the **Source** address\. 

   You don't need to add an outbound rule, because the default outbound rule allows all traffic to leave\. If this were not the case, you'd need to add an outbound rule to open the TCP connection on the NFS port, identifying the mount target security group as the destination\.  
![\[Image NOT FOUND\]](http://docs.aws.amazon.com/efs/latest/ug/images/gs-ec2-resources-100.png)

   For instructions, see [Adding and Removing Rules](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html#AddRemoveRules) in the *Amazon VPC User Guide*\. 

1. Add a rule for the mount target security group to allow inbound access from the EC2 security group as shown following\. The EC2 security group is identified as the source\.  
![\[Image NOT FOUND\]](http://docs.aws.amazon.com/efs/latest/ug/images/gs-ec2-resources-120.png)

1. Verify that both security groups now authorize inbound and outbound access\.

For more information about security groups, see [Security Groups for EC2\-VPC](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html#vpc-security-groups) in the *Amazon EC2 User Guide for Linux Instances*\. 

### Security Considerations for Mounting an Amazon EFS File System<a name="sg-information"></a>

An NFSv4\.1 client can only mount a file system if it can make a network connection to the NFS port of one of the file system's mount targets\. Similarly, an NFSv4\.1 client can only assert a user and group ID when accessing a file system if it can make this network connection\. 

The ability to make this network connection is governed by a combination of the following:

+ **Network isolation provided by the mount targets' VPC** – File system mount targets can't have public IP addresses associated with them\. Only Amazon EC2 instances in the Amazon VPC or on\-premises servers connected to the Amazon VPC by using AWS Direct Connect can mount Amazon EFS file systems\. 

  You can't currently use other mechanisms for connecting to a VPC's private IP addresses from outside the VPC to mount Amazon EFS file systems\. For example, you can't use VPN connections or VPC peering to do so\. Don't rely on such other methods for file system access control\.

+ **Network access control lists \(ACLs\) for the VPC subnets of the client and mount targets, for access from outside the mount target's subnets** – To mount a file system, the client must be able to make a TCP connection to the NFS port of a mount target and receive return traffic\. 

+ **Rules of the client's and mount targets' VPC security groups, for all access** – For an EC2 instance to mount a file system, the following security group rules must be in effect: 

  +  The file system must have a mount target whose network interface has a security group with a rule that enables inbound connections on the NFS port from the instance\. You can enable inbound connections either by IP address \(CIDR range\) or security group\. The source of the security group rules for the inbound NFS port on mount target network interfaces is a key element of file system access control\. Inbound rules other than the one for the NFS port, and any outbound rules, aren't used by network interfaces for file system mount targets\. 

  +  The mounting instance must have a network interface with a security group rule that enables outbound connections to the NFS port on one of the file system's mount targets\. You can enable outbound connections either by IP address \(CIDR range\) or security group\.

For more information, see [Creating Mount Targets](accessing-fs.md)\.

## Read, Write, and Execute Permissions for EFS Files and Directories<a name="user-and-group-permissions"></a>

Files and directories in an EFS file system support standard Unix\-style read, write, and execute permissions based on the user and group ID asserted by the mounting NFSv4\.1 client\.  For more information, see [Network File System \(NFS\)–Level Users, Groups, and Permissions](accessing-fs-nfs-permissions.md)\.

**Note**  
This layer of access control depends on trusting the NFSv4\.1 client in its assertion of the user and group ID\. There is no authentication of the identity of the NFSv4\.1 client when establishing a mount connection\. Thus, any NFSv4\.1 client that can make a network connection to the NFS port of a file system's mount target IP address can read and write the file system as the root user ID\. 

As an example of read, write, and execute permissions for files and directories, Alice might have permissions to read and write to any files that she wants to in her personal directory on a file system, /alice\. However, in this example Alice is not allowed to read or write to any files in Mark's personal directory on the same file system, /mark\. Both Alice and Mark are allowed to read but not write files in the shared directory /share\. 