# Guidance for L2 Stretch Network with Cisco 8000v on AWS

## Table of Contents

### Required

1. [Overview](#overview)
    - [Cost](#cost)
2. [Prerequisites](#prerequisites)
    - [Operating System](#operating-system)
3. [Deployment Steps](#deployment-steps)
4. [Deployment Validation](#deployment-validation)
5. [Running the Guidance](#running-the-guidance)
6. [Next Steps](#next-steps)
7. [Cleanup](#cleanup)
8. [Notices](#notices)

### Optional

9. [FAQ, known issues, additional considerations, and limitations](#faq-known-issues-additional-considerations-and-limitations)
10. [Authors](#authors)

## Overview

This Guidance enables enterprise customers to seamlessly extend on-premises network subnets to AWS using Cisco Catalyst 8000V routers and LISP (Locator/ID Separation Protocol) technology. It solves the critical problem of migrating legacy applications with hardcoded IP addresses to AWS without requiring complex IP reconfiguration.

**Why was this Guidance built?**

Many enterprise applications, particularly legacy systems, have hardcoded IP addresses and complex network dependencies that make cloud migration challenging. Changing IP addresses during migration can lead to extended downtime, increased risks, and complex reconfiguration across interconnected systems. This Guidance provides a Layer 2 network extension solution that allows virtual machines to migrate to AWS while maintaining their original IP addresses, ensuring business continuity and reducing migration complexity.

**Note:** This Guidance focuses on the AWS cloud-side infrastructure deployment. A companion CloudFormation template (`L2E-lisp-on-prem-vpc-v3.yaml`) is provided for lab testing purposes to simulate an on-premises environment within AWS, but production deployments would connect to actual on-premises Cisco routers.

**Architecture Overview:**

```
┌──────────────────────┐    IPSec/LISP Tunnel      ┌─────────────────────┐
│   On-Premises        │◄─────────────────────────►│     AWS Cloud       │
│   VMware vSphere     │                           │                     │
│                      │                           │                     │
│  ┌───────────────┐   │                           │  ┌───────────────┐  │
│  │ Cisco 8000V   │   │                           │  │ Cisco 8000V   │  │
│  │ (Physical or  │   │                           │  │ (EC2 Instance)│  │
│  │  Virtual)     │   │                           │  │               │  │
│  └───────────────┘   │                           │  └───────────────┘  │
│         │            │                           │         │           │
│  ┌─────────────┐     │                           │  ┌─────────────┐    │
│  │  Extended   │     │                           │  │  Extended   │    │
│  │  Subnet     │     │                           │  │  Subnet     │    │
│  │172.16.1.0/24│     │                           │  │172.16.1.0/24│    │
│  └─────────────┘     │                           │  └─────────────┘    │
└──────────────────────┘                           └─────────────────────┘
```

**High-Level Architecture Flow:**

1. **VPC Infrastructure**: Deploys a VPC with public and private subnets in AWS
2. **Cisco 8000V Deployment**: Launches a Cisco Catalyst 8000V router instance with dual network interfaces
3. **IPSec Tunnel**: Establishes encrypted IPSec tunnel between on-premises and AWS
4. **LISP Configuration**: Enables dynamic endpoint registration and Layer 2 adjacency
5. **OSPF Routing**: Provides routing protocol convergence between sites
6. **Layer 2 Extension**: Allows VMs in the extended subnet to communicate seamlessly across sites

### Cost

You are responsible for the cost of the AWS services used while running this Guidance. As of November 2024, the cost for running this Guidance with the default settings in the US East (N. Virginia) Region is approximately **$150-$200 per month** for a basic deployment with c5n.large instance type.

We recommend creating a [Budget](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) through [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/) to help manage costs. Prices are subject to change. For full details, refer to the pricing webpage for each AWS service used in this Guidance.

### Sample Cost Table

The following table provides a sample cost breakdown for deploying this Guidance with the default parameters in the US East (N. Virginia) Region for one month.

| AWS Service | Dimensions | Cost [USD] |
| ----------- | ------------ | ------------ |
| Amazon EC2 (Cisco 8000V) | c5n.large instance, 730 hours/month | $78.84/month |
| Amazon EC2 (Test Instance) | t3.micro instance, 730 hours/month | $7.59/month |
| Elastic IP Address | 1 EIP attached to 8000V, 730 hours/month | $3.65/month |
| NAT Gateway | 1 NAT Gateway, 100 GB processed | $37.35/month |
| VPC | Standard VPC, subnet, routing | $0.00 |
| Data Transfer | 100 GB out to Internet | $9.00/month |
| **Total Estimated Cost** | | **~$138.13/month** |

**Note:** Costs will vary based on:
- Instance type selection (larger instances for higher throughput)
- Data transfer volumes
- EC2/Compute Savings plans (recommended)
- Cisco C8000v License cost (purchased directly with Cisco/Partner)

## Prerequisites

### Operating System

These deployment instructions are optimized to work on **macOS, Linux, or Windows** with AWS CLI installed. The CloudFormation templates can be deployed from any operating system with AWS CLI access.

**Required Tools:**

```bash
# AWS CLI (version 2.x recommended)
aws --version

# Optional: TaskCat for automated testing
pip install taskcat
```

### Third-party tools

**Cisco Catalyst 8000V License**
- BYOL (Bring Your Own License) required for production use
- Rate limited evaluation license included for testing
- Subscribe to Cisco Catalyst 8000V in AWS Marketplace before deployment

### AWS account requirements

**Required Pre-requisites:**

1. **AWS Marketplace Subscription**
   - Subscribe to [Cisco Catalyst 8000V](https://aws.amazon.com/marketplace/pp/prodview-gvkcuwm3c6dru) in AWS Marketplace
   
2. **EC2 Key Pair**
   - Create an EC2 key pair in your target region for SSH access
   
3. **IAM Permissions**
   - CloudFormation full access
   - EC2 full access
   - VPC full access
   - IAM role creation permissions
   
4. **VPC Quotas**
   - Ensure sufficient VPC quota for creating new VPCs
   - Default limit: 5 VPCs per region (can be increased)

5. **Elastic IP Address**
   - Requires 1 available Elastic IP for the 8000V public interface
   - Default limit: 5 EIPs per region (can be increased)

### Service limits

**Critical Service Limits:**

- **EC2 Instance Limits**: Ensure your account has sufficient quota for c5n.large instances (or your chosen instance type)
- **Elastic Network Interfaces**: Each deployment requires multiple ENIs
- **IPv4 CIDR Blocks**: Ensure your chosen CIDR blocks don't conflict with existing VPCs
- **Secondary Private IPs per ENI**: Default limit is ~50 per ENI (varies by instance type)
- **IPv4 Prefixes per ENI**: Scales better than secondary IPs (up to 464 IPs on c5n.4xlarge with 29 prefixes)

To request limit increases, visit the [AWS Service Quotas console](https://console.aws.amazon.com/servicequotas/).

### Supported Regions

This Guidance supports all AWS Regions where Cisco Catalyst 8000V is available in AWS Marketplace. Tested regions include:
- US East (N. Virginia) - us-east-1
- US West (Oregon) - us-west-2
- EU (Ireland) - eu-west-1
- Asia Pacific (Singapore) - ap-southeast-1

## Deployment Steps

Follow these steps to deploy the L2 Stretch Network solution:

### Option 1: Automated Deployment with TaskCat (Recommended)

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-org/lisp-l2e.git
   cd lisp-l2e
   ```

2. **Configure AWS CLI profile**
   ```bash
   # Verify AWS credentials
   aws sts get-caller-identity --profile <YOUR-PROFILE>
   ```

3. **Install TaskCat**
   ```bash
   pip install taskcat
   ```

4. **Edit TaskCat configuration**
   ```bash
   # Edit .taskcat.yml with your parameters
   vim .taskcat.yml
   ```
   
   Update the following parameters:
   - `KeyName`: Your EC2 key pair name
   - `ParticipantIPAddress`: Your public IP address in CIDR format
   - `TunnelDestinationPublicIP`: On-premises public IP (if known)

5. **Run TaskCat deployment**
   ```bash
   taskcat test run
   ```

6. **Monitor deployment**
   TaskCat will create the stack and monitor progress. View the HTML report:
   ```bash
   open taskcat_outputs/index.html
   ```

7. **Retrieve stack outputs**
   ```bash
   aws cloudformation describe-stacks \
     --stack-name tCaT-lisp-l2e-deployment-lisp-cloud-deployment \
     --query 'Stacks[0].Outputs' \
     --output table
   ```

### Option 2: Manual CloudFormation Deployment

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-org/lisp-l2e.git
   cd lisp-l2e
   ```

2. **Deploy the CloudFormation stack**
   ```bash
   aws cloudformation create-stack \
     --stack-name lisp-cloud-extension \
     --template-body file://L2E-lisp-cloud-vpc-v3.yaml \
     --parameters \
       ParameterKey=LispCloudVpcCidr,ParameterValue=172.16.0.0/16 \
       ParameterKey=LispCloudPublicSubnetCidr,ParameterValue=172.16.0.0/24 \
       ParameterKey=LispCloudPrivateSubnetCidr,ParameterValue=172.16.1.0/24 \
       ParameterKey=LispCloud8KvInstanceType,ParameterValue=c5n.large \
       ParameterKey=LispCloud8KvPrivateInterfaceIP,ParameterValue='' \
       ParameterKey=ParticipantIPAddress,ParameterValue=1.2.3.4/32 \
       ParameterKey=Hostname,ParameterValue=lisp-cloud-router \
       ParameterKey=AuthenticationType,ParameterValue=KeyPair \
       ParameterKey=KeyName,ParameterValue=your-key-name \
       ParameterKey=Username,ParameterValue=admin \
       ParameterKey=PrivilegePwd,ParameterValue=YourPassword123 \
       ParameterKey=LispCloudLoopback,ParameterValue=33.33.33.33 \
       ParameterKey=LispOnPremLoopback,ParameterValue=11.11.11.11 \
       ParameterKey=IPSecPreShareKey,ParameterValue=YourSecretKey \
       ParameterKey=TunnelDestinationPublicIP,ParameterValue=12.34.56.78 \
     --capabilities CAPABILITY_IAM \
     --profile <YOUR-PROFILE>
   ```

3. **Monitor stack creation**
   ```bash
   aws cloudformation wait stack-create-complete \
     --stack-name lisp-cloud-extension \
     --profile <YOUR-PROFILE>
   ```

4. **Retrieve important outputs**
   ```bash
   # Get 8KV public IP
   aws cloudformation describe-stacks \
     --stack-name lisp-cloud-extension \
     --query 'Stacks[0].Outputs[?OutputKey==`LispCloud8KvPublicIp`].OutputValue' \
     --output text
   
   # Get 8KV instance ID
   aws cloudformation describe-stacks \
     --stack-name lisp-cloud-extension \
     --query 'Stacks[0].Outputs[?OutputKey==`C8KVInstanceId`].OutputValue' \
     --output text
   
   # Get private ENI ID
   aws cloudformation describe-stacks \
     --stack-name lisp-cloud-extension \
     --query 'Stacks[0].Outputs[?OutputKey==`LispCloud8KvPrivateInterfaceId`].OutputValue' \
     --output text
   ```

## Deployment Validation

### Validate CloudFormation Stack

1. **Check stack status**
   ```bash
   aws cloudformation describe-stacks \
     --stack-name lisp-cloud-extension \
     --query 'Stacks[0].StackStatus' \
     --output text
   ```
   
   Expected output: `CREATE_COMPLETE`

2. **Verify all resources created**
   ```bash
   aws cloudformation describe-stack-resources \
     --stack-name lisp-cloud-extension \
     --output table
   ```

### Validate EC2 Instances

3. **Verify Cisco 8000V instance is running**
   ```bash
   INSTANCE_ID=$(aws cloudformation describe-stacks \
     --stack-name lisp-cloud-extension \
     --query 'Stacks[0].Outputs[?OutputKey==`C8KVInstanceId`].OutputValue' \
     --output text)
   
   aws ec2 describe-instances \
     --instance-ids $INSTANCE_ID \
     --query 'Reservations[0].Instances[0].State.Name' \
     --output text
   ```
   
   Expected output: `running`

### Validate Network Configuration

4. **Verify Elastic IP association**
   ```bash
   aws ec2 describe-addresses \
     --filters "Name=instance-id,Values=$INSTANCE_ID" \
     --query 'Addresses[0].PublicIp' \
     --output text
   ```

5. **Verify ENI attachments**
   ```bash
   aws ec2 describe-instances \
     --instance-ids $INSTANCE_ID \
     --query 'Reservations[0].Instances[0].NetworkInterfaces[*].{Index:Attachment.DeviceIndex,ENI:NetworkInterfaceId,PrivateIP:PrivateIpAddress}' \
     --output table
   ```
   
   Expected: Two network interfaces (index 0 and 1)

### Validate Cisco 8000V Configuration

6. **SSH to Cisco 8000V**
   ```bash
   PUBLIC_IP=$(aws cloudformation describe-stacks \
     --stack-name lisp-cloud-extension \
     --query 'Stacks[0].Outputs[?OutputKey==`LispCloud8KvPublicIp`].OutputValue' \
     --output text)
   
   ssh -i your-key.pem admin@$PUBLIC_IP
   ```

7. **Verify LISP configuration**
   ```cisco
   show ip lisp
   show ip lisp database
   show running-config | section lisp
   ```

8. **Verify IPSec configuration**
   ```cisco
   show crypto isakmp sa
   show crypto ipsec sa
   show running-config | section crypto
   ```

## Running the Guidance

### Initial Setup

After successful deployment, configure the on-premises side and test connectivity:

### 1. Configure On-Premises Cisco 8000V

Deploy matching configuration on your on-premises Cisco router:

```cisco
! Configure loopback interface
interface Loopback1
 ip address 11.11.11.11 255.255.255.255

! Configure IPSec
crypto isakmp policy 200
 encryption aes
 hash sha
 authentication pre-share
 group 21
 lifetime 28800
 
crypto isakmp key <YOUR-PRESHARED-KEY> address 0.0.0.0
crypto isakmp keepalive 10 10

crypto ipsec transform-set dmvpn esp-aes esp-sha-hmac
 mode tunnel

crypto ipsec profile Catalyst8000v1
 set transform-set dmvpn

! Configure tunnel (replace AWS_8KV_PUBLIC_IP with actual IP)
interface Tunnel0
 ip address 12.0.0.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination <AWS_8KV_PUBLIC_IP>
 tunnel protection ipsec profile Catalyst8000v1

! Configure LISP
router lisp
 locator-set onprem
  11.11.11.11 priority 1 weight 100
 exit-locator-set
 service ipv4
  itr map-resolver 33.33.33.33
  itr
  etr map-server 33.33.33.33 key <YOUR-PRESHARED-KEY>
  etr
  use-petr 33.33.33.33
 exit-service-ipv4
 instance-id 0
  dynamic-eid subnet1
   database-mapping 172.16.1.0/24 locator-set onprem
   map-notify-group 239.0.0.1
  exit-dynamic-eid
  service ipv4
   eid-table default
  exit-service-ipv4
 exit-instance-id
exit-router-lisp

! Configure OSPF
router ospf 1
 network 12.0.0.0 0.0.0.255 area 0
 network 11.11.11.11 0.0.0.0 area 0
```

### 2. Verify Tunnel Establishment

**On AWS 8000V:**
```cisco
show crypto isakmp sa
! Expected: QM_IDLE state

show crypto ipsec sa
! Expected: Active SAs with encaps/decaps counters incrementing

show ip ospf neighbor
! Expected: Neighbor 12.0.0.1 in FULL state
```

**On On-Premises 8000V:**
```cisco
show crypto isakmp sa
show crypto ipsec sa
show ip ospf neighbor
! Expected: Neighbor 12.0.0.2 in FULL state
```

### 3. Verify LISP Registration

```cisco
show ip lisp map-cache
! Expected: See mappings for remote subnet

show ip lisp database
! Expected: See local subnet mappings

show ip lisp statistics
! Expected: Map-Requests/Replies counters incrementing
```

### 4. Test Connectivity

**From AWS test instance:**
```bash
# Get test instance private IP
aws cloudformation describe-stacks \
  --stack-name lisp-cloud-extension \
  --query 'Stacks[0].Outputs[?OutputKey==`LispCloudAppServerPrivateIp`].OutputValue' \
  --output text

# SSH to test instance and ping on-premises
ping <on-prem-server-ip>
```

### Expected Output

**Successful Deployment Indicators:**
- IPSec tunnel status: `QM_IDLE`
- OSPF neighbor state: `FULL`
- LISP map-cache populated with remote subnet
- Ping successful between AWS and on-premises subnets
- Continuous packet flow without drops

### 5. Configure IPv4 Prefixes for Scale (Optional)

For deployments requiring Layer 2 extension support for many on-premises endpoints, IPv4 prefixes provide superior scalability compared to individual secondary IP addresses. Instead of managing IPs one at a time (limited to ~50 per ENI), IPv4 prefixes allow you to allocate entire /28 subnets to the Cisco 8000V's private network interface. Each /28 prefix provides 16 usable IP addresses, and instance types like c5n.4xlarge support up to 29 prefixes, enabling connectivity for up to 464 on-premises endpoints through a single Cisco 8000V instance.

This approach is essential for enterprise migrations where hundreds of on-premises servers with static IP addresses need to maintain their addresses while moving workloads to AWS. The prefixes enable the LISP protocol to properly register and route traffic for all on-premises endpoint IPs through the Layer 2 extension tunnel.

**Scalability with IPv4 Prefixes:**
- c5n.large: Up to 144 IPs (9 prefixes × 16 IPs each)
- c5n.2xlarge: Up to 224 IPs (14 prefixes × 16 IPs each)
- c5n.4xlarge: Up to 464 IPs (29 prefixes × 16 IPs each)

#### Step 1: Retrieve the Private ENI ID

First, get the private network interface ID from your CloudFormation stack outputs:

```bash
PRIVATE_ENI=$(aws cloudformation describe-stacks \
  --stack-name lisp-cloud-extension \
  --query 'Stacks[0].Outputs[?OutputKey==`LispCloud8KvPrivateInterfaceId`].OutputValue' \
  --output text)

echo "Private ENI ID: $PRIVATE_ENI"
```

#### Step 2: Plan Your /28 Prefix Allocations

For a 172.16.1.0/24 private subnet:

```bash
# /28 subnets provide 16 IPs each
172.16.1.0/28    → 172.16.1.0 - 172.16.1.15    (16 IPs)
172.16.1.16/28   → 172.16.1.16 - 172.16.1.31   (16 IPs)
172.16.1.32/28   → 172.16.1.32 - 172.16.1.47   (16 IPs)
172.16.1.48/28   → 172.16.1.48 - 172.16.1.63   (16 IPs)
172.16.1.64/28   → 172.16.1.64 - 172.16.1.79   (16 IPs)
172.16.1.80/28   → 172.16.1.80 - 172.16.1.95   (16 IPs)
172.16.1.96/28   → 172.16.1.96 - 172.16.1.111  (16 IPs)
172.16.1.112/28  → 172.16.1.112 - 172.16.1.127 (16 IPs)
172.16.1.128/28  → 172.16.1.128 - 172.16.1.143 (16 IPs)
# ... and so on
```

**Important:** All 16 IPs in each /28 are usable, including boundary addresses (e.g., .16 and .31 in a .16/28 range).

#### Step 3: Assign IPv4 Prefixes via AWS Console

1. Navigate to the **EC2 Console** → **Network Interfaces**
2. Search for the ENI ID from step 1
3. Select the network interface and choose **Actions** → **Manage IP addresses**
4. In the **IPv4 Prefixes** section, click **Assign new IP prefix**
5. Enter your /28 CIDR blocks (e.g., `172.16.1.16/28`)
6. Add additional prefixes as needed for your on-premises endpoints
7. Click **Save**

#### Step 4: Verify Prefix Assignment

You can verify the assigned prefixes in the EC2 console under the network interface details, or use AWS CLI:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids $PRIVATE_ENI \
  --query 'NetworkInterfaces[0].Ipv4Prefixes[*].Ipv4Prefix' \
  --output table
```

#### Best Practices for IPv4 Prefix Management

1. **Plan Your Addressing**
   - Document which /28 ranges correspond to on-premises server groups
   - Leave the first /28 (172.16.1.0/28) for AWS resources if needed
   - Assign consecutive /28 blocks for easier management

2. **Scale Appropriately**
   - Choose instance type based on the number of on-premises endpoints
   - Each /28 supports 16 on-premises servers
   - Calculate: `Required_Prefixes = ceil(Total_OnPrem_Servers / 16)`

3. **Production Best Practices**
   - Document your prefix allocations in a spreadsheet or configuration management system
   - Map each prefix to specific on-premises server groups or VLANs
   - Plan for growth by reserving additional prefix space
   - Test with a small number of prefixes before scaling up

#### IPv4 Prefix vs Secondary IP Decision Guide

**Use IPv4 Prefixes when:**
- You have more than 50 on-premises endpoints
- You need to scale beyond secondary IP limits
- You want to allocate IPs in bulk by subnet ranges
- Your on-premises servers are logically grouped

**Use Secondary IPs when:**
- You have fewer than 50 on-premises endpoints
- You need precise control over individual IPs
- Your on-premises endpoints are scattered across the subnet
- You prefer simpler management for small deployments

#### Troubleshooting IPv4 Prefix Assignment

**Common Issues:**

1. **Prefix Limit Reached**
   ```bash
   # Check current prefix count
   aws ec2 describe-network-interfaces \
     --network-interface-ids $PRIVATE_ENI \
     --query 'length(NetworkInterfaces[0].Ipv4Prefixes[*])'
   
   # Compare with instance type limits
   aws ec2 describe-instance-types \
     --instance-types c5n.large \
     --query 'InstanceTypes[0].NetworkInfo.Ipv4PrefixesPerInterface'
   ```

2. **Overlapping Prefixes**
   - Ensure /28 ranges don't overlap
   - Use a subnet calculator to plan allocations
   - Verify against existing secondary IPs

3. **Subnet CIDR Conflicts**
   ```bash
   # Verify prefix is within subnet range
   aws ec2 describe-subnets \
     --subnet-ids $(aws ec2 describe-network-interfaces \
       --network-interface-ids $PRIVATE_ENI \
       --query 'NetworkInterfaces[0].SubnetId' \
       --output text) \
     --query 'Subnets[0].CidrBlock'
   ```

## Next Steps

Enhance your deployment with these recommendations:

### 1. High Availability

- Deploy a second Cisco 8000V instance in a different Availability Zone
- Configure BGP or OSPF for redundant routing
- Use multiple IPSec tunnels for path diversity

### 2. Monitoring and Logging

- Enable VPC Flow Logs for traffic analysis
- Configure CloudWatch alarms for:
  - EC2 instance health
  - Network throughput
  - Tunnel status
- Use AWS Systems Manager Session Manager for secure access

### 3. Security Enhancements

- Restrict security group rules to minimum required ports
- Implement AWS WAF for application layer protection
- Enable AWS GuardDuty for threat detection
- Use AWS Secrets Manager for credential management

### 4. Performance Optimization

- Upgrade to larger instance types for higher throughput (c5n.2xlarge, c5n.4xlarge)
- Enable Enhanced Networking (already enabled on c5n instances)
- Configure MTU optimization for tunnel overhead
- Implement QoS policies for traffic prioritization

### 5. Cost Optimization

- Use Reserved Instances or Savings Plans for predictable workloads
- Implement auto-scaling for variable workloads
- Review data transfer patterns and optimize accordingly
- Consider AWS Direct Connect for high-volume data transfer

### 6. Operational Excellence

- Document custom configurations
- Implement Infrastructure as Code with version control
- Create runbooks for common operational tasks
- Schedule regular backup and disaster recovery tests

## Cleanup

To avoid ongoing charges, delete the deployed resources:

### Option 1: Delete via CloudFormation Console

1. Open the [CloudFormation console](https://console.aws.amazon.com/cloudformation/)
2. Select your stack (e.g., `lisp-cloud-extension`)
3. Click **Delete**
4. Confirm deletion

### Option 2: Delete via AWS CLI

```bash
# Delete the stack
aws cloudformation delete-stack \
  --stack-name lisp-cloud-extension

# Wait for deletion to complete
aws cloudformation wait stack-delete-complete \
  --stack-name lisp-cloud-extension
```

### Manual Cleanup Steps

If you added IPv4 prefixes or secondary IPs manually through the AWS console, remove them before deleting the stack:

1. Navigate to the EC2 console → Network Interfaces
2. Find the private network interface for the Cisco 8000V (use the `LispCloud8KvPrivateInterfaceId` output)
3. Select the network interface and choose **Manage IP addresses**
4. Remove any IPv4 prefixes you added
5. Remove any secondary private IP addresses you added
6. Click **Save**

Then proceed with stack deletion.

### Verify Cleanup

```bash
# Verify stack deletion
aws cloudformation describe-stacks \
  --stack-name lisp-cloud-extension

# Expected: Stack does not exist error
```

## FAQ, known issues, additional considerations, and limitations

### Known Issues

**Issue 1: Tunnel Not Establishing**
- **Symptom**: IPSec SA shows `MM_WAIT_MSG2` or `MM_NO_STATE`
- **Cause**: Firewall blocking UDP 500/4500, or incorrect pre-shared key
- **Resolution**: 
  - Verify security group allows UDP 500 and 4500 from on-premises IP
  - Confirm pre-shared key matches on both sides
  - Check on-premises firewall rules

**Issue 2: LISP Database Not Populating**
- **Symptom**: `show ip lisp database` empty or incomplete
- **Cause**: LISP configuration mismatch or tunnel not established
- **Resolution**:
  - Ensure IPSec tunnel is up first
  - Verify LISP map-server/map-resolver IPs are correct
  - Check LISP shared key matches

**Issue 3: Maximum Secondary IPs Reached**
- **Symptom**: Cannot add more secondary IPs to the private network interface
- **Cause**: Hit ENI secondary IP limit (~50 for most instance types)
- **Resolution**: Use IPv4 prefixes instead - they provide better scalability (up to 464 IPs on c5n.4xlarge)

### Additional Considerations

**Billing Considerations:**
- Cisco 8000V instances run 24/7 and incur continuous charges
- Data transfer out to Internet incurs charges ($0.09/GB in us-east-1)
- NAT Gateway charges are per hour and per GB processed
- Elastic IPs are free when attached to running instances

**Performance Considerations:**
- IPSec encryption adds CPU overhead - monitor instance CPU utilization
- MTU should be reduced for tunnel overhead (1400 bytes recommended)
- c5n instance types provide enhanced networking and up to 100 Gbps network performance
- Each /28 IPv4 prefix provides 16 usable IP addresses for on-premises endpoint connectivity

**Security Considerations:**
- This Guidance creates public IP addresses for 8000V management
- IPSec tunnels are encrypted but validate pre-shared key security
- Restrict management access to known IP addresses only
- Regularly update Cisco IOS-XE software for security patches

### Limitations

- **Maximum Endpoints per 8000V Instance**: Limited by ENI IP capacity:
  - ~50 endpoints with secondary IPs only
  - Up to 464 endpoints with IPv4 prefixes (c5n.4xlarge)
- **Supported Instance Types**: Cisco 8000V requires compute-optimized instances (c5/c5n/c6in family)
- **IPv6**: This Guidance focuses on IPv4; IPv6 support requires additional LISP configuration
- **Multicast**: LISP-based L2E has limited multicast support (some multicast may not work)
- **Broadcast**: Broadcast traffic is not forwarded across LISP tunnels by design
- **On-Premises Requirement**: Requires compatible Cisco router with LISP and IPSec support on-premises

For any feedback, questions, or suggestions, please use the issues tab under this repo.

## Notices

*Customers are responsible for making their own independent assessment of the information in this Guidance. This Guidance: (a) is for informational purposes only, (b) represents AWS current product offerings and practices, which are subject to change without notice, and (c) does not create any commitments or assurances from AWS and its affiliates, suppliers or licensors. AWS products or services are provided "as is" without warranties, representations, or conditions of any kind, whether express or implied. AWS responsibilities and liabilities to its customers are controlled by AWS agreements, and this Guidance is not part of, nor does it modify, any agreement between AWS and its customers.*

## Authors

- AWS Sr. Partner Solutions Architect: Josh Leatham, jlleatha@amazon.com

