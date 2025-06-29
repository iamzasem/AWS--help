# AWS Virtual Private Cloud (VPC)

## What is a VPC

A VPC (Virtual Private Cloud) is your own private network inside AWS. It lets you control your own IP addresses, subnets, routing, and firewall settings. 

It works like a company’s private office network, but instead of cables and switches, everything is virtual and runs inside AWS.

In your VPC, you can launch services like:
- EC2 (servers)
- RDS (databases)
- Load Balancers
- Lambda functions
- S3 buckets (with private access using VPC endpoints)

You decide:
- Which part of your app can talk to the internet
- Which part stays private
- How to protect it from hackers
- How your app grows over time

Without VPC, all your cloud services would be open or mixed with other customers, which is not safe.

---

## Use Case

You run an **online electronics store** hosted on AWS. Your website sells phones and laptops. Your system needs:
- A public frontend so users can see products
- A secure backend database for storing orders
- A load balancer for handling many users
- A payment system that talks to an external service
- A private analytics app that collects data

To organize all this, you create a **VPC**. Inside the VPC, you create:
- Public subnets for web servers
- Private subnets for databases and apps
- Internet Gateway for public access
- NAT Gateway for private servers to get updates
- Security rules to block hackers

This setup is secure, flexible, and scalable.

---

## Components of a VPC 

### 1. CIDR Block

CIDR means Classless Inter-Domain Routing. It defines how many IP addresses you want in your VPC.

For example:  
`10.0.0.0/16` → gives you 65,536 IPs (from 10.0.0.0 to 10.0.255.255)

Important rules:
- Use private ranges like 10.x.x.x or 192.168.x.x
- Plan CIDR carefully so that future connections (like peering) don’t conflict

If you make a VPC with too small a CIDR, you will run out of IPs later.

---

### 2. Subnets

A subnet is a smaller network inside the VPC. You divide your VPC into subnets to control access and improve structure.

- **Public subnet**: Can connect to the internet using an Internet Gateway  
- **Private subnet**: No internet access unless through a NAT Gateway  

Each subnet stays in one **Availability Zone**. To make your app highly available, use at least 2 subnets in 2 zones.

Example:
- Public Subnet A: 10.0.1.0/24 (web server in zone 1)
- Public Subnet B: 10.0.2.0/24 (web server in zone 2)
- Private Subnet A: 10.0.3.0/24 (database in zone 1)
- Private Subnet B: 10.0.4.0/24 (backend service in zone 2)

---

### 3. Internet Gateway (IGW)

IGW connects your VPC to the internet.

- It’s required for public subnets
- You attach it to the VPC and set a route to 0.0.0.0/0

No IGW → no internet access

Your public web servers must:
- Be in a subnet that has a route to IGW
- Have a public or Elastic IP
- Security group must allow internet ports (80, 443)

---

### 4. Route Tables

Route tables control where traffic goes. Each subnet uses one route table.

Example:

| Destination      | Target          |
|------------------|-----------------|
| 10.0.0.0/16      | local           |
| 0.0.0.0/0        | igw-xxxxx       |

- Use 0.0.0.0/0 → IGW for public subnet
- Use 0.0.0.0/0 → NAT Gateway for private subnet

Each subnet is linked to only one route table.

---

### 5. NAT Gateway

Private subnets cannot go to the internet directly. But sometimes, they need to download updates or software.

NAT Gateway solves this:
- Deployed in a public subnet
- Has an Elastic IP
- Private subnet routes outbound traffic to the NAT

NAT Gateway does not allow inbound traffic. This keeps your private servers safe.

---

### 6. Security Groups

Security groups are like firewalls for AWS services.

- They control inbound and outbound rules
- Example: Allow port 80 and 443 for web server
- They are stateful: if outbound is allowed, response traffic is auto-allowed

Example rules for a web server:

| Type   | Protocol | Port | Source      |
|--------|----------|------|-------------|
| HTTP   | TCP      | 80   | 0.0.0.0/0   |
| HTTPS  | TCP      | 443  | 0.0.0.0/0   |
| SSH    | TCP      | 22   | YourIP/32   |

---

### 7. Network ACLs

NACLs are firewall rules at the subnet level.

- They are stateless: both inbound and outbound must be allowed
- Useful to block IPs from entire subnet

Use case:
- You want to block a malicious IP from reaching any server

---

### 8. Elastic IP

A static public IP address.

Use case:
- You want a fixed IP for your EC2
- Good for DNS, external services, or whitelisting

Note: AWS charges for unused Elastic IPs

---

### 9. VPC Peering

Used to connect two VPCs privately.

- Must be manually accepted on both sides
- Add routes to reach the other VPC

Use case:
- Your frontend app is in one VPC
- Your analytics system is in another VPC

They communicate over private IPs.

---

### 10. VPC Endpoints

Let you connect to AWS services **without** going through the internet.

- Saves cost
- Increases security

Example:  
You store images in S3. Instead of using public internet, your EC2 in private subnet connects using VPC endpoint.

Two types:
- Gateway endpoint (S3, DynamoDB)
- Interface endpoint (other AWS services)

---

### 11. VPC Flow Logs

Flow logs capture network traffic.

- Useful for debugging or security checks
- Can send logs to CloudWatch or S3

Use case:
- Check why a server is not reachable
- Detect unusual traffic patterns

---

### 12. Transit Gateway

Used when you need to connect many VPCs.

- Works like a central router
- Easier than setting up many peering connections

Use case:
- A large company with 10+ VPCs
- One Transit Gateway manages them all

---

### 13. PrivateLink

Secure way to connect to third-party services or internal APIs across VPCs.

- Uses interface endpoint
- Traffic stays inside AWS network

Use case:
- You use a third-party payment API
- PrivateLink lets you call it without going to public internet

---

### 14. DHCP Options Set

DHCP options allow you to control DNS and hostname settings in your VPC.

Use case:
- You want to use a custom DNS server for your EC2s

---

## Production Problems and Fixes

### Problem 1: EC2 in Public Subnet Not Accessible from Internet

**Symptoms**:
- EC2 has public IP
- But still cannot reach it from browser

**Fix**:
- Check if Internet Gateway is attached to VPC
- Check if subnet route table has `0.0.0.0/0` to IGW
- Check security group allows port 80/443
- Ensure NACL is not blocking traffic
- Make sure the EC2 is running and health checks are OK

---

### Problem 2: EC2 in Private Subnet Cannot Update OS or Packages

**Symptoms**:
- yum or apt-get fails
- No internet access from private instance

**Fix**:
- Create NAT Gateway in public subnet
- Assign Elastic IP to NAT Gateway
- In private subnet’s route table, add:
  `0.0.0.0/0 → NAT Gateway`
- Test again from EC2

---

### Problem 3: Peering Connection Active But EC2s Can’t Talk

**Symptoms**:
- VPC peering status is “active”
- Ping or curl fails

**Fix**:
- Add routes in both VPCs’ route tables to reach each other
- Check if security group allows traffic from the other VPC’s CIDR
- Make sure NACLs don’t block the traffic

---

### Problem 4: High S3 Data Transfer Bill

**Symptoms**:
- EC2 downloads from S3
- Bills show high data egress charges

**Fix**:
- Use a **Gateway VPC Endpoint** for S3
- No internet traffic = no data egress costs
- Update route table to send S3 traffic through endpoint

---

### Problem 5: Random Port Scans on Public IP

**Symptoms**:
- Flow logs show hits on random ports
- Web server is slow

**Fix**:
- Use NACL to block common attack ports (e.g., 3389, 23)
- Restrict security group to known IPs (e.g., for SSH)
- Use AWS WAF in front of Load Balancer

---

### Problem 6: Can’t Connect EC2 to RDS in Same VPC

**Symptoms**:
- EC2 ping works
- App cannot connect to DB

**Fix**:
- Check if RDS security group allows EC2 private IP or SG
- Check route table (should already be local)
- Make sure RDS is in same or reachable subnet
- Test with `telnet` or `nc` to the port

---

# Steps to Create and Set Up a VPC in AWS

## E-Commerce Website Example

The steps below create a VPC for an e-commerce website selling electronics, like smartphones and laptops. The setup includes web servers in public subnets for customers to access the website, a database in a private subnet for storing orders, and a load balancer to handle traffic. The configuration ensures security, scalability, and connectivity using AWS best practices.

## Steps to Create and Set Up a VPC

1. Sign in to the AWS Management Console using your AWS account credentials.
2. Go to the VPC service by searching for "VPC" in the search bar or navigating to Services, then Networking and Content Delivery, then VPC.
3. Click Create VPC to start setting up a new VPC.
4. In the Create VPC window, select VPC only, enter a name like "EcommerceVPC", and set the IPv4 CIDR block to 10.0.0.0/16 for 65,536 IP addresses.
5. Click Create VPC to create the VPC, then click Close.
6. Go to Subnets in the VPC Dashboard and click Create subnet.
7. Select the VPC you created (EcommerceVPC) and create four subnets:
   1. Public Subnet 1: Name it "PublicSubnet1", set Availability Zone to us-east-1a, and use CIDR block 10.0.1.0/24 (256 IPs).
   2. Public Subnet 2: Name it "PublicSubnet2", set Availability Zone to us-east-1b, and use CIDR block 10.0.2.0/24.
   3. Private Subnet 1: Name it "PrivateSubnet1", set Availability Zone to us-east-1a, and use CIDR block 10.0.3.0/24.
   4. Private Subnet 2: Name it "PrivateSubnet2", set Availability Zone to us-east-1b, and use CIDR block 10.0.4.0/24.
8. Click Create subnet, then click Close.
9. Go to Internet Gateways in the VPC Dashboard and click Create internet gateway.
10. Name it "EcommerceIGW" and click Create internet gateway, then click Close.
11. Select the internet gateway, click Actions, and choose Attach to VPC.
12. Select EcommerceVPC and click Attach internet gateway.
13. Go to Route Tables in the VPC Dashboard and find the default route table for EcommerceVPC (this will be the private route table).
14. Create a new route table for public subnets by clicking Create route table, naming it "PublicRouteTable", selecting EcommerceVPC, and clicking Create.
15. Select PublicRouteTable, go to the Routes tab, and click Edit routes.
16. Add a route: set Destination to 0.0.0.0/0 and Target to the internet gateway (EcommerceIGW), then click Save routes.
17. Go to the Subnet associations tab for PublicRouteTable, click Edit subnet associations, select PublicSubnet1 and PublicSubnet2, and click Save associations.
18. Go to Elastic IPs in the EC2 Dashboard and click Allocate Elastic IP address.
19. Click Allocate, then select the new Elastic IP and note its address.
20. Go to NAT Gateways in the VPC Dashboard and click Create NAT gateway.
21. Select PublicSubnet1, choose the Elastic IP, and click Create NAT gateway.
22. Go to the private route table, click Edit routes, add a route with Destination 0.0.0.0/0 and Target as the NAT gateway, then click Save routes.
23. Go to Security Groups in the EC2 Dashboard and click Create security group.
24. Create two security groups:
    1. WebServerSG: Name it "WebServerSG", select EcommerceVPC, add inbound rules for HTTP (port 80) and HTTPS (port 443) from 0.0.0.0/0, and add an outbound rule allowing all traffic.
    2. DatabaseSG: Name it "DatabaseSG", select EcommerceVPC, add an inbound rule for MySQL (port 3306) from the WebServerSG security group, and add an outbound rule allowing all traffic.
25. Click Create security group for each.
26. Go to Network ACLs in the VPC Dashboard and create a new NACL named "EcommerceNACL".
27. Add inbound rules to allow HTTP (port 80), HTTPS (port 443), and MySQL (port 3306) from 10.0.0.0/16, and allow all outbound traffic.
28. Associate EcommerceNACL with PublicSubnet1, PublicSubnet2, PrivateSubnet1, and PrivateSubnet2.
29. Go to EC2 Dashboard, click Launch instances, and launch two EC2 instances:
    1. Web Server: Use Amazon Linux 2 AMI, t2.micro instance type, place in PublicSubnet1, assign WebServerSG, and enable Auto-assign Public IP.
    2. Second Web Server: Same settings, but place in PublicSubnet2.
30. Go to RDS Dashboard, click Create database, and launch an RDS instance:
    1. Choose MySQL, Free Tier, place in PrivateSubnet1, assign DatabaseSG, and disable public access.
31. Go to Elastic Load Balancing in the EC2 Dashboard and click Create load balancer.
32. Choose Application Load Balancer, name it "EcommerceLB", select EcommerceVPC, map to PublicSubnet1 and PublicSubnet2, assign WebServerSG, and add the two EC2 instances to the target group for HTTP (port 80).
33. Go to VPC Endpoints in the VPC Dashboard and create a Gateway Endpoint for S3.
34. Select EcommerceVPC, choose S3 service, associate with the private route table, and click Create endpoint.
35. Go to VPC Flow Logs in the VPC Dashboard and click Create flow log.
36. Set Filter to All, Destination to an S3 bucket or CloudWatch Logs, select EcommerceVPC, and click Create.
37. Test the setup:
    1. Access the load balancer’s DNS name in a browser to verify the website loads.
    2. Connect to the EC2 instances via SSH and confirm they can access S3 via the endpoint.
    3. Verify the RDS database is only accessible from the EC2 instances.

## E-Commerce Website Setup Outcome

The VPC setup supports an e-commerce website selling electronics:
1. Web servers in PublicSubnet1 and PublicSubnet2 handle customer traffic, accessible via the load balancer.
2. The RDS database in PrivateSubnet1 stores order data securely, accessible only by web servers.
3. The NAT gateway allows the database to download updates without public exposure.
4. The S3 endpoint enables web servers to retrieve product images privately.
5. Flow Logs monitor traffic for security and troubleshooting.



## Common Interview Questions with Answers

**1. What is a VPC in AWS?**  
A VPC is a private network in AWS where you can launch resources like EC2 and RDS. You control IP ranges, subnets, routing, and security.

**2. What is the difference between public and private subnet?**  
Public subnet has a route to the internet via an Internet Gateway. Private subnet has no direct internet route and uses NAT Gateway for outbound traffic.

**3. How does a NAT Gateway work?**  
NAT Gateway lets private resources access the internet. It sits in a public subnet and translates private IPs to a public Elastic IP.

**4. What is the difference between Security Group and NACL?**  
Security Group is at instance level, stateful. NACL is at subnet level, stateless. Security groups are easier to manage for EC2.

**5. How do you connect two VPCs together?**  
Using VPC peering. Create a peering connection and update route tables on both sides.

**6. What is a VPC Endpoint?**  
A way to privately connect to AWS services like S3 or DynamoDB without going through the internet.

**7. What are Flow Logs used for?**  
To record and analyze VPC traffic. Helps in security and troubleshooting.

**8. What is a Transit Gateway?**  
A central hub to connect many VPCs and networks. Easier to manage than multiple peering connections.

**9. How do you secure a web application in VPC?**  
Use private subnets for backend, security groups to control traffic, NACLs for subnet-level rules, and VPC endpoints for private service access.

**10. What happens if two VPCs have overlapping CIDR blocks?**  
You cannot create peering between them. Traffic will not be routed correctly.

