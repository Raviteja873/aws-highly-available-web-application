# AWS Highly Available Web Application

A hands-on AWS project demonstrating a highly available web application deployed across multiple Availability Zones using Amazon VPC, EC2, Auto Scaling Group, Application Load Balancer, Target Group, NAT Gateway, Bastion Host, Security Groups, and private subnets.

---

## 📌 Project Overview

This project demonstrates how to build a highly available web application on AWS.

The application servers are deployed on Amazon EC2 instances inside private subnets across two Availability Zones.

An Internet-facing Application Load Balancer receives HTTP requests from users and forwards the requests to healthy EC2 instances through a Target Group.

The EC2 instances are managed by an Auto Scaling Group.

A Bastion Host is deployed in a public subnet and is used to securely connect to the private EC2 instances through SSH.

The architecture is designed to demonstrate:

- AWS VPC networking
- Public and private subnets
- Multi-AZ architecture
- Internet Gateway
- NAT Gateway
- VPC Endpoint
- Security Groups
- Amazon EC2
- Launch Template
- Auto Scaling Group
- Bastion Host
- Application Load Balancer
- Target Group
- Health Checks
- SSH connectivity
- HTTP traffic
- High Availability
- Load Balancing

---

# 🏗️ Architecture Diagram

![AWS Architecture Diagram](architecture/architecture-diagram.png)

### High-Level Architecture

```text
                              INTERNET
                                  |
                                  | HTTP :80
                                  v
                    +-----------------------------+
                    | Application Load Balancer   |
                    |       aws-project-lb        |
                    |       Internet-facing       |
                    +-------------+---------------+
                                  |
                                  | HTTP :8000
                                  v
                    +-----------------------------+
                    |       Target Group          |
                    |         aws-project         |
                    +-------------+---------------+
                                  |
                    +-------------+-------------+
                    |                           |
                    v                           v
          +-------------------+       +-------------------+
          |   Private EC2 #1  |       |   Private EC2 #2  |
          |   ap-south-1a     |       |   ap-south-1b     |
          |   10.0.131.226    |       |   10.0.144.215    |
          |   Port 8000       |       |   Port 8000       |
          +-------------------+       +-------------------+
                    |                           |
                    |                           |
                    v                           v
             NAT Gateway 1               NAT Gateway 2
                    |                           |
                    +-------------+-------------+
                                  |
                                  v
                              INTERNET


                 ADMINISTRATOR / DEVELOPER
                           |
                           | SSH :22
                           v
                  +----------------------+
                  |     Bastion Host     |
                  | bastion-host-for-    |
                  |      project         |
                  +----------+-----------+
                             |
                    SSH :22  |
                 +-----------+-----------+
                 |                       |
                 v                       v
        Private EC2 #1           Private EC2 #2
        10.0.131.226              10.0.144.215
