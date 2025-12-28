# Private Lambda Connection with MongoDB by Using VPC Peering - Simple Step-by-Step Guide

## Overview

### What is VPC Peering?

VPC Peering allows secure communication between your Lambda function (in AWS VPC) and MongoDB Atlas database (in MongoDB's VPC) through AWS private network backbone.

### Architecture

```
User → API Gateway → VPC[ Lambda (Private Subnet) ] → VPC Peering → VPC[ MongoDB Atlas ]
```

### Two Connection Options

**Option 1: Public Access (NOT Recommended)**
- MongoDB allows public internet access
- Lambda connects over internet
- ```x``` Security Risk: Vulnerable to attacks
- ```x``` Slower: Goes over public internet
- ```x``` Fails compliance: HIPAA, PCI-DSS, SOC 2

**Option 2: VPC Peering (Recommended)**
- MongoDB in private VPC
- Lambda in private VPC
- ```☑``` Secure: No internet exposure
- ```☑``` Fast: AWS backbone network
- ```☑``` Compliant: Meets security standards

---

## Step 1: Create MongoDB Cluster

### 1.1 Go to MongoDB Atlas

```
Website: https://cloud.mongodb.com
Login with your credentials
```

### 1.2 Create Cluster

```
Dashboard
  → Data Storage
  → Clusters
  → Build a Cluster
  → Choose "Dedicated Clusters" (for Network isolation)
  → Create a cluster
```

**Note**: Shared clusters are FREE but don't support network peering.

### 1.3 Configure Cluster

```
Cloud Provider & Region:
  - Cloud Provider: AWS
  - Region: us-east-1 (match your Lambda region)

Cluster Tier:
  - Select: M10 (minimum for VPC peering)
  - Storage: 10 GB

Cluster Name: <CLUSTER-NAME>

Click: Create Cluster
(Wait 10-15 minutes for setup)
```

### 1.4 Set Billing Details

```
Complete payment information
```

### 1.5 Create Database and Collection

```
Open your Cluster
  → Collections tab
  → Add Your Own Data

Database Name: <DB-NAME>
Collection Name: <COLLECTION-NAME>

Click: Create
```

### 1.6 Create Database User

```
Security → Database Access
  → Add New Database User

Username: <USER-NAME>
Password: <PASSWORD> (save securely)

Click: Create User
```

### 1.7 Get Connection String

```
Clusters → Connect
  → Choose a connection method
  → Connect your application
  → Select: Node.js

Copy Connection String:
mongodb+srv://USERNAME:PASSWORD@cluster.xxxxx.mongodb.net/DB-NAME?retryWrites=true&w=majority

Save this - you'll need it for Lambda code
```

---

## Step 2: Create AWS VPC

### 2.1 Create VPC

```
AWS Console → VPC → VPCs
  → Create VPC

Name: <VPC-NAME>
IPv4 CIDR block: 10.0.0.0/16

Click: Create VPC
```

**Note the VPC ID**: vpc-xxxxxxxxxxxxx

### 2.2 Create Private Subnet

```
VPC → Subnets
  → Create subnet

VPC: <Select the VPC created above>
Subnet Name: <SUBNET-NAME>
Availability Zone: us-east-1a (match your region)
IPv4 CIDR block: 10.0.1.0/24

Click: Create subnet
```

**Note the Subnet ID**: subnet-xxxxxxxxxxxxx

---

## Step 3: Create Security Group

### 3.1 Create Security Group

```
VPC → Security Groups
  → Create security group

Security group name: <SG-NAME>
Description: Outbound for Lambda to MongoDB
VPC: <Select your VPC>

Click: Create security group
```

### 3.2 Configure Outbound Rules

```
Security Groups → Select your SG
  → Outbound rules tab
  → Edit outbound rules

Add Rule:
  Type: Custom TCP Rule
  Protocol: TCP
  Port range: 27017 (MongoDB port)
  Destination: 0.0.0.0/0 (or specific MongoDB CIDR later)

Click: Save rules
```

**Note the Security Group ID**: sg-xxxxxxxxxxxxx

---

## Step 4: Create and Configure Lambda Function

### Option A: Configure Existing Lambda

```
AWS Console → Lambda
  → Select your function
  → Configuration tab
  → VPC section
  → Click: Edit

VPC: <Select your VPC>
Subnets: <Select your private subnet>
Security groups: <Select your security group>

Click: Save
```

### Option B: Create with Serverless Framework

#### 4.1 Install Serverless

```bash
npm install -g serverless

sls --version
```

#### 4.2 Configure AWS Credentials

```bash
# Get credentials from AWS → IAM → Users → Security Credentials

serverless config credentials \
  --provider aws \
  --key YOUR_ACCESS_KEY \
  --secret YOUR_SECRET_KEY \
  --region us-east-1
```

#### 4.3 Create Project

```bash
mkdir lambda-mongodb
cd lambda-mongodb

serverless create --template aws-nodejs
```

#### 4.4 Update serverless.yml

```yaml
service: lambda-mongodb

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1

  environment:
    MONGODB_URI: ${env:MONGODB_URI}

functions:
  getData:
    handler: handler.getData
    vpc:
      securityGroupIds:
        - sg-xxxxxxxxxxxxx
      subnetIds:
        - subnet-xxxxxxxxxxxxx
    events:
      - http:
          path: data
          method: get
          cors: true
```

#### 4.5 Create handler.js

```javascript
'use strict';
const { MongoClient } = require('mongodb');

let cachedDb = null;

async function connectToDatabase(uri) {
  if (cachedDb) {
    return cachedDb;
  }

  const client = await MongoClient.connect(uri, {
    maxPoolSize: 10,
    serverSelectionTimeoutMS: 5000,
  });

  cachedDb = client.db();
  return cachedDb;
}

module.exports.getData = async (event) => {
  try {
    const db = await connectToDatabase(process.env.MONGODB_URI);

    const data = await db
      .collection('your-collection')
      .find({})
      .limit(10)
      .toArray();

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        success: true,
        data: data,
      }),
    };
  } catch (error) {
    console.error('Error:', error);

    return {
      statusCode: 500,
      body: JSON.stringify({
        success: false,
        error: error.message,
      }),
    };
  }
};
```

#### 4.6 Install Dependencies

```bash
npm install mongodb
```

#### 4.7 Set Environment Variable

**Create .env file:**

```bash
MONGODB_URI=mongodb+srv://USERNAME:PASSWORD@cluster.xxxxx.mongodb.net/DB-NAME?retryWrites=true&w=majority
```

#### 4.8 Deploy

```bash
sls deploy --stage dev

# Output will show your API endpoint
# Example: https://xxxxx.execute-api.us-east-1.amazonaws.com/dev/data
```

#### 4.9 View Logs

```bash
sls logs -f getData --tail
```

---

## Step 5: Enable DNS in VPC

### 5.1 Enable DNS Hostnames

```
AWS Console → VPC
  → Select your VPC
  → Actions
  → Edit DNS hostnames

☑ Enable DNS hostnames

Click: Save
```

### 5.2 Enable DNS Resolution

```
AWS Console → VPC
  → Select your VPC
  → Actions
  → Edit DNS resolution

☑ Enable DNS resolution

Click: Save
```

**Why**: Lambda needs DNS to resolve MongoDB connection string

---

## Step 6: Create VPC Peering Connection

### 6.1 Initiate Peering from MongoDB Atlas

```
MongoDB Atlas → Network Access
  → Peering tab
  → Add Peering Connection

Your Application VPC:
  Cloud Provider: AWS
  Account ID: <YOUR AWS ACCOUNT ID>
  VPC ID: vpc-xxxxxxxxxxxxx
  VPC CIDR: 10.0.0.0/16
  Region: us-east-1

Your Atlas VPC:
  Atlas VPC Region: us-east-1

Click: Initiate Peering
```

**Status**: "Pending Acceptance"
**Save**: Peering Connection ID (pcx-xxxxxxxxxxxxx)

### 6.2 Accept Peering in AWS

```
AWS Console → VPC
  → Peering Connections
  → Find pending request
  → Select it
  → Actions
  → Accept Request

Click: Accept
```

**Status**: "Active"

### 6.3 Note MongoDB CIDR Block

```
MongoDB Atlas → Network Access
  → Peering tab
  → View active peering connection

Note: Atlas CIDR (example: 172.31.0.0/16)
```

---

## Step 7: Configure Route Tables

### 7.1 Update Route Table

```
AWS Console → VPC
  → Subnets
  → Select your private subnet
  → Route Table tab
  → Click route table ID

Routes tab
  → Edit routes

Add Route:
  Destination: 172.31.0.0/16 (MongoDB CIDR)
  Target: Peering Connection (pcx-xxxxxxxxxxxxx)

Click: Save routes
```

**Result**: Route table should now show:
```
10.0.0.0/16       → Local
172.31.0.0/16     → pcx-xxxxxxxxxxxxx
```

---

## Step 8: Update Security Group

### 8.1 Configure Outbound Rules

```
AWS Console → EC2
  → Security Groups
  → Select your security group
  → Outbound rules tab
  → Edit outbound rules

Update Rule:
  Type: Custom TCP Rule
  Protocol: TCP
  Port: 27017
  Destination: 172.31.0.0/16 (MongoDB CIDR)

Click: Save rules
```

---

## Step 9: Test Connection

### 9.1 Test via Serverless

```bash
# Invoke function locally
sls invoke local -f getData --stage dev

# Check response
```

### 9.2 Test via HTTP

```bash
# Get your API endpoint from deployment
curl https://xxxxx.execute-api.us-east-1.amazonaws.com/dev/data

# Expected output:
{
  "success": true,
  "data": [...]
}
```

### 9.3 View Logs

```bash
sls logs -f getData -t --stage dev

# Should show successful MongoDB connection
```

---

## Alternative: Connect via NAT Gateway (Optional)

**Use this if you prefer internet routing instead of VPC peering**

### A1. Create Elastic IP

```
AWS Console → EC2
  → Elastic IPs
  → Allocate

Region: us-east-1

Save Allocation ID: eipalloc-xxxxxxxxxxxxx
```

### A2. Create NAT Gateway

```
VPC → NAT Gateways
  → Create NAT gateway

Name: <NAT-NAME>
Subnet: <Select a public subnet>
Elastic IP: <Select the elastic IP created above>

Click: Create NAT Gateway

Save NAT ID: nat-xxxxxxxxxxxxx
```

### A3. Update Route Table

```
VPC → Route Tables
  → Select private subnet route table
  → Edit routes

Add Route:
  Destination: 0.0.0.0/0
  Target: NAT Gateway (nat-xxxxxxxxxxxxx)

Click: Save routes
```

### Cost Comparison

```
VPC Peering:
- Monthly: ~$7 ($0.01/hour)

NAT Gateway:
- Monthly: ~$32 + data transfer ($0.045/GB)
- Example: 1TB traffic = $32 + $46 = $78/month

Recommendation: Use VPC Peering (10x cheaper)
```

---

## Troubleshooting

### Problem 1: "Cannot connect to MongoDB"

**Cause**: DNS not enabled

**Fix**:
```bash
# Verify DNS is enabled
aws ec2 describe-vpcs \
  --vpc-ids vpc-xxxxxxxxxxxxx \
  --query 'Vpcs[0].[EnableDnsHostnames,EnableDnsSupport]'

# If false, enable
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-xxxxxxxxxxxxx \
  --enable-dns-hostnames

aws ec2 modify-vpc-attribute \
  --vpc-id vpc-xxxxxxxxxxxxx \
  --enable-dns-support
```

### Problem 2: "Connection timeout"

**Causes**:
- Route table not configured
- Security group rules incorrect
- Peering connection not active

**Fix**:
```bash
# Check route table
aws ec2 describe-route-tables \
  --subnet-ids subnet-xxxxxxxxxxxxx

# Check peering connection status
aws ec2 describe-vpc-peering-connections \
  --query 'VpcPeeringConnections[*].[VpcPeeringConnectionId,Status.Code]'

# Should show: Active
```

### Problem 3: "Authentication failed"

**Cause**: Wrong MongoDB credentials

**Fix**:
```bash
# Verify connection string
echo $MONGODB_URI

# Test locally with mongosh
mongosh "your-connection-string"
```

### Problem 4: "Security group has no rules for port 27017"

**Fix**:
```bash
# Add outbound rule
aws ec2 authorize-security-group-egress \
  --group-id sg-xxxxxxxxxxxxx \
  --protocol tcp \
  --port 27017 \
  --cidr 172.31.0.0/16
```

---

## Summary

1. Create MongoDB cluster (M10 Dedicated)
2. Create AWS VPC (10.0.0.0/16) and subnet (10.0.1.0/24)
3. Create security group with port 27017 access
4. Deploy Lambda in VPC + subnet + security group
5. Enable DNS in VPC (hostnames + resolution)
6. Initiate peering connection (MongoDB Atlas)
7. Accept peering connection (AWS)
8. Add route to MongoDB CIDR via peering connection
9. Update security group for MongoDB CIDR (port 27017)
10. Test connection via API endpoint
