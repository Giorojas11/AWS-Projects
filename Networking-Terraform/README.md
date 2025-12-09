# AWS Networking Project using Terraform

### This project demonstrates building a secure, multi-VPC AWS environment using Terraform. The network includes public and private subnets, gateways, route tables, security groups, EC2 instances, NACLs, VPC peering, S3 Bucket connectivity using VPC Endpoint, and CloudWatch monitoring.
------------------------------------------
## Table of Contents
1. [Project Overview](#1-project-overview)
      - [Network Diagram](#network-diagram)
2. [VPC Creation](#2-vpc-creation)
3. [Subnets](#3-subnets)
   - [Public Subnet & Internet Gateway](#public-subnet--internet-gateway)
   - [Private Subnet & NAT](#private-subnet--nat)
4. [Routing](#4-routing)
5. [Network Access Control Lists](#5-network-access-control-lists)
6. [EC2 Instances](#6-ec2-instances)
   - [Public EC2](#public-ec2)
   - [Private EC2](#private-ec2)
7. [Security Groups](#7-security-groups)
8. [VPC Peering](#8-vpc-peering)
9. [S3 Access & VPC Endpoints](#9-s3-access--vpc-endpoints)
10. [Monitoring with CloudWatch](#10-monitoring-with-cloudwatch)
11. [Lessons Learned & Next Steps](#11-lessons-learned--next-steps)

## 1. Project Overview
This project sets up a secure, multi-tier AWS network environment using Terraform:
- Two VPCs with public and private subnets
- Defense-in-Depth - NACLs, Security Groups, IAM policies
- SSH + RSA Key Pair(s)
- Routing 
- NAT Gateway for outbound internet access for private subnet
- Internet Gateway for inbound outbound internet access
- VPC peering for private communication between VPCs
- EC2 instances for public-facing and private workloads
- S3 bucket access with private VPC endpoint
- Network monitoring and logging using CloudWatch

  
All infrastructure is fully managed using Terraform, ensuring reproducibility and infrastructure-as-code best practices.

### Network Diagram

![AWS Network Diagram](https://github.com/user-attachments/assets/ef2e31b8-ccfe-4a9e-b475-fae2716d62f6)

## 2. VPC Creation
A Virtual Private Cloud (VPC) is an isolated section of AWS used to organize and secure cloud resources.
- Main VPC CIDR: 10.0.0.0/16 (10.0.0.0 - 10.0.255.255)

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
<img width="1654" height="722" alt="VPC" src="https://github.com/user-attachments/assets/b9c9c0e1-b378-48f3-9e32-5f5fe4e44632" />

## 3. Subnets
Subnetting creates smaller networks within my VPC. I created a public subnet and private subnet for my Main VPC.

### Public Subnet & Internet Gateway
The public subnet hosts resources that need direct internet access.
- CIDR: 10.0.0.0/24 (10.0.0.0 - 10.0.0.255)
- Public IP assignment: Enabled

Key points:
- The internet gateway enables inbound and outbound internet access.
- The route table sends 0.0.0.0/0 traffic to the IGW.


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
<img width="1639" height="512" alt="public subnet" src="https://github.com/user-attachments/assets/35840773-25b9-47b8-aac9-b08e98f8250c" />

<img width="1644" height="469" alt="IGW" src="https://github.com/user-attachments/assets/318ded6b-e853-4446-8025-8762a4ee2745" />


### Private Subnet & NAT
The private subnet hosts sensitive or backend resources that should not be directly exposed to the internet.
- CIDR: 10.0.1.0/24 (10.0.1.0 - 10.0.1.255)
- Public IP assignment: Disabled

Key Points:
- The NAT gateway enables outbound internet access while blocking inbound connections.
- An Elastic IP provides a static IP for the NAT gateway.


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
<img width="1692" height="743" alt="private_subnet" src="https://github.com/user-attachments/assets/4246732b-9b46-4285-aedf-7df224dd88e1" />

<img width="1638" height="568" alt="NAT" src="https://github.com/user-attachments/assets/ff838186-3ae7-48a5-bab4-abdae5ebc611" />
<img width="1636" height="205" alt="EIP" src="https://github.com/user-attachments/assets/a736f58f-c2b9-4ba3-a17c-c31808a1c277" />

## 4. Routing
Route tables determine how network traffic flows.

For the public subnet, the route table sends traffic outbound to the internet gateway and sends traffic to VPC 2, which will be used for the VPC Peering connection.


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


The route table was tied to the public subnet using route table association.


```
resource "aws_route_table_association" "public_rt_assoc" {
    subnet_id      = aws_subnet.public_subnet.id
    route_table_id = aws_route_table.route_table.id
}
```


<img width="693" height="604" alt="rt_and_assoc" src="https://github.com/user-attachments/assets/73176d82-d00b-4b0a-931d-4b224e6ba721" />


This route table routes outbound traffic to the NAT gateway for outbound internet access from the private subnet.


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
Network Access Control Lists (NACLs) manage traffic at the Network layer (L3). They provide subnet-level control over inbound and outbound traffic.

Public Subnet
Allow: 
- HTTPS
- SSH
- Ephemeral response traffic
- VPC Peering to VPC 2
- All outbound traffic.


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

resource "aws_network_acl_association" "public_acl_assoc" {
    subnet_id       = aws_subnet.public_subnet.id
    network_acl_id  = aws_network_acl.public_acl.id
}
```


<img width="1656" height="710" alt="ACL_inbound" src="https://github.com/user-attachments/assets/387469e9-873d-4825-8d27-481b9109f112" />

<img width="1662" height="668" alt="acl_outbound" src="https://github.com/user-attachments/assets/245d9e04-5767-4b0f-9741-7be9cb01a185" />


In an actual production environment I would change my SSH rule to only allow connections from specific IP Ranges and may allow HTTP depending on the needs of my EC2 instance.

Private Subnet
Allow: 
- SSH from public subnet
- Ephemeral return traffic
- Allow outbound traffic.


```
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

<img width="1638" height="529" alt="private_acl" src="https://github.com/user-attachments/assets/dba6cf1b-f481-48ad-a8c9-8b9fb47d503c" />

## 6. EC2 Instances
EC2 instances were deployed in each subnet using t3.micro (free-tier eligible) Amazon Linux AMIs. Key pairs were generated dynamically using Terraform for secure SSH access. 

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
<img width="1722" height="705" alt="EC2" src="https://github.com/user-attachments/assets/a62740f6-cfa7-42ff-a73b-0eca3690a2fc" />


#### Connectivity Test: 
I used my newly created key pair to SSH from my local machine directly to my public EC2.


<img width="1017" height="436" alt="ssh" src="https://github.com/user-attachments/assets/298dcc59-848d-4e43-876e-5274862942eb" />

### Private EC2
- Hosted in private subnet
- SSH restricted to public subnet
- Connection via OpenSSH/ProxyJump
- Future improvement: SSM Session Manager to eliminate public SSH connection


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
<img width="1644" height="663" alt="private_EC2" src="https://github.com/user-attachments/assets/cfad6240-1b8b-432d-a809-769c54adf042" />

#### Connectivity Test:
From the public EC2, I attempted to SSH using the private EC2's key pair. Unique key pairs for each EC2 prevents full compromise of both EC2s if the public EC2 were compromised. The connectivity test was unsuccessful. After some research I learned I could connect using OpenSSH. I added my private keys in ssh-agent and used ProxyJump to successfully SSH to my private EC2.


<img width="1062" height="343" alt="ssh_to_prvEC2_troubleshooting" src="https://github.com/user-attachments/assets/6b5fc5d5-548a-43d2-be20-384011a1584e" />


## 7. Security Groups
Security Groups are similar to NACLs in that they create inbound and outbound rules for resources. Two security groups are needed for the public EC2 and the private EC2.
This provides defense-in-depth and hardening by providing layers to my network security when used with NACLs.

The public and private security groups have a similar layout to the NACLs. Since they operate at the resource level, the biggest difference is that they need to be associated with the EC2 instances instead of a subnet.

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


<img width="1616" height="660" alt="EC2_SG" src="https://github.com/user-attachments/assets/6b71d45b-72db-464e-b2e9-7b0110726914" />


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


<img width="1610" height="485" alt="priv_SG" src="https://github.com/user-attachments/assets/d68c1ccc-d275-4851-a764-875139c50ce2" />

## 8. VPC Peering
VPC Peering offers direct communication between VPCs, through the use of their private IP addresses. This is much more secure than sending traffic from a VPC, to the Internet, and then to another VPC.

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
<img width="1654" height="741" alt="VPC_2" src="https://github.com/user-attachments/assets/6d928739-e5f8-4ec9-bab9-5a834b2a8f63" />
<img width="1628" height="711" alt="pub_subnet_2" src="https://github.com/user-attachments/assets/de895958-e1a8-4453-a853-0a4dad1246ed" />
<img width="1650" height="321" alt="IGW2" src="https://github.com/user-attachments/assets/5b9a2c0f-8d7a-4552-a126-5f8cb77c274a" />
<img width="1610" height="557" alt="Pub_EC2_2" src="https://github.com/user-attachments/assets/b603f7fc-5c24-4322-bef2-0631f04b116e" />

Routing, NACLs, and SGs were created and updated to allow communication between VPCs.

Creation of the VPC Peering connection between the Main VPC and VPC 2.

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
<img width="1650" height="726" alt="peering vpc" src="https://github.com/user-attachments/assets/296e7189-6cf2-47dd-a65f-c1db61bb8791" />

### Connectivity Test:
I used OpenSSH previously, so I will be trying EC2 Instance Connect to connect to Main VPC - Public EC2. To test VPC Peering works, I sent pings from Main VPC - Public EC2 to VPC 2 - Public EC2. 

<img width="945" height="660" alt="ping_EC2s_connected" src="https://github.com/user-attachments/assets/35dfc5ca-b0c5-45d1-a1e2-acc100945457" />

## 9. S3 Access & VPC Endpoints
VPC Endpoint allows for secure access from VPC to [S3 Bucket](https://github.com/Giorojas11/AWS-Projects/tree/main/S3-Bucket-Terraform). Some AWS services aren't hosted in VPCs and require internet access, which can be a security risk. VPC Endpoint allows you to establish a secure connection from an Endpoint to your services without the internet.

I created an endpoint for the Main VPC updated the routing for the public and private subnets.

```
resource "aws_vpc_endpoint" "s3_endpoint" {
  vpc_id       = aws_vpc.main_vpc.id
  service_name = "com.amazonaws.us-east-2.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.private_route_table.id,
    aws_route_table.route_table.id
   ]

  tags = {
    Environment = "MAIN VPC"
  }
}
```

<img width="911" height="97" alt="s3_ec2" src="https://github.com/user-attachments/assets/f07dd97e-938c-4e94-b4fb-e8d3aa15c7da" />


The following bucket policy denies ALL traffic to the S3 Bucket and only allows access from the VPC Enpoint and GROJAS-IAM-ADMIN account.


```
resource "aws_s3_bucket_policy" "vpc_endpoint" {
  bucket = aws_s3_bucket.my_bucket.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "Deny All"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource  = [
            "${aws_s3_bucket.my_bucket.arn}",
            "${aws_s3_bucket.my_bucket.arn}/*"
        ]
        Condition = {
            StringNotEquals = {
                "aws:SourceVpce"   = "${aws_vpc_endpoint.s3_endpoint.id}"
                "aws:PrincipalArn" = [
                    "arn:aws:iam::660410403267:user/GROJAS-IAM-ADMIN"
                ]
            }
        }
      }
    ]
  })
}
```

### Connectivity Test:
To test connectivity, I connected to my Public EC2 server and successfully downloaded image.png from the S3 Bucket and saved it to /home/ec2-user/.
<img width="942" height="266" alt="img_download" src="https://github.com/user-attachments/assets/03005b5a-2a81-4e6a-8a7b-f01e8c70fd8c" />


But is the bucket policy fully in effect? Yes, when logged into the Root account, I cannot view the bucket's objects and receive error messages. When signed into GROJAS-IAM-USER, I am able to view S3 Bucket's object: image.png.

<img width="624" height="668" alt="access_error_root_acct" src="https://github.com/user-attachments/assets/d749be2d-c870-4422-8aaf-d39829f19c35" />

<img width="643" height="590" alt="s3_IAM_account_accessible" src="https://github.com/user-attachments/assets/74a338e5-35d6-4733-beed-89d63d572870" />

The image:


<img width="350" height="500" alt="gnomey" src="https://github.com/user-attachments/assets/4bc056ca-9b7b-401d-91fb-232e6b53278b" />


## 10. Monitoring with CloudWatch
CloudWatch Flow Logs were enabled for the VPCs, capturing:
- Accepted traffic
- Rejected traffic
- Traffic metadata

A dedicated Log Group, IAM role and IAM policy were created for flow logs. 

```
   # VPC CloudWatch Log Group
resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
    name              = "/aws/vpc/flow-logs"
    retention_in_days = 30  
}

   # IAM Role for VPC Flow Logs
resource "aws_iam_role" "vpc_flow_logs_role" {
    name = "VPCFlowLogsRole"

    assume_role_policy = jsonencode({
        Version   = "2012-10-17"
        Statement = [
            {
                Action    = "sts:AssumeRole"
                Effect    = "Allow"
                Principal = {
                    Service = "vpc-flow-logs.amazonaws.com"
                }
            }
        ]
    })
}

   # IAM Policy for VPC Flow Logs
resource "aws_iam_role_policy" "vpc_flow_logs_policy" {
    name = "vpc-flow-logs-policy"
    role = aws_iam_role.vpc_flow_logs_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
        Effect = "Allow"
        Action = [
            "logs:CreateLogStream",
            "logs:PutLogEvents",
            "logs:DescribeLogGroups",
            "logs:DescribeLogStreams"
      ]
         Resource = "*"
    }]
  })
}

   # MAIN VPC Flow Logs
resource "aws_flow_log" "main_vpc_flow" {
    vpc_id               = aws_vpc.main_vpc.id
    log_destination      = aws_cloudwatch_log_group.vpc_flow_logs.arn
    log_destination_type = "cloud-watch-logs"
    traffic_type         = "ALL"
    iam_role_arn         = aws_iam_role.vpc_flow_logs_role.arn
}

   # VPC 2 Flow Logs
resource "aws_flow_log" "vpc2_flow" {
    vpc_id               = aws_vpc.vpc2.id
    log_destination      = aws_cloudwatch_log_group.vpc_flow_logs.arn
    log_destination_type = "cloud-watch-logs"
    traffic_type         = "ALL"
    iam_role_arn         = aws_iam_role.vpc_flow_logs_role.arn
}
```

### Testing
When pinging from VPC to VPC, logs are created for the ICMP traffic.
<img width="1254" height="631" alt="CloudWatch_Ping_test" src="https://github.com/user-attachments/assets/4dfca927-b93e-4e6d-bd83-1f80c141a8c3" />

<img width="1636" height="736" alt="ping_flow_logs" src="https://github.com/user-attachments/assets/b1d3cf5f-7bbf-4c91-bb4e-aaf93791a165" />

## 11. Lessons Learned & Next Steps
- VPCs require careful planning of CIDR blocks and routing
- Security is layered: Security Groups, NACLs, and IAM policies
- Jump hosts and proxy SSH can be replaced with SSM for production
- Terraform organization is critical for maintainability
- Monitoring with CloudWatch provides insight into network traffic

Next Steps:
- Add additional subnets in different AZs for redundancy and isolated subnets for sensitive data
- Implement VPC endpoints for other AWS services
- Utilize other AWS services in my homelab like Lambda Functions and GuardDuty
- Integrate CloudWatch alarms and dashboards for proactive monitoring of attacks - SSH Brute Force, anomalous network traffic, malicious source IPs, etc. 
