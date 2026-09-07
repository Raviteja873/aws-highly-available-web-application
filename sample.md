# AWS Highly Available Web Application

A hands-on AWS project demonstrating how to build a highly available web application using Amazon VPC, public and private subnets, EC2, Auto Scaling Group, Application Load Balancer, Target Group, NAT Gateway, Internet Gateway, Bastion Host, Security Groups, and Multi-AZ architecture.

---

## 📌 Project Overview

This project demonstrates a production-style AWS web application architecture deployed across two Availability Zones in the Mumbai (`ap-south-1`) AWS Region.

The application servers run on Amazon EC2 instances inside private subnets. Users access the application through an Internet-facing Application Load Balancer deployed across public subnets.

The Application Load Balancer forwards incoming HTTP requests to healthy EC2 instances registered in a Target Group.

An Auto Scaling Group manages the application instances and maintains the desired number of servers.

A Bastion Host is deployed in a public subnet and is used to connect to the private EC2 instances through SSH.

The private EC2 instances use NAT Gateways for outbound internet connectivity when required.

---

## 🏗️ Architecture

![AWS Architecture Diagram](architecture/architecture-diagram.png)

### High-Level Architecture

    Internet
        |
        | HTTP :80
        v
    +---------------------------+
    | Application Load Balancer |
    |       aws-project-lb      |
    |       Internet-facing     |
    +-------------+-------------+
                  |
                  | HTTP :8000
                  v
    +---------------------------+
    |       Target Group        |
    |        aws-project        |
    +-------------+-------------+
                  |
          +-------+-------+
          |               |
          v               v
    +-------------+ +-------------+
    | Private EC2 | | Private EC2 |
    |     #1      | |     #2      |
    | ap-south-1a | | ap-south-1b |
    | 10.0.131.226| | 10.0.144.215|
    | Port 8000   | | Port 8000   |
    +-------------+ +-------------+
          |               |
          v               v
      NAT Gateway      NAT Gateway


    Administrator
          |
          | SSH :22
          v
    +----------------------+
    |     Bastion Host     |
    | bastion-host-for-    |
    |      project         |
    +----------+-----------+
               |
               +--------------------> Private EC2 #1
               |
               +--------------------> Private EC2 #2

---

# 1. AWS Region

The project was created in the Mumbai AWS Region.

    Region:
    ap-south-1

Two Availability Zones were used:

    ap-south-1a
    ap-south-1b

Using two Availability Zones improves application availability because the application is not dependent on a single Availability Zone.

### Screenshot

![AWS Region](screenshots/01-region.png)

---

# 2. Create VPC

The first infrastructure component created was the Amazon VPC.

AWS Console:

    AWS Console
    → VPC
    → Your VPCs
    → Create VPC

The `VPC and more` option was selected instead of `VPC only`.

### VPC Configuration

    Name:
    aws-project-vpc-vpc

    IPv4 CIDR:
    10.0.0.0/16

    Availability Zones:
    2

    Public Subnets:
    2

    Private Subnets:
    2

    NAT Gateways:
    1 per Availability Zone

    VPC Endpoint:
    S3 Gateway

The VPC provides the main networking boundary for the entire project.

### Why VPC?

A VPC provides an isolated virtual network where AWS resources such as EC2 instances can communicate with each other using private IP addresses.

### Screenshot

![VPC Creation](screenshots/02-vpc-creation.png)

---

# 3. VPC Subnet Design

The VPC contains four subnets distributed across two Availability Zones.

    VPC
    10.0.0.0/16
    |
    +----------------------------------+
    |                                  |
    |          ap-south-1a             |
    |                                  |
    |   Public Subnet                  |
    |                                  |
    |   Private Subnet                 |
    |                                  |
    +----------------------------------+
    |
    +----------------------------------+
    |                                  |
    |          ap-south-1b             |
    |                                  |
    |   Public Subnet                  |
    |                                  |
    |   Private Subnet                 |
    |                                  |
    +----------------------------------+

Public subnets are used for resources that need to receive internet traffic, such as the Internet-facing ALB and Bastion Host.

Private subnets are used for application EC2 instances.

### Screenshot

![VPC Subnets](screenshots/03-vpc-subnets.png)

---

# 4. Internet Gateway

An Internet Gateway was created and attached to the VPC.

The Internet Gateway provides a path between the VPC and the public internet for resources in public subnets.

Traffic from public resources follows:

    Public Resource
          |
          v
    Public Route Table
          |
          v
    Internet Gateway
          |
          v
       Internet

### Screenshot

![Internet Gateway](screenshots/04-internet-gateway.png)

---

# 5. Route Tables

Route tables were automatically created as part of the VPC configuration.

The public route table provides internet access through the Internet Gateway.

    Public Subnet
          |
          v
    Public Route Table
          |
          | 0.0.0.0/0
          v
    Internet Gateway
          |
          v
       Internet

Private subnets use private route tables.

Private subnet outbound internet traffic is sent through a NAT Gateway.

    Private EC2
          |
          v
    Private Route Table
          |
          v
    NAT Gateway
          |
          v
    Internet Gateway
          |
          v
       Internet

### Screenshot

![Route Tables](screenshots/05-route-tables.png)

---

# 6. NAT Gateways

Two NAT Gateways were created, one in each Availability Zone.

    ap-south-1a
        |
        +---- NAT Gateway 1

    ap-south-1b
        |
        +---- NAT Gateway 2

The purpose of a NAT Gateway is to allow resources in private subnets to initiate outbound connections to the internet without giving those resources public IP addresses.

For example:

    Private EC2
        |
        v
    NAT Gateway
        |
        v
    Internet

The internet cannot directly initiate a connection to the private EC2 through the NAT Gateway.

### Screenshot

![NAT Gateways](screenshots/06-nat-gateways.png)

---

# 7. S3 Gateway Endpoint

An S3 Gateway VPC Endpoint was configured.

The endpoint allows resources in the VPC to access Amazon S3 without requiring traffic to go through the NAT Gateway.

Conceptually:

    Private EC2
        |
        v
    S3 Gateway Endpoint
        |
        v
       S3

This can reduce NAT Gateway usage when private resources need to access S3.

### Screenshot

![S3 Gateway Endpoint](screenshots/07-s3-endpoint.png)

---

# 8. Create Security Group

A Security Group was created for the project.

    Security Group:
    aws-project-sg

The Security Group was associated with the EC2 instances.

The application uses:

    TCP
    Port 8000

SSH uses:

    TCP
    Port 22

The Security Group acts as a virtual firewall controlling traffic to the EC2 instances.

### Important Security Concept

The application servers are not directly exposed to the public internet.

The intended traffic path is:

    Internet
        |
        v
    Application Load Balancer
        |
        v
    Private EC2

Administrative access follows:

    Administrator
        |
        v
    Bastion Host
        |
        v
    Private EC2

### Screenshot

![Security Group](screenshots/08-security-group.png)

---

# 9. Create Launch Template

A Launch Template was created for the application EC2 instances.

AWS Console:

    EC2
    → Launch Templates
    → Create launch template

### Launch Template Configuration

    Launch Template Name:
    aws-project-launch-template

    Operating System:
    Ubuntu 26.04 LTS

    Instance Type:
    t3.micro

    Key Pair:
    my-key

    Security Group:
    aws-project-sg

    VPC:
    aws-project-vpc-vpc

The Launch Template stores the configuration required when launching EC2 instances.

Instead of manually configuring every EC2 instance, the Auto Scaling Group can use the Launch Template to launch instances with the same configuration.

### Screenshot

![Launch Template](screenshots/09-launch-template.png)

---

# 10. Create Auto Scaling Group

An Auto Scaling Group was created using the previously created Launch Template.

    Auto Scaling Group:
    aws-project

    Launch Template:
    aws-project-launch-template

The Auto Scaling Group was configured to use the two private subnets.

    Private Subnet:
    ap-south-1a

    Private Subnet:
    ap-south-1b

### Capacity Configuration

    Desired Capacity:
    2

    Minimum Capacity:
    1

    Maximum Capacity:
    4

The desired capacity of two means that the Auto Scaling Group attempts to maintain two application instances.

### Architecture

    Auto Scaling Group
           |
           +---- EC2 #1
           |
           +---- EC2 #2

The instances are distributed across two Availability Zones.

### Screenshot

![Auto Scaling Group](screenshots/10-auto-scaling-group.png)

---

# 11. Verify Private EC2 Instances

The Auto Scaling Group launched two EC2 instances.

## Private EC2 #1

    Instance:
    Private EC2 #1

    Availability Zone:
    ap-south-1a

    Private IP:
    10.0.131.226

    Instance Type:
    t3.micro

    Operating System:
    Ubuntu 26.04 LTS

## Private EC2 #2

    Instance:
    Private EC2 #2

    Availability Zone:
    ap-south-1b

    Private IP:
    10.0.144.215

    Instance Type:
    t3.micro

    Operating System:
    Ubuntu 26.04 LTS

Both instances are located in private subnets and do not have public IPv4 addresses.

### Screenshot

![Private EC2 Instances](screenshots/11-private-ec2-instances.png)

---

# 12. Create Bastion Host

A Bastion Host was created to provide administrative SSH access to the private EC2 instances.

### Bastion Host Configuration

    Name:
    bastion-host-for-project

    Operating System:
    Ubuntu 26.04 LTS

    Instance Type:
    t3.micro

    Key Pair:
    my-key

    VPC:
    aws-project-vpc-vpc

    Subnet:
    Public Subnet

The Bastion Host has a public IP address.

The private EC2 instances do not have public IP addresses.

Therefore, the administrator connects through the Bastion Host.

### SSH Architecture

    Administrator Laptop
            |
            | SSH :22
            v
    Bastion Host
            |
            | SSH :22
            +--------------------> Private EC2 #1
            |
            +--------------------> Private EC2 #2

### Screenshot

![Bastion Host](screenshots/12-bastion-host.png)

---

# 13. Copy PEM Key to Bastion Host

The EC2 key pair used in this project was:

    my-key

The PEM file was stored on the Windows machine:

    C:\Users\Zarthi\Downloads\my-key.pem

Because Git Bash was used, the path was represented as:

    /c/Users/Zarthi/Downloads/my-key.pem

The PEM key was copied to the Bastion Host using SCP.

Example:

    scp -i /c/Users/Zarthi/Downloads/my-key.pem \
    /c/Users/Zarthi/Downloads/my-key.pem \
    ubuntu@<BASTION_PUBLIC_IP>:/home/ubuntu/

After connecting to the Bastion Host:

    ls

The key was verified:

    my-key.pem

The key permissions were configured:

    chmod 400 my-key.pem

The private key is required to establish SSH connections to the private EC2 instances.

### Security Note

The PEM file must NEVER be uploaded to GitHub.

The repository should contain:

    .gitignore

with:

    *.pem

### Screenshot

![PEM Key on Bastion Host](screenshots/13-pem-key-bastion.png)

---

# 14. SSH from Bastion Host to Private EC2 #1

The first private EC2 instance:

    Private IP:
    10.0.131.226

From the Bastion Host, the following command was used:

    ssh -i my-key.pem ubuntu@10.0.131.226

The SSH connection was successful.

The shell changed to the private EC2 instance:

    ubuntu@ip-10-0-131-226:~$

This confirmed that the Bastion Host could communicate with the private EC2 instance.

### Screenshot

![SSH Private EC2 1](screenshots/14-ssh-private-ec2-1.png)

---

# 15. SSH from Bastion Host to Private EC2 #2

The second private EC2 instance:

    Private IP:
    10.0.144.215

From the Bastion Host:

    ssh -i my-key.pem ubuntu@10.0.144.215

The SSH connection was successful.

The shell changed to:

    ubuntu@ip-10-0-144-215:~$

This confirmed connectivity from the Bastion Host to the second private EC2 instance.

### Screenshot

![SSH Private EC2 2](screenshots/15-ssh-private-ec2-2.png)

---

# 16. Install Web Application on Private EC2 #1

The first private EC2 instance hosts Server 1.

    EC2:
    Private EC2 #1

    Availability Zone:
    ap-south-1a

    Private IP:
    10.0.131.226

An `index.html` file was created.

The application contains information identifying the server, Availability Zone, and private IP address.

The application was started using Python's built-in HTTP server.

    python3 -m http.server 8000

Expected output:

    Serving HTTP on 0.0.0.0 port 8000

The application is listening on:

    Port 8000

### Screenshot

![Server 1 Application](screenshots/16-server-1-application.png)

---

# 17. Install Web Application on Private EC2 #2

The second private EC2 instance hosts Server 2.

    EC2:
    Private EC2 #2

    Availability Zone:
    ap-south-1b

    Private IP:
    10.0.144.215

An `index.html` file was created.

The application was started using:

    python3 -m http.server 8000

The application is listening on:

    Port 8000

The Server 2 page displays:

    EC2 Server: Private EC2 #2
    Availability Zone: ap-south-1b
    Private IP: 10.0.144.215

### Screenshot

![Server 2 Application](screenshots/17-server-2-application.png)

---

# 18. Create Target Group

A Target Group was created for the two private EC2 instances.

AWS Console:

    EC2
    → Target Groups
    → Create Target Group

### Target Group Configuration

    Target Type:
    Instances

    Target Group Name:
    aws-project

    Protocol:
    HTTP

    Port:
    8000

    Protocol Version:
    HTTP1

    VPC:
    aws-project-vpc-vpc

### Health Check

    Protocol:
    HTTP

    Path:
    /

    Port:
    traffic-port

    Success Code:
    200

The two private EC2 instances were registered as targets.

    Target 1:
    Private EC2 #1
    10.0.131.226
    Port 8000

    Target 2:
    Private EC2 #2
    10.0.144.215
    Port 8000

### Target Group Architecture

    Target Group
        |
        +---- Private EC2 #1
        |
        +---- Private EC2 #2

The Target Group performs health checks to determine whether the instances are healthy.

### Screenshot

![Target Group](screenshots/18-target-group.png)

---

# 19. Verify Target Health

The Target Group was verified after registering both EC2 instances.

The final state showed:

    Total Targets:
    2

    Healthy:
    2

    Unhealthy:
    0

This confirmed that both private EC2 instances were responding correctly on port 8000.

### Screenshot

![Healthy Targets](screenshots/19-healthy-targets.png)

---

# 20. Create Application Load Balancer

An Application Load Balancer was created.

AWS Console:

    EC2
    → Load Balancers
    → Create Load Balancer
    → Application Load Balancer

### ALB Configuration

    Name:
    aws-project-lb

    Type:
    Application Load Balancer

    Scheme:
    Internet-facing

    IP Address Type:
    IPv4

    VPC:
    aws-project-vpc-vpc

The ALB was deployed across two public subnets:

    ap-south-1a

    ap-south-1b

### Listener

    Protocol:
    HTTP

    Port:
    80

The listener forwards requests to:

    Target Group:
    aws-project

### Screenshot

![Application Load Balancer](screenshots/20-application-load-balancer.png)

---

# 21. Application Load Balancer Traffic Flow

The complete application traffic flow is:

    User
      |
      | HTTP :80
      v
    Internet
      |
      v
    Application Load Balancer
      |
      | HTTP :8000
      v
    Target Group
      |
      +------------------+
      |                  |
      v                  v
    Private EC2 #1     Private EC2 #2
    10.0.131.226       10.0.144.215
    ap-south-1a        ap-south-1b

The user never directly accesses the private EC2 instances.

The Application Load Balancer is the public entry point.

---

# 22. Verify Application Load Balancer

After creating the Application Load Balancer, the ALB became:

    Active

AWS generated an ALB DNS name.

The ALB DNS name created for this project was:

    aws-project-lb-526567170.ap-south-1.elb.amazonaws.com

The application was accessed using:

    http://aws-project-lb-526567170.ap-south-1.elb.amazonaws.com/

### Screenshot

![ALB Active](screenshots/21-alb-active.png)

---

# 23. Test Server 1 Through ALB

The ALB successfully forwarded a request to Private EC2 #1.

The browser displayed:

    AWS Highly Available Web Application

    Server 1

    EC2 Server: Private EC2 #1
    Availability Zone: ap-south-1a
    Private IP: 10.0.131.226

This confirmed:

    Internet
      |
      v
    ALB
      |
      v
    Target Group
      |
      v
    Private EC2 #1

### Screenshot

![ALB Server 1](screenshots/22-alb-server-1.png)

---

# 24. Test Server 2 Through ALB

The ALB also successfully forwarded requests to Private EC2 #2.

The browser displayed:

    AWS Highly Available Web Application

    Server 2

    EC2 Server: Private EC2 #2
    Availability Zone: ap-south-1b
    Private IP: 10.0.144.215

This confirmed:

    Internet
      |
      v
    ALB
      |
      v
    Target Group
      |
      v
    Private EC2 #2

### Screenshot

![ALB Server 2](screenshots/23-alb-server-2.png)

---

# 25. Test Load Balancing

The Application Load Balancer was tested using Git Bash.

Command:

    for i in {1..10}; do
        curl --no-keepalive http://aws-project-lb-526567170.ap-south-1.elb.amazonaws.com/ | grep -E "Server [12]|ap-south-1"
    done

The responses showed that requests could reach both servers.

Example:

    Server 1
    ap-south-1a
    10.0.131.226

and:

    Server 2
    ap-south-1b
    10.0.144.215

This demonstrates that the Application Load Balancer is distributing requests between healthy targets.

### Screenshot

![Load Balancing Test](screenshots/24-load-balancing-test.png)

---

# 26. Verify Target Group After ALB Testing

After testing the ALB, the Target Group was checked again.

The target health status showed:

    Private EC2 #1
    Healthy

    Private EC2 #2
    Healthy

Therefore:

    Total Targets:
    2

    Healthy:
    2

    Unhealthy:
    0

### Screenshot

![Target Group Healthy](screenshots/25-target-group-healthy.png)

---

# 27. High Availability Design

The application is distributed across two Availability Zones.

    AWS Region: ap-south-1

            |
            +----------------------+
            |                      |
            v                      v
       ap-south-1a            ap-south-1b
            |                      |
            v                      v
      Private EC2 #1         Private EC2 #2
      10.0.131.226           10.0.144.215

The Application Load Balancer operates across both Availability Zones.

This architecture avoids placing the complete application in only one Availability Zone.

---

# 28. Failure Scenario

Suppose Private EC2 #1 becomes unavailable.

Before failure:

    ALB
     |
     +---- EC2 #1 ---- Healthy
     |
     +---- EC2 #2 ---- Healthy

After EC2 #1 becomes unavailable:

    ALB
     |
     +---- EC2 #1 ---- Unhealthy
     |
     +---- EC2 #2 ---- Healthy

The Target Group health check detects that EC2 #1 is unhealthy.

The ALB stops sending new requests to the unhealthy target.

Traffic continues toward the healthy target.

The Auto Scaling Group can launch a replacement instance according to its configuration.

### Screenshot

![High Availability Test](screenshots/26-high-availability-test.png)

---

# 29. Complete Network Traffic

## Incoming Traffic

    User
      |
      | HTTP :80
      v
    Internet
      |
      v
    Internet Gateway
      |
      v
    Public Subnet
      |
      v
    Application Load Balancer
      |
      | HTTP :8000
      v
    Target Group
      |
      +------------------+
      |                  |
      v                  v
    Private EC2 #1     Private EC2 #2

---

## Private EC2 Outbound Traffic

    Private EC2
          |
          v
    Private Route Table
          |
          v
    NAT Gateway
          |
          v
    Public Subnet
          |
          v
    Internet Gateway
          |
          v
       Internet

---

## Administrative SSH Traffic

    Administrator
          |
          | SSH :22
          v
    Bastion Host
          |
          | SSH :22
          +--------------------> Private EC2 #1
          |
          +--------------------> Private EC2 #2

---

# 30. Security Architecture

The application follows a private application server model.

The application EC2 instances are deployed inside private subnets.

The public-facing component is the Application Load Balancer.

The Bastion Host provides administrative SSH access.

Conceptually:

    INTERNET
       |
       | HTTP
       v
    ALB
       |
       | HTTP :8000
       v
    PRIVATE EC2

Administrative access:

    ADMINISTRATOR
       |
       | SSH
       v
    BASTION HOST
       |
       | SSH
       v
    PRIVATE EC2

This separation reduces direct exposure of the application servers.

---

# 31. Complete Architecture Flow

    +-------------------------------------------------------------+
    |                         AWS REGION                           |
    |                         ap-south-1                           |
    |                                                             |
    |   +-----------------------------------------------------+   |
    |   |                    VPC                              |   |
    |   |                 10.0.0.0/16                         |   |
    |   |                                                     |   |
    |   |  +----------------------+  +----------------------+ |   |
    |   |  |    ap-south-1a      |  |    ap-south-1b      | |   |
    |   |  |                      |  |                      | |   |
    |   |  |  Public Subnet      |  |  Public Subnet      | |   |
    |   |  |      |               |  |      |               | |   |
    |   |  |      +---- ALB ------+--+------+- ALB          | |   |
    |   |  |      |               |  |      |               | |   |
    |   |  |  NAT Gateway         |  |  NAT Gateway         | |   |
    |   |  |                      |  |                      | |   |
    |   |  |  Private Subnet      |  |  Private Subnet      | |   |
    |   |  |      |               |  |      |               | |   |
    |   |  |   EC2 #1             |  |   EC2 #2             | |   |
    |   |  | 10.0.131.226         |  | 10.0.144.215         | |   |
    |   |  |                      |  |                      | |   |
    |   |  +----------------------+  +----------------------+ |   |
    |   |                                                     |   |
    |   +-----------------------------------------------------+   |
    |                                                             |
    +-------------------------------------------------------------+

---

# 32. AWS Services Used

| AWS Service | Purpose |
|---|---|
| Amazon VPC | Creates the isolated network |
| Subnets | Separates public and private resources |
| Availability Zones | Provides Multi-AZ deployment |
| Internet Gateway | Provides internet connectivity for public resources |
| NAT Gateway | Provides outbound internet access for private resources |
| VPC Endpoint | Provides private access to S3 |
| Route Tables | Control network traffic paths |
| Security Groups | Control inbound and outbound traffic |
| Amazon EC2 | Hosts the web application |
| Launch Template | Defines EC2 launch configuration |
| Auto Scaling Group | Manages EC2 capacity |
| Bastion Host | Provides SSH access to private instances |
| Target Group | Registers and health-checks EC2 targets |
| Application Load Balancer | Distributes incoming HTTP traffic |

---

# 33. Project Resource Summary

    AWS Region:
    ap-south-1

    VPC:
    aws-project-vpc-vpc

    VPC CIDR:
    10.0.0.0/16

    Public Subnets:
    2

    Private Subnets:
    2

    NAT Gateways:
    2

    VPC Endpoint:
    S3 Gateway

    Security Group:
    aws-project-sg

    Launch Template:
    aws-project-launch-template

    Auto Scaling Group:
    aws-project

    Instance Type:
    t3.micro

    Operating System:
    Ubuntu 26.04 LTS

    Key Pair:
    my-key

    Bastion Host:
    bastion-host-for-project

    Target Group:
    aws-project

    Target Port:
    8000

    Application Load Balancer:
    aws-project-lb

    ALB Scheme:
    Internet-facing

    ALB Listener:
    HTTP :80

    Private EC2 #1:
    10.0.131.226
    ap-south-1a

    Private EC2 #2:
    10.0.144.215
    ap-south-1b

---

# 34. Repository Structure

The GitHub repository is organized as:

    aws-highly-available-web-application/
    |
    +-- README.md
    |
    +-- architecture/
    |     |
    |     +-- architecture-diagram.png
    |
    +-- screenshots/
    |     |
    |     +-- 01-region.png
    |     +-- 02-vpc-creation.png
    |     +-- 03-vpc-subnets.png
    |     +-- 04-internet-gateway.png
    |     +-- 05-route-tables.png
    |     +-- 06-nat-gateways.png
    |     +-- 07-s3-endpoint.png
    |     +-- 08-security-group.png
    |     +-- 09-launch-template.png
    |     +-- 10-auto-scaling-group.png
    |     +-- 11-private-ec2-instances.png
    |     +-- 12-bastion-host.png
    |     +-- 13-pem-key-bastion.png
    |     +-- 14-ssh-private-ec2-1.png
    |     +-- 15-ssh-private-ec2-2.png
    |     +-- 16-server-1-application.png
    |     +-- 17-server-2-application.png
    |     +-- 18-target-group.png
    |     +-- 19-healthy-targets.png
    |     +-- 20-application-load-balancer.png
    |     +-- 21-alb-active.png
    |     +-- 22-alb-server-1.png
    |     +-- 23-alb-server-2.png
    |     +-- 24-load-balancing-test.png
    |     +-- 25-target-group-healthy.png
    |     +-- 26-high-availability-test.png
    |
    +-- application/
          |
          +-- server-1/
          |     |
          |     +-- index.html
          |
          +-- server-2/
                |
                +-- index.html

---

# 35. .gitignore

The repository should contain a `.gitignore` file.

    *.pem
    .env
    .DS_Store

The AWS private key must not be committed to GitHub.

---

# 36. What This Project Demonstrates

This project demonstrates practical understanding of AWS networking and application architecture.

The most important concept is understanding how traffic moves through the system.

### User to Application

    User
      |
      v
    Internet
      |
      v
    Application Load Balancer
      |
      v
    Target Group
      |
      v
    Private EC2
      |
      v
    Web Application

### Private EC2 to Internet

    Private EC2
      |
      v
    Private Route Table
      |
      v
    NAT Gateway
      |
      v
    Internet Gateway
      |
      v
    Internet

### Administrator to Private EC2

    Administrator
      |
      v
    Bastion Host
      |
      v
    Private EC2

---

# 37. Key Concepts Learned

Through this project, the following AWS concepts were practiced:

- Amazon VPC
- CIDR
- Subnets
- Public Subnets
- Private Subnets
- Availability Zones
- Route Tables
- Internet Gateway
- NAT Gateway
- VPC Endpoint
- Security Groups
- Amazon EC2
- Ubuntu
- SSH
- SCP
- Bastion Host
- Launch Templates
- Auto Scaling Groups
- Application Load Balancer
- Target Groups
- Health Checks
- HTTP
- Multi-AZ Architecture
- High Availability
- Load Balancing
- Private Application Servers

---

# 38. Project Completion Checklist

- [x] Created AWS VPC
- [x] Used VPC and more option
- [x] Configured VPC CIDR `10.0.0.0/16`
- [x] Created two Availability Zones
- [x] Created two public subnets
- [x] Created two private subnets
- [x] Configured Internet Gateway
- [x] Configured two NAT Gateways
- [x] Configured S3 Gateway Endpoint
- [x] Verified route tables
- [x] Created Security Group
- [x] Created Launch Template
- [x] Selected Ubuntu 26.04 LTS
- [x] Selected t3.micro
- [x] Configured my-key
- [x] Created Auto Scaling Group
- [x] Configured desired capacity of 2
- [x] Configured minimum capacity of 1
- [x] Configured maximum capacity of 4
- [x] Launched private EC2 instances
- [x] Verified EC2 instance health
- [x] Created Bastion Host
- [x] Copied PEM key to Bastion Host
- [x] Connected to Private EC2 #1
- [x] Connected to Private EC2 #2
- [x] Installed application on Private EC2 #1
- [x] Installed application on Private EC2 #2
- [x] Started application on port 8000
- [x] Created Target Group
- [x] Registered two EC2 targets
- [x] Configured HTTP health checks
- [x] Verified two healthy targets
- [x] Created Application Load Balancer
- [x] Configured Internet-facing ALB
- [x] Configured HTTP listener on port 80
- [x] Forwarded traffic to Target Group
- [x] Tested Server 1 through ALB
- [x] Tested Server 2 through ALB
- [x] Tested load balancing
- [x] Verified Multi-AZ deployment
- [x] Verified high availability architecture

---

# 39. Final Result

The project successfully demonstrates a highly available web application architecture on AWS.

The final request path is:

    USER
      |
      v
    INTERNET
      |
      | HTTP :80
      v
    APPLICATION LOAD BALANCER
    aws-project-lb
      |
      | HTTP :8000
      v
    TARGET GROUP
    aws-project
      |
      +--------------------------+
      |                          |
      v                          v
    PRIVATE EC2 #1          PRIVATE EC2 #2
    ap-south-1a             ap-south-1b
    10.0.131.226            10.0.144.215
    Port 8000               Port 8000
      |                          |
      +------------+-------------+
                   |
                   v
           AUTO SCALING GROUP
              aws-project

Administrative access:

    ADMINISTRATOR
          |
          | SSH :22
          v
    BASTION HOST
    bastion-host-for-project
          |
          +--------------------> PRIVATE EC2 #1
          |
          +--------------------> PRIVATE EC2 #2

The application servers are protected inside private subnets, while the Application Load Balancer provides the public entry point.

The two EC2 instances are distributed across two Availability Zones, allowing the architecture to continue serving application traffic when one instance becomes unavailable, assuming remaining capacity and dependencies are healthy.

---

# 🎯 Conclusion

This project provided hands-on experience building a complete AWS network and application architecture from the ground up.

Instead of deploying EC2 instances directly to the public internet, the application servers were placed in private subnets.

An Internet-facing Application Load Balancer was used as the public entry point.

A Target Group was used to register and health-check the application servers.

An Auto Scaling Group was used to manage the EC2 instances.

A Bastion Host was used for administrative SSH access to the private servers.

NAT Gateways were used to provide outbound internet connectivity from the private subnets.

The application was deployed across two Availability Zones to demonstrate high availability.

The final architecture demonstrates how AWS networking, compute, security, scaling, and load balancing services work together to build a resilient cloud application.

---

# 🏁 Final Architecture Diagram

![Final AWS Architecture](architecture/architecture-diagram.png)

---

## Project Status

    STATUS: COMPLETED

    AWS REGION: ap-south-1

    ARCHITECTURE: MULTI-AZ

    APPLICATION: HIGHLY AVAILABLE WEB APPLICATION

    LOAD BALANCER: aws-project-lb

    AUTO SCALING GROUP: aws-project

    TARGET GROUP: aws-project

    VPC: aws-project-vpc-vpc
