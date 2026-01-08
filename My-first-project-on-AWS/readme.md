# The "First Project" Titles 
This project implements a production-grade AWS architecture designed for high security and availability 
# overview of project 
**What:** This production-grade project establishes a secure AWS environment using a VPC with public and private subnets distributed across two Availability Zones,. 

**Why:** It is built to ensure **high availability**, preventing site outages if a single data center fails, and to protect application data by isolating servers from the public internet,.

**How:** Public traffic is managed through an **Application Load Balancer**, while a **NAT Gateway** masks private IP addresses to allow secure outbound communication and Finally, a **Bastion host** is placed in the public subnet to serve as a secure, auditable "jump point" for administrative access to the isolated private instances.

# Architecture Diagram 
![Project Screenshot](images/unnamed.png)

# Step-by-Step Implementation
# Step 1: Create VPC
Go to VPC → Create VPC.
Select = VPC & more: This helps in understanding the process of creating imagination on understanding the concept of VPC.
Name: project-vpc (any name).
IPv4 CIDR block: 10.0.0.0/16 - This is for the range of VPC.
Selection of AZ depending on needs: 1, 2, or 3.
Selecting subnets in this VPC:

public-subnet-1, public-subnet-2 in different AZs.
private-subnet-1, private-subnet-2 in different AZs.

Selecting NAT gateway: Depending on your need, but I have selected 1 per AZ.
Click on Create VPC: This will automatically create IGW, Route tables, create subnets, allocate Elastic IP addresses, etc. You will understand when you create it.

# Step 2: Create Security Groups
Bastion host SG (bastion-sg)

Inbound:

SSH (22) from anywhere.

Outbound: Allow all traffic (default).

Private app SG (private-app-sg)

Inbound:

SSH (22) from bastion-sg.

HTTP (80) from ALB SG 

Custom TCP   8000

Outbound: Allow all traffic.


# Step 3: Create Launch Template for Auto Scaling
Go to EC2 → Launch templates → Create launch template.

Fill basic details:

Template name: web-app-template.

Template version description: ubuntu-web-template-v1.

Choose AMI: Ubuntu Server (free tier eligible).

Choose Instance type: t2.micro (or similar).

Configure:

Key pair: select an existing key pair.

setect the SG of Auto Scaling group 

# Step 4: Create Auto Scaling Group
Go to EC2 → Auto Scaling groups → Create Auto Scaling group.
​

Choose launch template: select web-app-template.

Name the ASG (for example, web-asg).

Network: VPC: select your VPC.

Subnets: select all private subnets where app instances should run. depending your need i have selected private the resion is i need to keep my website safe if i need any updates form internet NAT will take care of it

Load balancing:

Skip for now if ALB is not yet created (can be attached later), or select Attach to an existing load balancer once ALB and target group are ready.
​
Group size:

Desired capacity: 2.

Minimum capacity: 1.

Maximum capacity: 3 (or more as needed).

Scaling policies (example):

Target tracking policy on Average CPU utilization.

Target value: 50–60%.

Review and create the Auto Scaling group.

Result: EC2 instances are automatically launched in private subnets using the template, with capacity controlled by scaling policies

# Step 5: Create Bastion Host in Public Subnet
Go to EC2 → Instances → Launch instance.

Name: bastion-host.

AMI: Ubuntu Server (same region as VPC).

Instance type: t2.micro (or similar).

Key pair: select the same key pair used in the template (or another known key).

Network settings:

VPC: same VPC.

Subnet: choose a public subnet.

Auto-assign public IP: Enable.

Security group: bastion-sg.

Launch the instance and wait until status is running.
​

# Step 6: Copy Private Key to Bastion Host
From your local machine terminal:

Ensure the key file (for connecting to private instances) exists on your local system.

Use scp to copy the private key to the Bastion host:

bash
scp -i <path-to-bastion-key.pem> <path-to-app-key.pem> ubuntu@<bastion-public-ip>:/home/ubuntu/
On the Bastion host, set correct permissions:

bash
chmod 400 /home/ubuntu/<app-key.pem>
Now the Bastion host can SSH into private instances using this key.
​

# Step 7: Connect to Private Instances via Bastion
SSH into the Bastion host from your local system:

bash
ssh -i <path-to-bastion-key.pem> ubuntu@<private-instance-ip>
From the Bastion host, SSH into a private instance:

bash
ssh -i <app-key.pem> ubuntu@<private-instance-ip>
If user data did not start the Python server, you can manually run:

bash
cd /home/ubuntu
python3 -m http.server 8000
Verify that the instance is serving HTTP traffic internally on port 8000.
​

Step 8: Create Application Load Balancer
Go to EC2 → Load balancers → Create load balancer → Application Load Balancer.
​

Basic configuration:

Name: web-alb.

Scheme: internet-facing.

IP type: IPv4.

Network mapping:

VPC: same VPC.

Subnets: choose public subnets in at least two AZs.

Security groups: attach alb-sg.

Listeners and routing:

Listener: HTTP port 80 (or 8000 if you prefer).

Create a new target group:

Type: Instances.

Protocol/Port: HTTP : 8000.

Health check path: /.
​

Register targets:

Select the private EC2 instances (or attach the ASG to this target group so that new instances are auto‑registered).

Create the ALB.

Result: External traffic hits the ALB, which forwards HTTP requests on port 8000 to healthy instances in private subnets.
​
# Step 9: Attach Auto Scaling Group to Target Group (If Not Done Earlier)
Open your Auto Scaling group → Edit.

In Load balancing, choose Attach to a new or existing load balancer.

Select the Application Load Balancer and the target group created in the previous step.

Save changes.

# Step 10: Test the Setup
Go to EC2 → Load balancers, select web-alb, and copy the DNS name.

Open it in a browser:

You should see the sample HTML page or Python HTTP server response coming from one of the private instances.