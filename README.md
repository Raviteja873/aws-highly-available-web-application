# AWS Highly Available Web Application

> A hands-on AWS project demonstrating a highly available web application using VPC, public and private subnets, NAT Gateway, Bastion Host, EC2, Auto Scaling Group, Target Group, and Application Load Balancer.

---

## 📌 Project Overview

In this project, I built a highly available web application architecture on AWS.

The application servers are deployed inside **private subnets** across **two Availability Zones**.

An **Internet-facing Application Load Balancer (ALB)** receives requests from users and distributes the traffic to the private EC2 instances.

A **Bastion Host** is used to securely connect to the private EC2 instances for administration and application deployment.

The architecture is designed to provide:

- High availability
- Private application servers
- Multi-AZ deployment
- Load balancing
- Auto Scaling
- Secure administrative access
- Controlled network traffic

---

# 🏗️ Architecture

```text
                         INTERNET
                            |
                            |
                            v
                +-----------------------+
                | Application Load      |
                | Balancer              |
                | aws-project-lb        |
                | Internet-facing       |
                +-----------+-----------+
                            |
                            |
                       Target Group
                        HTTP :8000
                            |
              +-------------+-------------+
              |                           |
              v                           v
       +-------------+              +-------------+
       | Private EC2 |              | Private EC2 |
       | Server 1    |              | Server 2    |
       |             |              |             |
       | ap-south-1a |              | ap-south-1b |
       | 10.0.131.226|              | 10.0.144.215|
       +-------------+              +-------------+
              |                           |
              |                           |
              v                           v
        Private Route Table        Private Route Table
              |                           |
              v                           v
        NAT Gateway                NAT Gateway
              |                           |
              +-------------+-------------+
                            |
                            v
                    Internet Gateway
                            |
                            v
                         INTERNET
