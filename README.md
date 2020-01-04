# Creating an EC2 Instance in a VPC with the AWS-CLI
These instructions were published at vaneyckt https://coderwall.com/p/ndm54w/creating-an-ec2-instance-in-a-vpc-with-the-aws-command-line-interface

###  Creating the VPC
Start by installing the AWS Command Line Interface on your machine if you haven't done so already. \
With this done, we can now create our VPC.
```
$ vpcId=`aws ec2 create-vpc --cidr-block 10.0.0.0/28 --query 'Vpc.VpcId' --output text`
```


### Adding an Internet Gateway
Next we need to connect our VPC to the rest of the internet by attaching an internet gateway. Our VPC would be isolated from the internet without this.
```
$ aws ec2 modify-vpc-attribute --vpc-id $vpcId --enable-dns-support "{\"Value\":true}"
$ aws ec2 modify-vpc-attribute --vpc-id $vpcId --enable-dns-hostnames "{\"Value\":true}"
$ internetGatewayId=`aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text`
$ aws ec2 attach-internet-gateway --internet-gateway-id $internetGatewayId --vpc-id $vpcId
```


### Creating a Subnet
A VPC can have multiple subnets. Since our use case only requires one, we can reuse the cidr-block specified during VPC creation so as to get a single subnet that spans the entire VPC address space.
```
$ subnetId=`aws ec2 create-subnet --vpc-id $vpcId --cidr-block 10.0.0.0/28 --query 'Subnet.SubnetId' --output text`
```


### Configuring the Route Table
Each subnet needs to have a route table associated with it to specify the routing of its outbound traffic. By default every subnet inherits the default VPC route table which allows for intra-VPC communication only. \
The following adds a route table to our subnet that allows traffic not meant for an instance inside the VPC to be routed to the internet through our earlier created internet gateway.
```
$ routeTableId=`aws ec2 create-route-table --vpc-id $vpcId --query 'RouteTable.RouteTableId' --output text`
$ aws ec2 associate-route-table --route-table-id $routeTableId --subnet-id $subnetId
$ aws ec2 create-route --route-table-id $routeTableId --destination-cidr-block 0.0.0.0/0 --gateway-id $internetGatewayId
```

### Adding a Security Group
Before we can launch an instance, we need to create a security group that specifies which ports should allow traffic. For now we'll just allow any IP to try and make an SSH connection by opening port 22 to any IP.
```
$ securityGroupId=`aws ec2 create-security-group --group-name my-security-group --description "my-security-group" --vpc-id $vpcId --query 'GroupId' --output text`
$ aws ec2 authorize-security-group-ingress --group-id $securityGroupId --protocol tcp --port 22 --cidr 0.0.0.0/0
```
### Launching your Instance
All that's left to do is to create an SSH key pair and then launch an instance secured by this. Let's generate this key pair and store it locally with the correct permissions.
```
$ aws ec2 create-key-pair --key-name my-key --query 'KeyMaterial' --output text > ~/.ssh/my-key.pem
$ chmod 400 ~/.ssh/my-key.pem
```
### Launching a single t2.micro instance based on the public AWS Ubuntu image can now be done as follows.
```
$ instanceId=`aws ec2 run-instances --image-id ami-8c122be9 --count 1 --instance-type t2.micro --key-name my-key --security-group-ids $securityGroupId --subnet-id $subnetId --associate-public-ip-address --query 'Instances[0].InstanceId' --output text`
```
### Log into the running instance
```
$ instanceUrl=`aws ec2 describe-instances --instance-ids $instanceId --query 'Reservations[0].Instances[0].PublicDnsName' --output text`
$ ssh -i ~/.ssh/my-key.pen ec2-user@$instanceUrl
```
### List all active instances in the VPC
```
$ aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId]' --filters Name=instance-state-name,Values=running --output text
```

```
$ aws ec2 describe-instances --instance-ids $instanceId --output text | grep -w STATE | awk '{print $3}'
$ aws ec2 start-instances --instance-ids $instanceId --output text | grep -w CURRENTSTATE | awk '{print $3}'
$ aws ec2 stop-instances --instance-ids $instanceId --output text | grep -w CURRENTSTATE | awk '{print $3}'
$ aws ec2 terminate-instances --instance-ids $instanceId --output text | grep -w CURRENTSTATE | awk '{print $3}'
```
