# Lab 9: Networking Configuration and Troubleshooting

## Estimated Time: 1 hour 30 minutes
## Objectives
- Build a VPC from scratch with public/private subnets
- Understand Security Groups vs NACLs
- Configure and analyze VPC Flow Logs
- Master Route 53 routing policies
- Set up CloudFront and understand edge services
- Troubleshoot common networking issues

## Prerequisites
- AWS account with admin access
- AWS CLI configured

---

## Part 1: VPC Architecture

### Step 1: Create VPC and Subnets

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=lab9-vpc

# Create public subnet
PUBLIC_SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)
aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_SUBNET --map-public-ip-on-launch

# Create private subnet
PRIVATE_SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)
```

### Step 2: Internet Gateway and NAT Gateway

```bash
# Internet Gateway (for public subnets)
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Elastic IP for NAT Gateway
EIP_ID=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

# NAT Gateway (in public subnet, used by private subnet)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_SUBNET \
  --allocation-id $EIP_ID --query 'NatGateway.NatGatewayId' --output text)

# Wait for NAT Gateway
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID
```

### Step 3: Route Tables

```bash
# Public route table
PUBLIC_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUBLIC_RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET

# Private route table
PRIVATE_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PRIVATE_RT \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_ID
aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $PRIVATE_SUBNET
```

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Level** | Instance (ENI) | Subnet |
| **State** | Stateful (return traffic auto-allowed) | Stateless (must allow return traffic explicitly) |
| **Rules** | Allow rules only | Allow AND Deny rules |
| **Evaluation** | All rules evaluated (most permissive wins) | Rules evaluated in number order (first match wins) |
| **Default** | Deny all inbound, allow all outbound | Allow all inbound and outbound |
| **Association** | Multiple SGs per instance | One NACL per subnet |

```bash
# Create security group
SG_ID=$(aws ec2 create-security-group --group-name lab9-web-sg \
  --description "Web server SG" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 22 --cidr 203.0.113.0/32  # Your IP only!

# Create NACL with explicit rules
NACL_ID=$(aws ec2 create-network-acl --vpc-id $VPC_ID \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Inbound rules
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 100 --protocol tcp --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress

aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 110 --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress

# CRITICAL: Allow ephemeral ports for return traffic (stateless!)
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 120 --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress

# Outbound rules
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 100 --protocol tcp --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 --rule-action allow --egress

aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 110 --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow --egress

# Ephemeral ports outbound (for responses to inbound requests)
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 120 --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow --egress
```

> **Exam Tip:** The #1 NACL gotcha is forgetting ephemeral ports (1024-65535). Since NACLs are stateless, response traffic on ephemeral ports must be explicitly allowed. Security groups handle this automatically (stateful).

### VPC Endpoints

```bash
# Gateway Endpoint for S3 (free, route-table based)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $PRIVATE_RT

# Interface Endpoint for SSM (ENI-based, uses PrivateLink)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ssm \
  --subnet-ids $PRIVATE_SUBNET \
  --security-group-ids $SG_ID \
  --private-dns-enabled
```

| Endpoint Type | Services | How it Works | Cost |
|---------------|----------|-------------- |------|
| **Gateway** | S3, DynamoDB only | Route table entry | Free |
| **Interface** | Most other services | ENI with private IP (PrivateLink) | Hourly + data |

> **Exam Tip:** Gateway endpoints are free and go in the route table. Interface endpoints cost money and create an ENI. For S3/DynamoDB, always choose Gateway endpoints unless you need specific PrivateLink features.

---

## Part 2: VPC Flow Logs

### Enable Flow Logs

```bash
# Create log group
aws logs create-log-group --log-group-name "/vpc/flow-logs"

# Enable VPC-level flow logs
aws ec2 create-flow-log \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name "/vpc/flow-logs" \
  --deliver-logs-permission-arn "arn:aws:iam::ACCOUNT:role/VPCFlowLogsRole"

# OR send to S3 for long-term storage
aws ec2 create-flow-log \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination "arn:aws:s3:::my-flow-logs-bucket/vpc-logs/"
```

### Flow Log Record Format

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
2 123456789012 eni-abc123 10.0.1.5 52.94.76.5 49152 443 6 12 840 1680000000 1680000060 ACCEPT OK
2 123456789012 eni-abc123 203.0.113.1 10.0.1.5 0 0 1 1 28 1680000000 1680000060 REJECT OK
```

| Field | Description |
|-------|-------------|
| **srcaddr/dstaddr** | Source and destination IP |
| **srcport/dstport** | Source and destination port |
| **protocol** | 6=TCP, 17=UDP, 1=ICMP |
| **action** | ACCEPT or REJECT |
| **packets/bytes** | Traffic volume |

### Analyze with CloudWatch Logs Insights

```sql
-- Top rejected traffic (potential attacks or misconfigurations)
filter action = "REJECT"
| stats count(*) as rejectCount by srcAddr, dstPort
| sort rejectCount desc
| limit 20

-- Top talkers by bytes transferred
stats sum(bytes) as totalBytes by srcAddr
| sort totalBytes desc
| limit 10

-- SSH attempts from external IPs
filter dstPort = 22 and action = "REJECT"
| stats count(*) as attempts by srcAddr
| sort attempts desc

-- Traffic between specific subnets
filter srcAddr like /^10\.0\.1/ and dstAddr like /^10\.0\.2/
| stats sum(bytes) as bytesTransferred by srcAddr, dstAddr, dstPort
| sort bytesTransferred desc
```

### Flow Logs vs CloudTrail vs Packet Mirroring

| Service | What it Captures | Use Case |
|---------|-----------------|----------|
| **VPC Flow Logs** | IP traffic metadata (src, dst, port, action) | Network troubleshooting, security analysis |
| **CloudTrail** | AWS API calls | Who did what (audit) |
| **Traffic Mirroring** | Full packet capture | Deep packet inspection, IDS/IPS |

> **Exam Tip:** Flow logs capture metadata only (not packet contents). They DON'T capture: DNS queries to Route 53 Resolver, DHCP traffic, traffic to the instance metadata service (169.254.169.254), or traffic to the VPC DNS server.

---

## Part 3: Route 53

### Routing Policies

| Policy | Use Case | How It Works |
|--------|----------|--------------|
| **Simple** | Single resource | Returns one or all values (client picks randomly) |
| **Weighted** | A/B testing, blue/green | Distribute traffic by weight (e.g., 90/10) |
| **Latency** | Global users | Routes to lowest-latency region |
| **Failover** | Disaster recovery | Active-passive: primary fails -> standby |
| **Geolocation** | Compliance, localization | Routes based on user's geographic location |
| **Geoproximity** | Traffic shaping | Routes based on proximity with bias adjustment |
| **Multivalue Answer** | Simple load balancing | Returns multiple healthy IPs (up to 8) |
| **IP-based** | ISP optimization | Routes based on client's IP prefix |

### Health Checks

```bash
# HTTP health check
aws route53 create-health-check --caller-reference "$(date +%s)" \
  --health-check-config '{
    "IPAddress": "203.0.113.1",
    "Port": 80,
    "Type": "HTTP",
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'

# Calculated health check (combine multiple checks)
aws route53 create-health-check --caller-reference "$(date +%s)" \
  --health-check-config '{
    "Type": "CALCULATED",
    "HealthThreshold": 2,
    "ChildHealthChecks": ["HEALTH_CHECK_ID_1", "HEALTH_CHECK_ID_2", "HEALTH_CHECK_ID_3"]
  }'
```

### Route 53 Resolver

```
On-premises DNS <---> Route 53 Resolver Inbound Endpoint (on-prem resolves AWS names)
                <---> Route 53 Resolver Outbound Endpoint (AWS resolves on-prem names)
```

- **Inbound endpoint:** On-premises servers forward DNS queries TO AWS
- **Outbound endpoint:** AWS forwards DNS queries TO on-premises
- **Resolver rules:** Define which domains forward where

### Private Hosted Zones

```bash
# Create private hosted zone associated with VPC
aws route53 create-hosted-zone \
  --name "internal.mycompany.com" \
  --vpc "VPCRegion=us-east-1,VPCId=$VPC_ID" \
  --caller-reference "$(date +%s)" \
  --hosted-zone-config PrivateZone=true
```

> **Exam Tip:** Private hosted zones resolve DNS ONLY within associated VPCs. To share across VPCs, associate additional VPCs. For hybrid DNS (on-premises + AWS), use Resolver endpoints.

---

## Part 4: CloudFront & Edge Services

### CloudFront Key Concepts

| Concept | Description |
|---------|-------------|
| **Distribution** | The CDN configuration (domain name: d111111abcdef8.cloudfront.net) |
| **Origin** | Source of content (S3 bucket, ALB, custom HTTP server) |
| **Behavior** | Rules for how CloudFront handles requests (path pattern, caching, HTTPS) |
| **Edge Location** | Where content is cached (200+ locations worldwide) |
| **Regional Edge Cache** | Intermediate cache between edge and origin |
| **OAC** | Origin Access Control - restricts S3 access to CloudFront only |

### Create Distribution with S3 Origin

```bash
# Create OAC for secure S3 access
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "lab9-oac",
    "OriginAccessControlOriginType": "s3",
    "SigningBehavior": "always",
    "SigningProtocol": "sigv4"
  }' --query 'OriginAccessControl.Id' --output text)

# Create distribution
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "lab9-dist",
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "S3Origin",
        "DomainName": "my-website-bucket.s3.amazonaws.com",
        "OriginAccessControlId": "'$OAC_ID'",
        "S3OriginConfig": {"OriginAccessIdentity": ""}
      }]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "S3Origin",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "Compress": true,
      "AllowedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]}
    },
    "Enabled": true,
    "DefaultRootObject": "index.html",
    "Comment": "Lab 9 distribution"
  }'
```

### Cache Invalidation

```bash
# Invalidate specific files
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/index.html" "/images/*"
```

### CloudFront Functions vs Lambda@Edge

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|-------------|
| **Runtime** | JavaScript only | Node.js, Python |
| **Execution** | Viewer request/response only | All 4 events (viewer + origin) |
| **Duration** | < 1ms | Up to 30s (origin) |
| **Memory** | 2 MB | Up to 10 GB |
| **Network** | No | Yes |
| **Use case** | Header manipulation, URL rewrites, simple redirects | Complex logic, authentication, dynamic content |
| **Scale** | Millions of requests/sec | Thousands of requests/sec |

### CloudFront vs Global Accelerator

| Feature | CloudFront | Global Accelerator |
|---------|-----------|-------------------|
| **Purpose** | Content caching + delivery | Network performance optimization |
| **Caching** | Yes | No |
| **Protocol** | HTTP/HTTPS/WebSocket | TCP/UDP |
| **IP** | Domain name (CNAME) | 2 static anycast IPs |
| **Use case** | Static/dynamic web content | Gaming, IoT, VoIP, non-HTTP workloads |
| **Health checks** | Origin health | Endpoint health with instant failover |

---

## Part 5: Network Security

### AWS WAF

```bash
# Create a web ACL with rate limiting and SQL injection protection
aws wafv2 create-web-acl \
  --name "lab9-web-acl" \
  --scope REGIONAL \
  --default-action '{"Allow": {}}' \
  --rules '[
    {
      "Name": "RateLimit",
      "Priority": 1,
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "RateLimit"}
    },
    {
      "Name": "SQLInjection",
      "Priority": 2,
      "Statement": {
        "SqliMatchStatement": {
          "FieldToMatch": {"AllQueryArguments": {}},
          "TextTransformations": [{"Priority": 0, "Type": "URL_DECODE"}]
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "SQLi"}
    }
  ]' \
  --visibility-config '{"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "lab9-waf"}'
```

### Shield Standard vs Advanced

| Feature | Shield Standard | Shield Advanced |
|---------|----------------|-----------------|
| **Cost** | Free (included) | $3,000/month |
| **Protection** | L3/L4 DDoS | L3/L4/L7 DDoS |
| **Response team** | No | AWS DDoS Response Team (DRT) |
| **Cost protection** | No | Yes (credits for scaling during attack) |
| **Visibility** | Basic | Real-time metrics, attack diagnostics |
| **WAF included** | No | Yes (WAF charges included) |

### AWS Network Firewall

- Deployed in a dedicated firewall subnet
- Stateful and stateless rule groups
- Domain filtering (allow/block specific domains)
- IPS/IDS with Suricata-compatible rules
- Integrates with Firewall Manager for multi-account

---

## Part 6: Troubleshooting Checklists

### Instance Can't Reach the Internet

```
1. Is the instance in a public or private subnet?
   - Public: Check for public IP or Elastic IP
   - Private: Check for NAT Gateway in a public subnet

2. Check the subnet's route table:
   - Public: 0.0.0.0/0 -> IGW
   - Private: 0.0.0.0/0 -> NAT Gateway

3. Check the Internet Gateway is attached to the VPC

4. Check Security Group outbound rules (allow outbound traffic)

5. Check NACL:
   - Outbound: Allow traffic to destination
   - Inbound: Allow ephemeral ports (1024-65535) for response traffic

6. Check if NAT Gateway is in the same AZ or has cross-AZ route

7. Check if the instance has the correct IAM role (for AWS API calls)
```

### Can't SSH to Instance

```
1. Security Group: Port 22 allowed from YOUR IP?
2. NACL: Port 22 inbound allowed? Ephemeral ports outbound allowed?
3. Route Table: Route to IGW for public subnet?
4. Instance has public IP or Elastic IP?
5. Key pair matches? (.pem file permissions: chmod 400)
6. Instance is running (not stopped/terminated)?
7. OS-level firewall (iptables/firewalld) not blocking?
8. SSH service running on the instance?
```

### Application Timeout Between Services

```
1. Security Group of TARGET: Allows inbound on the application port from SOURCE's SG/IP?
2. Security Group of SOURCE: Allows outbound? (usually all outbound is allowed by default)
3. NACL of both subnets: Allows the traffic in BOTH directions?
4. Route tables: Can the subnets reach each other?
   - Same VPC: Local route (automatic)
   - Different VPCs: VPC peering or Transit Gateway needed?
5. DNS resolution: Can the source resolve the target's hostname?
   - Check private hosted zones
   - Check VPC DNS settings (enableDnsHostnames, enableDnsSupport)
6. Application listening on the correct port and interface (0.0.0.0, not 127.0.0.1)?
```

### S3 Access Denied from EC2

```
1. IAM Role: Does the instance profile have S3 permissions?
2. S3 Bucket Policy: Does it allow the role/account?
3. S3 Block Public Access: Is it blocking the request?
4. VPC Endpoint Policy: If using a VPC endpoint, does the policy allow the bucket?
5. KMS: If bucket uses SSE-KMS, does the role have kms:Decrypt permission?
6. Object-level permissions: Does the object ACL/ownership allow access?
7. STS: If cross-account, is the assume-role chain correct?
```

> **Exam Tip:** Networking questions follow a pattern: check from INSIDE OUT. Start at the instance (SG), then subnet (NACL, route table), then VPC (IGW/NAT), then external (DNS, firewall).

---

## Part 7: Review Questions

### Question 1

An EC2 instance in a private subnet can ping other instances in the VPC but cannot reach the internet. The subnet's route table has a route to a NAT Gateway. What should be checked FIRST?

**A)** Whether the instance has a public IP
**B)** Whether the NAT Gateway is in a public subnet with a route to an Internet Gateway
**C)** Whether the security group allows ICMP outbound
**D)** Whether the NACL allows all traffic

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The NAT Gateway itself must be in a public subnet that has a route to the IGW. If the NAT Gateway is in a private subnet, it can't reach the internet either. Private instances don't need public IPs (A) - that's the point of NAT. The instance can already ping within VPC, so SG outbound and local routing work (C, D may need checking but B is the most likely root cause).
</details>

---

### Question 2

A web application behind an ALB uses Security Groups and NACLs. Users report intermittent timeouts. The Security Group allows inbound port 80 and all outbound. The NACL allows inbound port 80 and outbound port 80. What is the issue?

**A)** The Security Group needs to allow inbound on ephemeral ports
**B)** The NACL needs to allow outbound on ephemeral ports (1024-65535) for response traffic
**C)** The ALB needs a public IP
**D)** The route table is misconfigured

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** NACLs are STATELESS. When the web server responds on port 80, the response goes back to the client on an ephemeral port (1024-65535). The NACL must explicitly allow outbound traffic on ephemeral ports. Security Groups are stateful and handle return traffic automatically (A is wrong - SGs don't need this). This is one of the most commonly tested NACL concepts on the exam.
</details>

---

### Question 3

A company wants to ensure that EC2 instances in a private subnet can access S3 without traversing the internet. What is the MOST cost-effective solution?

**A)** Create a NAT Gateway and route S3 traffic through it
**B)** Create a VPC Gateway Endpoint for S3
**C)** Create a VPC Interface Endpoint for S3
**D)** Assign public IPs to the instances

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Gateway Endpoints for S3 are free and add a route to the route table. Traffic stays within the AWS network. NAT Gateway (A) costs money and traffic goes through the internet. Interface Endpoints (C) work but cost hourly + per GB. Public IPs (D) require an IGW and expose instances.
</details>

---

### Question 4

A company has users in the US and Europe. They want to route users to the nearest AWS region for lowest latency. Which Route 53 routing policy should they use?

**A)** Geolocation routing
**B)** Latency-based routing
**C)** Weighted routing
**D)** Multivalue answer routing

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Latency-based routing measures network latency between the user and each AWS region and routes to the lowest latency. Geolocation (A) routes based on geographic location (country/continent), which doesn't always correlate with lowest latency. Weighted (C) distributes by percentage, not performance. Multivalue (D) returns multiple IPs for simple load distribution.
</details>

---

### Question 5

VPC Flow Logs show that all traffic to an EC2 instance on port 443 has action=REJECT. The Security Group allows inbound 443 from 0.0.0.0/0. What could cause this?

**A)** The Security Group outbound rules are blocking the response
**B)** A Network ACL is denying the traffic
**C)** The instance is stopped
**D)** The HTTPS certificate is expired

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Since the Security Group allows port 443, the REJECT must come from the NACL (evaluated before the SG for inbound traffic at the subnet level). NACLs have ordered rules - a lower-numbered DENY rule before the ALLOW rule would reject the traffic. SG outbound (A) doesn't cause inbound REJECTs (SGs are stateful). Stopped instances (C) wouldn't generate flow logs. Certificate issues (D) wouldn't cause network-level rejects.
</details>

---

### Question 6

What is the key difference between CloudFront and AWS Global Accelerator?

**A)** CloudFront is faster than Global Accelerator
**B)** CloudFront caches content at edge locations; Global Accelerator optimizes the network path without caching
**C)** Global Accelerator supports HTTPS; CloudFront does not
**D)** CloudFront provides static IPs; Global Accelerator uses domain names

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFront is a CDN that caches content at 400+ edge locations. Global Accelerator uses the AWS global network to optimize routing to your application endpoints (ALB, NLB, EC2, EIP) but does NOT cache. Global Accelerator provides 2 static anycast IPs (D is backwards). Both support HTTPS (C is wrong).
</details>

---

### Question 7

A developer deployed a CloudFront distribution with an S3 origin. Users get 403 Access Denied errors. The S3 bucket policy allows public read. What is the MOST likely issue?

**A)** The CloudFront distribution needs an SSL certificate
**B)** S3 Block Public Access is enabled on the bucket, overriding the bucket policy
**C)** The CloudFront cache hasn't been warmed up yet
**D)** The bucket is in a different region than the distribution

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** S3 Block Public Access (BPA) overrides any bucket policy that allows public access. Even if the bucket policy says "allow public read," BPA blocks it. Solution: Use Origin Access Control (OAC) instead of public access, and update the bucket policy to allow the CloudFront service principal. CloudFront works with buckets in any region (D is wrong).
</details>

---

### Question 8

An AWS WAF web ACL has two rules: Rule 1 (priority 1) allows all US traffic, Rule 2 (priority 2) blocks all traffic. A request from a US IP address arrives. What happens?

**A)** The request is blocked because Block takes priority over Allow
**B)** The request is allowed because Rule 1 matches first (lower priority number = higher priority)
**C)** Both rules are evaluated and the most restrictive action applies
**D)** The default action determines the outcome

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** WAF rules are evaluated in priority order (lowest number first). Rule 1 (priority 1) matches the US IP and allows it. Once a rule matches, evaluation stops. Rule 2 is never evaluated for this request. This is similar to NACL rule evaluation.
</details>

---

### Question 9

A company uses VPC peering between VPC-A (10.0.0.0/16) and VPC-B (172.16.0.0/16). An instance in VPC-A needs to access an instance in VPC-C (192.168.0.0/16) which is peered with VPC-B. Can the traffic transit through VPC-B?

**A)** Yes, VPC peering supports transitive routing
**B)** No, VPC peering is NOT transitive. VPC-A must peer directly with VPC-C or use Transit Gateway
**C)** Yes, if you add routes in VPC-B's route table
**D)** Yes, but only if all three VPCs are in the same region

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** VPC peering is NOT transitive. A->B and B->C does NOT mean A->C. For full connectivity, either create a direct peering between A and C, or use AWS Transit Gateway which supports transitive routing between all attached VPCs.
</details>

---

### Question 10

A Route 53 failover routing policy has a primary record pointing to an ALB in us-east-1 and a secondary record pointing to a static S3 website in us-west-2. The health check for the ALB shows healthy, but users report they're being routed to the S3 website. What should be investigated?

**A)** The TTL is too long and DNS is cached
**B)** The health check might be evaluating the wrong endpoint or the ALB health check configuration is incorrect
**C)** Route 53 failover doesn't support ALB endpoints
**D)** The S3 website has higher priority

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** If Route 53 shows the health check as healthy but users get the secondary, check: (1) Is the health check associated with the correct record? (2) Is the health check evaluating the correct IP/endpoint? (3) Are there DNS propagation delays? (4) Is the client caching DNS? Route 53 failover absolutely supports ALB (C is wrong). The primary always has priority when healthy (D is wrong).
</details>

---

## Cleanup Steps

```bash
# Delete VPC Endpoints
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $(aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$VPC_ID" --query 'VpcEndpoints[*].VpcEndpointId' --output text)

# Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_ID
sleep 60  # Wait for NAT Gateway deletion

# Release EIP
aws ec2 release-address --allocation-id $EIP_ID

# Delete subnets
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET

# Delete route tables (non-main only)
aws ec2 delete-route-table --route-table-id $PUBLIC_RT
aws ec2 delete-route-table --route-table-id $PRIVATE_RT

# Detach and delete IGW
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# Delete security groups (non-default)
aws ec2 delete-security-group --group-id $SG_ID

# Delete NACL (non-default)
aws ec2 delete-network-acl --network-acl-id $NACL_ID

# Delete VPC
aws ec2 delete-vpc --vpc-id $VPC_ID

# Delete flow logs
aws ec2 delete-flow-logs --flow-log-ids $(aws ec2 describe-flow-logs \
  --filter "Name=resource-id,Values=$VPC_ID" --query 'FlowLogs[*].FlowLogId' --output text)

# Delete log group
aws logs delete-log-group --log-group-name "/vpc/flow-logs"

# Delete CloudFront distribution (disable first, wait, then delete)
# Delete WAF web ACL
# Delete Route 53 health checks and records
```
