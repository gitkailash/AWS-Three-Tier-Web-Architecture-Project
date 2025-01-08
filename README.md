# AWS-Three-Tier-Web-Architecture-Project

## Overview
This project implements a highly scalable and secure three-tier web architecture on AWS. The architecture consists of a Web Tier, App Tier, and Database Tier, deployed using best practices for networking, security, and automation.

---

## Table of Contents
1. [Setup](#setup)
2. [Networking and Security](#networking-and-security)
3. [Database Deployment](#database-deployment)
4. [App Tier Setup](#app-tier-setup)
5. [Internal Load Balancer and Auto Scaling](#internal-load-balancer-and-auto-scaling)
6. [Web Tier Setup](#web-tier-setup)
7. [Web Tier Deployment](#web-tier-deployment)
8. [Architecture Diagram](#architecture-diagram)

---

## Setup
### Step 1: Clone Repository
```bash
git clone https://github.com/aws-samples/aws-three-tier-web-architecture-workshop.git
```

### Step 2: Create S3 Bucket
- Create an S3 bucket to store application code and configuration files.

### Step 3: Create IAM Role
- **Role Name:** `EC2_S3_ReadOnly_SSM_Role`
- **Policies:**
  - AmazonS3ReadOnlyAccess
  - AmazonSSMManagedInstanceCore

---

## Networking and Security

### Step 1: Create VPC
- CIDR Block: `10.0.0.0/16`

### Step 2: Create Subnets
| Subnet Name                 | CIDR Block    | Type      | Availability Zone |
|-----------------------------|---------------|-----------|-------------------|
| public-web-subnet-AZ-1      | 10.0.0.0/20   | Public    | us-east-1a        |
| public-web-subnet-AZ-2      | 10.0.16.0/20  | Public    | us-east-1b        |
| private-app-subnet-AZ-1     | 10.0.32.0/20  | Private   | us-east-1a        |
| private-app-subnet-AZ-2     | 10.0.48.0/20  | Private   | us-east-1b        |
| private-db-subnet-AZ-1      | 10.0.64.0/20  | Private   | us-east-1a        |
| private-db-subnet-AZ-2      | 10.0.80.0/20  | Private   | us-east-1b        |

### Step 3: Create Internet Gateway
- Attach the Internet Gateway to the VPC.

### Step 4: Create NAT Gateways
- Create a NAT Gateway in each Availability Zone with Elastic IPs.

### Step 5: Route Tables
- **Public Route Table:**
  - Route: `0.0.0.0/0` to Internet Gateway
  - Associate with Public Subnets.
- **Private Route Tables:**
  - Route: `0.0.0.0/0` to NAT Gateway (specific to AZ)
  - Associate with Private Subnets.

### Step 6: Security Groups
| Security Group Name    | Rules                                                                                   |
|------------------------|-----------------------------------------------------------------------------------------|
| InternetFacing-LB-SG   | Allow HTTP (80) from anywhere (IPv4 & IPv6).                                            |
| WebTier-SG             | Allow HTTP from `InternetFacing-LB-SG` and your IP.                                    |
| Internal-LB-SG         | Allow HTTP from `WebTier-SG`.                                                          |
| AppTier-SG             | Allow custom TCP (4000) from `Internal-LB-SG` and your IP.                             |
| DB-SG                  | Allow MySQL/Aurora (3306) from `AppTier-SG` and your IP.                               |

---

## Database Deployment
### Step 1: Create Subnet Group
- Name: `three-tier-db-subnet-group`
- Subnets: Private DB Subnets in both AZs.

### Step 2: Create Aurora Database
- **Choose a database creation method:** Standard create
- **Engine options:** Aurora (MySQL Compatible)
- **Templates:** Dev/Test
- **Engine:** Aurora (MySQL Compatible)
- **Credentials:**
  - Username: `username`
  - Password: `password`
- **Multi-AZ Deployment:** Enabled
- **Virtual private cloud (VPC):** `Your VPC`
- **DB subnet group:** `three-tier-db-subnet-group`
- **VPC Security Group:** `DB-SG`


---

## App Tier Setup
### Step 1: Launch AppServer EC2 Instance
- **AMI:** Amazon Linux
- **Instance Type:** t2.micro
- **Edit Network Setting**:
  - **VPC:** `Your VPC`
  - **Subnet:** `private-app-subnet-AZ-1`
  - **Security Group:** `AppTier-SG`
- **Advance Setting:**
  - **IAM Role:** `EC2_S3_ReadOnly_SSM_Role`
- **Lunch instance** 

### Step 2: Connect to instance  (Session Manager)
Instructions for Setting Up the Application
```bash
# Switch to EC2 user
sudo -su ec2-user

# Test network connectivity
ping 8.8.8.8

# Download the MySQL repository package
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm 

# Install the MySQL repository package
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y

# Import the MySQL GPG key
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023

# Install the MySQL client and server
sudo dnf install mysql-community-client -y
sudo dnf install mysql-community-server -y

# Verify MySQL installation
mysql --version

# Connect to RDS instance (replace <RDS_Endpoint>, <username>, and enter password when prompted)
mysql -h <RDS_Endpoint> -u <username> -p

# SQL commands to create a database and table, and insert data
show databases;
create DATABASE webappdb;
show databases;
use webappdb;
CREATE TABLE IF NOT EXISTS transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    amount DECIMAL(10, 2) NOT NULL,
    description VARCHAR(255) NOT NULL
);
show tables;
INSERT INTO transactions (amount, description)
VALUES
    (100.00, 'Payment for services'),
    (250.50, 'Refund for returned item'),
    (500.00, 'Deposit to account');
SELECT * FROM transactions;

# Exit MySQL
exit;

# Edit the `DBConfig.js` File
1. Navigate to the `app-tier` directory.
2. Open the `DBConfig.js` file in a text editor.
3. Replace the placeholders with your database configurations:

   ```javascript
   module.exports = Object.freeze({
       DB_HOST : 'database-1.cluster-cbyeee0guq2d.us-east-1.rds.amazonaws.com', // Replace with your RDS endpoint
       DB_USER : 'user', // Replace with your RDS username
       DB_PWD : 'password', // Replace with your RDS password
       DB_DATABASE : 'webappdb' // Replace with your database name
   });


# Install NVM (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
source ~/.bashrc

# Check NVM version
nvm -v

# Install and use Node.js version 16
nvm install 16
nvm use 16

# Install PM2 globally
npm install -g pm2

# Navigate to the parent directory and verify structure
cd ../
pwd
ls -rlt

# Copy the app-tier folder from S3 to the EC2 instance
sudo aws s3 cp s3://aws-three-tier-web-architecture-workshop-2025/app-tier/ app-tier --recursive

# Verify the copied files
ls -rlt

# Navigate to the app-tier directory and start the application
cd app-tier/
pm2 start index.js

# Check the PM2 process list and logs
pm2 list
pm2 logs

# Change ownership of the app-tier directory
sudo chown -R ec2-user:ec2-user /usr/app-tier

# Install required Node.js dependencies
npm install

# Set custom npm directory and update PATH
npm config set prefix ~/.npm-global
export PATH=$PATH:~/.npm-global/bin

# Save the PM2 process to start on system boot
pm2 save
pm2 startup

# Run the command suggested by PM2 for enabling startup (replace with the output)
# Example:
sudo env PATH=$PATH:/home/ec2-user/.nvm/versions/node/v16.20.2/bin /home/ec2-user/.nvm/versions/node/v16.20.2/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user

# Test application health and transaction endpoints
curl http://localhost:4000/health
curl http://localhost:4000/transaction

```
---

## Internal Load Balancer and Auto Scaling
### Step 1: Create AMI of AppServer
  - Select AppServer Instance> Action> Image and templates> Create Image
### Step 2: Create Target Group
- **Name:** `AppServerTG`
- **Protocol:** HTTP (4000)
- **Health Check Path:** `/health`

### Step 3: Create Internal Load Balancer
- **Name:** `AppServer-LB`
- **Subnets:** Private App Subnets
- **Security Group:** `Internal-LB-SG`

### Step 4: Launch Template
- **Name:** `AppServer-LaunchTemplate`
- **AMI:** AppServer AMI
- **Security Group:** `AppTier-SG`
- **IAM Role:** `EC2_S3_ReadOnly_SSM_Role`

### Step 5: Auto Scaling Group
- **Name:** `AppTier-ASG`
- **Desired Capacity:** 2
- **Min:** 2
- **Max:** 2
- **Load Balancer Target Group:** `AppServerTG`

---

## Web Tier Setup
### Step 1: Launch WebServer EC2 Instance
- **AMI:** Amazon Linux
- **Instance Type:** t2.micro
- **Subnet:** Public Web Subnet (AZ-1)
- **Security Group:** `WebTier-SG`
- **IAM Role:** `EC2_S3_ReadOnly_SSM_Role`

### Step 2: Deploy Web Application
1. Upload `web-tier` and `nginx.conf` to S3.
2. Install and configure NGINX.
3. Start the web application.

---

## Web Tier Deployment
### Step 1: Create AMI of WebServer

### Step 2: Create Target Group
- **Name:** `WebServerTG`
- **Protocol:** HTTP (80)
- **Health Check Path:** `/health`

### Step 3: Create Internet-Facing Load Balancer
- **Name:** `WebServer-LB`
- **Subnets:** Public Web Subnets
- **Security Group:** `InternetFacing-lb-sg`

---

## Architecture Diagram
Include an illustrative diagram showcasing the three-tier architecture, including the flow between Web, App, and Database tiers.

---

## Conclusion
This README provides step-by-step guidance for deploying a secure, scalable, and reliable three-tier web architecture on AWS. Follow the instructions carefully to ensure a successful deployment.

