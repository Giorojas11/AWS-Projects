# AWS Networking Project using Terraform

### This project demonstrates building a secure, multi-VPC AWS environment using Terraform. The network includes public and private subnets, EC2 instances, NAT gateways, route tables, security groups, NACLs, VPC peering, S3 connectivity, and CloudWatch monitoring.
------------------------------------------
## Table of Contents
1. Project Overview
2. VPC Creation
3. Subnet
   - Public Subnet & Internet Gateway
   - Private Subnet & NAT
4. Routing
5. Network Access Control Lists
6. EC2 Instances
   - Public EC2
   - Private EC2
7. Security Groups
8. VPC Peering
9. S3 Access & VPC Endpoints
10. Monitoring with CloudWatch
11. Terraform Project Structure
12. Lessons Learned & Next Steps

## 1. Project Overview
This project sets up a secure, multi-tier AWS network environment:
- Two VPCs with public and private subnets
- EC2 instances for public-facing and private servers
- NAT Gateway for private subnet outbound internet access
- VPC peering to enable private communication between VPCs
- S3 bucket connectivity with private VPC endpoint
- Network monitoring using CloudWatch Flow Logs
All infrastructure is managed via Terraform, ensuring reproducibility and infrastructure-as-code best practices.

## 2. VPC Creation
A Virtual Private Cloud (VPC) is an isolated section of AWS that keeps resources private and secure.
- Main VPC CIDR: 10.0.0.0/16 (10.0.0.0 - 10.0.255.255)
- Internet Gateway: 10.0.0.1

```
resource "aws_vpc" "main_vpc" {
    cidr_block = var.vpc_cidr_block

    enable_dns_support   = true
    enable_dns_hostnames = true
    
    tags = {
        Name = "Main VPC"
    }
}
```
## 3. Subnets
Subnetting creates smaller networks within my VPC. I created a public subnet and private subnet for different use cases.

### Public Subnet & Internet Gateway
Public subnets host resources that need internet access, like web servers.
- CIDR: 10.0.0.0/24 (10.0.0.0 - 10.0.0.255)
- Public IP assignment: Enabled (map_public_ip_on_launch = true)

Key points:
- Internet Gateway required for inbound/outbound internet access
- Route table: Traffic to local VPC, then 0.0.0.0/0
```
resource "aws_subnet" "public_subnet" {
    vpc_id                   = aws_vpc.main_vpc.id
    cidr_block               = "10.0.0.0/24"
    map_public_ip_on_launch  = true
    availability_zone        = "us-east-2a"

    tags = {
        Name = "Public Subnet"
    }
}

resource "aws_internet_gateway" "IGW" {
    vpc_id = aws_vpc.main_vpc.id

    tags = {
        Name = "IGW"
    }
}
```
### Private Subnet & NAT
Private subnets host sensitive or backend resources without direct internet access.
- CIDR: 10.0.1.0/24 (10.0.1.0 - 10.0.1.255)
- Public IP assignment: Disabled

Key Points:
- NAT Gateway provides internet access and blocks inbound connections by nesting in the public subnet.
- Elastic IP Address: Provides a static public IP address for my NAT
- Route table: Private subnet traffic flows to the NAT Gateway which depends on the Internet Gateway for internet access.
```
resource "aws_subnet" "private_subnet" {
    vpc_id                   = aws_vpc.main_vpc.id
    cidr_block               = "10.0.1.0/24"
    map_public_ip_on_launch  = false
    availability_zone        = "us-east-2a"

    tags = {
        Name = "Private Subnet"
    }
}

resource "aws_eip" "EIP" {
    domain = "vpc"

    tags = {
        Name = "NAT"
    }
}

resource "aws_nat_gateway" "NAT" {
    allocation_id = aws_eip.EIP.id
    subnet_id     = aws_subnet.public_subnet.id
    depends_on    = [aws_internet_gateway.IGW]

    tags = {
        Name = "NAT"
```
## 4. Routing
So far, I've created a Virtual Private Cloud with a public and private subnet. The public subnet has an internet gateway for internet access and the private subnet has a NAT gateway that provides internet access while blocking inbound connections. To direct network traffic throughout my VPC and to the internet, I need to create route tables that dictate how data flows. 

For my public subnet, I want traffic to route outbund to the internet using the Internet gateway. I also added routing to my second VPC using VPC Peering, this is for later.
```
resource "aws_route_table" "route_table" {
    vpc_id = aws_vpc.main_vpc.id

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.IGW.id
    }

    route {
        cidr_block                = var.vpc2_cidr_block
        vpc_peering_connection_id = aws_vpc_peering_connection.MAIN_to_VPC_2.id
    }

    tags = {
        Name = "Public Route Table"
    }
}
```
I then associated this route table with my public subnet using route table association
```
resource "aws_route_table_association" "public_rt_assoc" {
    subnet_id      = aws_subnet.public_subnet.id
    route_table_id = aws_route_table.route_table.id
}
```
I set up a route table that directs traffic to my NAT gateway and associated it with my private subnet.
```
resource "aws_route_table" "private_route_table" {
    vpc_id = aws_vpc.main_vpc.id

    route {
        cidr_block     = "0.0.0.0/0"
        nat_gateway_id = aws_nat_gateway.NAT.id    
    }

    tags = {
        Name = "Private Route Table"
    }
}

resource "aws_route_table_association" "private_rt_assoc" {
    subnet_id      = aws_subnet.private_subnet.id
    route_table_id = aws_route_table.private_route_table.id
}
```
## 5. Network Access Control Lists
I now have traffic moving through the use of route tables, but best security practices require I restrict access to only what's needed. Network Access Control Lists (NACLs) manage traffic at the Layer 3 - Network level. Specifically, it controls traffic at the subnet-level. Using NACLs I can allow network traffic that is necessary, and block everything that isn't.

For my public subnet, I will allow inbound HTTPS, SSH, 10.1.0.0/16 (VPC 2 for Peering), and ephemeral return traffic. I will also allow all outbound traffic. These rules provide public access to my EC2 instance using HTTPS, SSH for direct access to my EC2, VPC Peering, and opens ephemeral ports for return traffic.
```
resource "aws_network_acl" "public_acl" {
    vpc_id = aws_vpc.main_vpc.id

    # Allow HTTPS
    ingress {
        rule_no    = 100
        protocol   = "tcp"
        action     = "allow"
        cidr_block = "0.0.0.0/0"
        from_port  = 443
        to_port    = 443
    }

    # Allow traffic from VPC 2
    ingress {
        rule_no    = 110
        protocol   = "-1"
        action     = "allow"
        cidr_block = var.vpc2_cidr_block
        from_port  = 0
        to_port    = 0
    }

    # Allow SSH
    ingress {
        rule_no    = 120
        protocol   = "tcp"
        action     = "allow"
        cidr_block = "0.0.0.0/0" 
        from_port  = 22
        to_port    = 22
    }

    # Allow ephemeral response traffic (1024-65535)
    ingress {
        rule_no    = 130
        protocol   = "tcp"
        action     = "allow"
        cidr_block = "0.0.0.0/0"
        from_port  = 1024
        to_port    = 65535
    }

    # Allow all outbound
    egress {
        rule_no    = 100
        protocol   = "-1"
        action     = "allow"
        cidr_block = "0.0.0.0/0"
        from_port  = 0
        to_port    = 0
    }

    tags = {
        Name = "Public NACL"
    }
}
```
In an actual production environment I would change my SSH rule to only allow connections from specific, secured, IP Ranges and may allow HTTP depending on the needs of my EC2 instance. I tied this NACL to my public subnet using NACL association.
```
resource "aws_network_acl_association" "public_acl_assoc" {
    subnet_id       = aws_subnet.public_subnet.id
    network_acl_id  = aws_network_acl.public_acl.id
}
```
For my private subnet, I am allowing SSH connections from my public subnet. For now, this will allow me to connect from my public EC2 to my private EC2 instance for testing but I plan to add a jump server or use SSM for SSH access to my private EC2. I'm also allowing return traffic and all outbound traffic.

```
resource "aws_network_acl_association" "public_acl_assoc_2" {
    subnet_id       = aws_subnet.public_subnet_2.id
    network_acl_id  = aws_network_acl.public_acl_2.id
}

resource "aws_network_acl" "private_acl" {
    vpc_id = aws_vpc.main_vpc.id
   
  # Inbound SSH from public subnet
    ingress {
        rule_no    = 100
        protocol   = "tcp"
        action     = "allow"
        cidr_block = aws_subnet.public_subnet.cidr_block
        from_port  = 22
        to_port    = 22
    }

    # Allow ephemeral response traffic (1024-65535)
    ingress {
        rule_no    = 110
        protocol   = "tcp"
        action     = "allow"
        cidr_block = "0.0.0.0/0"
        from_port  = 1024
        to_port    = 65535
    }

    # Allow all outbound for NAT access
    egress {
        rule_no    = 100
        protocol   = "-1"
        action     = "allow"
        cidr_block = "0.0.0.0/0"
        from_port  = 0
        to_port    = 0
    }

    tags = {
        Name = "Private NACL"
    }
}

resource "aws_network_acl_association" "private_acl_assoc" {
    subnet_id       = aws_subnet.private_subnet.id
    network_acl_id  = aws_network_acl.private_acl.id
}
```
## 6. EC2 Instances
Alright, my VPC has been set up and its subnets, secured. I will be using the free-tier AMI and instance type to avoid charges, Amazon Linux and T3.Micro. To start, I will test connectivity throughout my network by attempting to SSH from my local computer to my public EC2 and then my private EC2 by using key pairs for direct, secure access to my EC2s.

The following code creates my EC2 with my specified AMI and instance type, subnet, and security group. It then creates a private key using RSA, pairs it with the server's public key, and adds them to a generated PEM file.

Key Pairs: This is done through public key cryptography (asymmetric), a public key is stored on the server and used for encryption. It is paired with its counterpart, a private key, for decryption. This verification between a public and private key allows you to securely SSH into the EC2 instance. 

### Public EC2
-  Hosted in public subnet
-  Security: HTTPS/SSH only
-  Access via SSH key pair
```
resource "aws_instance" "AMI" {
    ami                    = var.AMI
    instance_type          = var.T3_Micro
    subnet_id              = aws_subnet.public_subnet.id
    vpc_security_group_ids = [aws_security_group.SG-public.id]
    key_name = "admin-key"

    tags = {
        Name = "Public Server"
    }
}

resource "tls_private_key" "pkey" {
    algorithm = "RSA"
    rsa_bits  = 4096
}

resource "aws_key_pair" "admin-key" {
    key_name   = "admin-key"
    public_key = tls_private_key.pkey.public_key_openssh
}

resource "local_file" "private_key" {
    content  = tls_private_key.pkey.private_key_pem
    filename = "${path.module}/my-keypair.pem"
}
```
#### Connectivity Test: 
I used my newly created key pair to SSH from my local command prompt directly to my public EC2.

### Private EC2
- Hosted in private subnet
- SSH restricted to public subnet
- Jump from public EC2 using OpenSSH ProxyJump
- Future improvement: SSM Session Manager to eliminate public SSH
```
resource "aws_instance" "AMI_3" {
    ami                    = var.AMI
    instance_type          = var.T3_Micro
    subnet_id              = aws_subnet.private_subnet.id
    vpc_security_group_ids = [aws_security_group.SG-private.id]
    key_name = "private-key"

    tags = {
        Name = "Private Server"
    }
}

resource "tls_private_key" "private_pkey" {
    algorithm = "RSA"
    rsa_bits  = 4096
}

resource "aws_key_pair" "private-key" {
    key_name   = "private-key"
    public_key = tls_private_key.private_pkey.public_key_openssh
}

resource "local_file" "private_key_file" {
    content         = tls_private_key.private_pkey.private_key_pem
    filename        = "${path.module}/private-key.pem"
}
```
#### Connectivity Test:
Currently connected to my public EC2, I attempted to SSH using the private EC2's key pair. Using unique key pairs avoids compromise of both EC2s if the public EC2 were to become compromised. The connectivity test was unsuccessful. After some research I learned I could jump servers using OpenSSH. I added my private keys in ssh-agent and used ProxyJump to successfully SSH to my private EC2.

## 7. Security Groups
I've added rules to my network to control traffic as needed but I can go a step further and manage traffic at the resource level using security groups. Security Groups are similar to NACLs in that they create inbound/outbound rules for resources. In this case, I will set up two security groups. One for my public EC2 instance in my public subnet and another for my private EC2 instance in my private subnet. 

This provides defense-in-depth and hardening by providing layers to my network security when used with NACLs.

My public and private security groups have a similar layout to my NACLs. Since they operate at the resource level, the biggest difference is that I will need to associate them with my EC2 instances and not the subnet(s).

Public EC2:
```
resource "aws_security_group" "SG-public" {
    name        = "Public Security Group"
    description = "Allow TLS/SSH inbound traffic and outbound traffic from the public subnet." 
    vpc_id      = aws_vpc.main_vpc.id

    ingress {
        description  = "HTTPS"
        from_port    = 443
        to_port      = 443
        protocol     = "tcp"
        cidr_blocks  = ["0.0.0.0/0"]
    }

    ingress {
        description  = "Allow traffic from peered VPC"
        from_port    = 0
        to_port      = 0
        protocol     = "-1"
        cidr_blocks  = [aws_vpc.vpc2.cidr_block]
    }

    ingress {
        description  = "SSH"
        from_port    = 22
        to_port      = 22
        protocol     = "tcp"
        cidr_blocks  = ["0.0.0.0/0"]
    }
    
    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }

    tags = {
        Name = "SG-Public"
    }
}
```
Private EC2:
```
resource "aws_security_group" "SG-private" {
    name        = "Private Security Group"
    description = "Allow certain inbound traffic and outbound traffic from the public subnet." 
    vpc_id      = aws_vpc.main_vpc.id

    # SSH only from public security group
    ingress {
        from_port       = 22
        to_port         = 22
        protocol        = "tcp"
        security_groups = [aws_security_group.SG-public.id]
    }
    
    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }

    tags = {
        Name = "SG-Private"
    }
}
```
## 8. VPC Peering
I have set up my VPC and its subnets, managed traffic using route tables, created EC2 servers in the public and private subnet and used NACLs and Security Groups to secure my network and resources. Now, I want to create another VPC. This VPC can be used for redundancy by putting it and its resources in a different availability zone or it can serve as a completely new network with its own infrastructure and resources. 

I want VPC 2 to communicate with my Main VPC using VPC Peering. VPC Peering offers direct communication between VPCs, through the use of their private IP addresses. This is much more secure than sending traffic from a VPC, to the Internet, and then to the other VPC.

- Main VPC: 10.0.0.0/16 (10.0.0.0 - 10.0.0.255.255)
- VPC 2: 10.1.0.0/16 (10.1.0.0 - 10.1.255.255)
```
resource "aws_vpc" "vpc2" {
    cidr_block = var.vpc2_cidr_block

    enable_dns_support   = true
    enable_dns_hostnames = true
    
    tags = {
        Name = "VPC 2"
    }
}
```
VPC 2 Configuration:
Public Subnet 2, Internet Gateway 2, and Route Table: 
```
#===========================================
# 10.1.0.0/16 
#===========================================

resource "aws_subnet" "public_subnet_2" {
    vpc_id                   = aws_vpc.vpc2.id
    cidr_block               = "10.1.0.0/24"
    map_public_ip_on_launch  = true
    availability_zone        = "us-east-2a"

    tags = {
        Name = "Public Subnet 2"
    }
}

resource "aws_internet_gateway" "IGW_2" {
    vpc_id = aws_vpc.vpc2.id

    tags = {
        Name = "IGW 2"
    }
}

resource "aws_route_table" "public_route_table_2" {
    vpc_id = aws_vpc.vpc2.id

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.IGW_2.id
    }

    route {
        cidr_block                = var.vpc_cidr_block
        vpc_peering_connection_id = aws_vpc_peering_connection.MAIN_to_VPC_2.id
    }

    tags = {
        Name = "Public Route Table 2"
    }
}

resource "aws_route_table_association" "public_rt_assoc_2" {
    subnet_id      = aws_subnet.public_subnet_2.id
    route_table_id = aws_route_table.public_route_table_2.id
}
```
I also created a public EC2 that will be hosted in VPC 2's public subnet and I created a NACL for the subnet. Finally, I created another security group for VPC 2 - EC2.

Now that I have two VPCs, Main VPC and VPC 2 I need to create the peering connection between them. I used `aws_vpc_peering_connection` to create this. The full code block:

```
resource "aws_vpc_peering_connection" "MAIN_to_VPC_2" {
    
    vpc_id        = aws_vpc.main_vpc.id
    peer_vpc_id   = aws_vpc.vpc2.id
    auto_accept   = true
    
    accepter {
        allow_remote_vpc_dns_resolution = true
  }

    requester {
        allow_remote_vpc_dns_resolution = true
  }

    tags = {
        Name = "VPC Peering - Main VPC <> VPC 2"
    }
}
```
As mentioned before, I had previously set up the routing, NACLs, and Security Groups to allow traffic to and from VPCs using VPC Peering.

### Connectivity Test:
I used OpenSSH previously, but will be trying EC2 Instance Connect to connect to my Main VPC - Public EC2. To test VPC Peering works, I sent pings using ICMP from my first public server, Main VPC - Public EC2 to VPC 2 - EC2. 

## 9. S3 Access & VPC Endpoints
I would now like to securely access my S3 Bucket I made in "project" from my VPCs. Some AWS services don't live in VPCs and require going through the internet to access. VPC Endpoint allows you to establish a secure connection from Endpoint to your services without going through the internet.
