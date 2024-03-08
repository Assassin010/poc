Architecture
The EC2 instance is our source server, from which we will get all the required files from the internet and send them to S3. Github Actions will initiate the connection to the server, and from the server, we will download the files. Since we have a datasync agent installed on the server (registered) with AWS DataSync, the system will sync all of the files at a specific point and send them to S3 using AWS DataSync. 

To reduce data transmission costs and keep data in the same Availability Zone, we'll place the DataSync agent and VPC endpoint on the same subnet as the source server.


Prerequesites:


DataSync agent
An EC2 instance with the latest DataSync AMI. https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html



VPC endpoint
Used for control plane communication between the agent and the DataSync service. The VPC endpoint allows traffic to remain on the Amazon network, thus the agent does not require access to the public internet or a NAT gateway. Read more about utilizing DataSync with VPC endpoints here: https://docs.aws.amazon.com/datasync/latest/userguide/datasync-in-vpc.html



Three EC2 security groups are required:

1. DataSync Agent: This needs to allow traffic on port 80 from your machine for the purpose of automatic agent activation. This rule can be removed after the activation process is completed.

2. VPC Endpoint and DataSync Task Elastic Network Interfaces (ENIs): it's ecessary to allow traffic on ports 1024–1064 for control plane operations and port 443 for data transfer and agent activation/creation within the DataSync service. This security group is used by DataSync for both the VPC endpoint and the task ENIs. Tests have shown that without port 443 open, the VPC endpoint fails to register the agent. Moreover, port 443 is necessary for data transfer as this security group is used for the VPC endpoint and task ENIs.

3. Source Server: This needs to allow TCP/UDP traffic on port 2049 from the DataSync agent. This should be attached to the source server post-deployment.



NFS Server
The next step is to turn the source server into an NFS server. NFS is a distributed file system protocol that allows clients to access files over a network. In our case, the DataSync agent acts as the client and facilitates transfer between the source server and the DataSync service. To enable NFS on the source server, first ensure nfs-utils is installed:


sudo yum -y install nfs-utils


Add the line below to /etc/exports (you likely need to create the file). This file serves as configuration for the NFS server, and indicates the directories that are shared with NFS clients as well as allowed IP addresses. Change the directory to your source directory, and the IP address range to your VPC CIDR (or the IP of your DataSync agent):


/home/ec2-user/datasync/  172.30.0.0/16(rw,sync,no_root_squash)


It may be necessary to update file permissions for the agent:
chmod -R 755 /home/ec2-user/datasync/

Finally, start the NFS server:
sudo systemctl start nfs-server

You can read more about DataSync and NFS here:
https://docs.aws.amazon.com/datasync/latest/userguide/create-nfs-location.html


DataSync Agent
The next step is to create and activate the agent in the DataSync service (we’ve already deployed the agent on EC2, but we need to register it with the service). Navigate to DataSync in the AWS Management Console. Choose the values below, specifying the DataSyncServiceSG security group created by the CloudFormation template:



Performance
Transfer performance is impacted by the EC2 instance type and EBS volume type of the source server, as well as the EC2 instance type of the DataSync agent. AWS recommends an agent instance size of m5.2xlarge for transfers of up to 20 million files. If your transfer speeds are slow, try increasing the instance size of your source server and agent, as network and EBS bandwidth generally increase with larger instance classes. I ran various tests below (I would expect more variance with a larger number of files).

Changing EBS volume type of source server
Both source server and agent were m5.large