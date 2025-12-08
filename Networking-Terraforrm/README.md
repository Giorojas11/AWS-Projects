# AWS Networking Project using Terraform

### This project demonstrates building a secure, multi-VPC AWS environment using Terraform. The network includes public and private subnets, gateways, route tables, security groups, EC2 instances, NACLs, VPC peering, S3 Bucket connectivity using VPC Endpoint, and CloudWatch monitoring.
------------------------------------------
## Table of Contents
1. [https://github.com/Giorojas11/AWS-Projects/blob/main/Networking-Terraforrm/README.md#1-project-overview](#-1.-project-overview)
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
11. Lessons Learned & Next Steps

## 1. Project Overview
This project sets up a secure, multi-tier AWS network environment using Terraform:
- Two VPCs with public and private subnets
- Defense-in-Depth - NACLs, Security Groups, IAM policies
- EC2 instances for public-facing and private workloads
- NAT Gateway for outbound internet access for private subnet
- VPC peering for private communication between VPCs
- S3 bucket access with private VPC endpoint
- Network monitoring and logging using CloudWatch
All infrastructure is fully managed using Terraform, ensuring reproducibility and infrastructure-as-code best practices.

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
## 3. Subnets
Subnetting creates smaller networks within my VPC. I created a public subnet and private subnet for my Main VPC.

### Public Subnet & Internet Gateway
The public subnet hosts resources that need direct internet access.
- CIDR: 10.0.0.0/24 (10.0.0.0 - 10.0.0.255)
- Public IP assignment: Enabled

Key points:
- The Internet Gateway required for inbound and outbound internet access.
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
### Private Subnet & NAT
The private subnet host sensitive or backend resources that should not be directly exposed to the internet.
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
## 4. Routing
Route tables determine how traffic flows within and outside the VPC.

For my public subnet, the route table sends traffic outbound to the Internet gateway. I also added a route sending traffic to VPC 2, which will be used in VPC Peering.
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
I associated this route table with my public subnet using route table association.
```
resource "aws_route_table_association" "public_rt_assoc" {
    subnet_id      = aws_subnet.public_subnet.id
    route_table_id = aws_route_table.route_table.id
}
```
This route table routes outbound traffic to the NAT gateway.
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
Network Access Control Lists (NACLs) manage traffic at the Layer 3 - Network. They provide subnet-level control over inbound and outbound traffic.

Public Subnet
Allow: HTTPS, SSH, ephemeral response traffic, VPC Peering to VPC 2, and all outbound traffic.
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
In an actual production environment I would change my SSH rule to only allow connections from specific, IP Ranges and may allow HTTP depending on the needs of my EC2 instance.
```
resource "aws_network_acl_association" "public_acl_assoc" {
    subnet_id       = aws_subnet.public_subnet.id
    network_acl_id  = aws_network_acl.public_acl.id
}
```
Private Subnet
Allow: SSH from public subnet, ephemeral return traffic, and allow outbound traffic.

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
#### Connectivity Test: 
I used my newly created key pair to SSH from my local machine directly to my public EC2.

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
#### Connectivity Test:
From the public EC2, I attempted to SSH using the private EC2's key pair. Unique key pairs prevents full compromise of both EC2s if the public EC2 were compromised. The connectivity test was unsuccessful. After some research I learned I could connect using OpenSSH. I added my private keys in ssh-agent and used ProxyJump to successfully SSH to my private EC2.

## 7. Security Groups
Security Groups are similar to NACLs in that they create inbound and outbound rules for resources. Two security groups are needed for the public EC2 and the private EC2.

This provides defense-in-depth and hardening by providing layers to my network security when used with NACLs.

The public and private security groups have a similar layouts to NACLs. Since they operate at the resource level, the biggest difference is that they need to be associated with the EC2 instances instead of a subnet.

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
VPC Peering offers direct communication between VPCs, through the use of their private IP addresses. This is much more secure than sending traffic from a VPC, to the Internet, and then to the other VPC.

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
Routing, NACLs, and SGs were updated to allow communication between VPCs.

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
### Connectivity Test:
I used OpenSSH previously, so I will be trying EC2 Instance Connect to connect to Main VPC - Public EC2. To test VPC Peering works, I sent pings from Main VPC - Public EC2 to VPC 2 - Public EC2. 

## 9. S3 Access & VPC Endpoints
VPC Endpoint allows for secure access from VPC to <a href=https://github.com/Giorojas11/AWS-Projects/tree/main/S3-Bucket-Terraform>S3 Bucket</a>. Some AWS services aren't hosted in VPCs and require internet access, which can be a security risk. VPC Endpoint allows you to establish a secure connection from an Endpoint to your services without the internet.

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

But is the bucket policy fully in effect? Yes, when logged into the Root account, I cannot view the bucket's objects and receive error messages. When signed into GROJAS-IAM-USER, I am able to view S3 Bucket's object: image.png.

The image:

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
