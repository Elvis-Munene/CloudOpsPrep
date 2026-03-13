# Domain 5: Networking and Content Delivery (18% of SOA-C03 Exam)

## AWS Certified SysOps Administrator – Associate (SOA-C03) Practice Questions

---

## Task 5.1: Networking Features and Connectivity

---
### Question 1

A company has an application running in a private subnet of a VPC. The application needs to download software updates from the internet. The operations team has deployed a NAT gateway in a public subnet, but instances in the private subnet still cannot reach the internet. The security groups on the instances allow all outbound traffic. What is the MOST likely cause of this issue?

**A)** The private subnet's route table does not have a route to the NAT gateway for internet-bound traffic
**B)** The NAT gateway does not have an Elastic IP address associated with it
**C)** The NAT gateway's security group is blocking outbound traffic
**D)** The instances need a public IP address to route through the NAT gateway

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** For instances in a private subnet to access the internet through a NAT gateway, the private subnet's route table must include a route that directs internet-bound traffic (0.0.0.0/0) to the NAT gateway. Without this route, traffic from the private subnet has no path to reach the NAT gateway. Option B is unlikely because a NAT gateway requires an Elastic IP at creation time and cannot be created without one. Option C is incorrect because NAT gateways do not have security groups — they are managed AWS resources. Option D is incorrect because the entire purpose of a NAT gateway is to allow private instances without public IPs to access the internet.
</details>

---
### Question 2

A SysOps administrator is configuring network access control lists (NACLs) for a subnet that hosts web servers. The NACL allows inbound traffic on port 443, but HTTPS responses are not reaching the clients. What should the administrator do to resolve this issue?

**A)** Add an outbound rule to the security group allowing traffic on port 443
**B)** Add an outbound NACL rule allowing traffic on ephemeral ports (1024-65535)
**C)** Modify the inbound NACL rule to also allow ephemeral port traffic
**D)** Disable the NACL and rely solely on security groups

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** NACLs are stateless, meaning they evaluate inbound and outbound traffic independently. When a client sends a request to port 443, the response traffic from the server uses an ephemeral port (typically in the range 1024-65535) as the source port. Since NACLs do not automatically allow return traffic (unlike security groups, which are stateful), an explicit outbound rule must be added to permit traffic on ephemeral ports. Option A is incorrect because security groups are stateful and would automatically allow return traffic if the inbound rule permits it. Option C is incorrect because the issue is with outbound traffic, not inbound. Option D is incorrect because disabling NACLs removes a layer of security and is not a best practice.
</details>

---
### Question 3

A company uses a VPC with multiple subnets across two Availability Zones. The operations team needs to allow resources in a private subnet to access Amazon S3 without traversing the internet. The solution must minimize data transfer costs. Which approach should the administrator take?

**A)** Deploy a NAT gateway and route S3 traffic through it
**B)** Create a VPC gateway endpoint for Amazon S3 and update the private subnet route tables
**C)** Create a VPC interface endpoint for Amazon S3 with a private DNS name
**D)** Set up a VPN connection to the S3 service endpoint

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** A VPC gateway endpoint for Amazon S3 provides private connectivity from a VPC to S3 without requiring an internet gateway, NAT gateway, or VPN connection. Gateway endpoints are free to create and use — there is no hourly charge or data processing charge, making them the most cost-effective option. The administrator simply creates the endpoint and adds a route to the relevant subnet route tables. Option A would work but incurs NAT gateway data processing charges ($0.045 per GB), which can be significant at scale. Option C refers to an interface endpoint, which does work with S3 but incurs hourly charges and per-GB data processing fees, making it more expensive than a gateway endpoint for this use case. Option D is unnecessary and overly complex for accessing an AWS service.
</details>

---
### Question 4

An operations engineer must enable private connectivity between an application in a VPC and a third-party SaaS service that is available through AWS PrivateLink. Which type of VPC endpoint should the engineer create?

**A)** Gateway endpoint
**B)** Interface endpoint
**C)** Gateway Load Balancer endpoint
**D)** NAT gateway endpoint

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS PrivateLink uses VPC interface endpoints to provide private connectivity to services. An interface endpoint creates an elastic network interface (ENI) with a private IP address in the specified subnet, serving as an entry point for traffic destined to the service. Interface endpoints support PrivateLink-powered services from AWS, third-party SaaS providers, and services hosted by other AWS accounts. Gateway endpoints only support Amazon S3 and DynamoDB and work through route table entries rather than ENIs. Gateway Load Balancer endpoints are used for inline traffic inspection with third-party appliances. There is no such thing as a "NAT gateway endpoint." Interface endpoints incur hourly and data processing charges but provide the required private connectivity through PrivateLink.
</details>

---
### Question 5

A company has three VPCs: VPC-A, VPC-B, and VPC-C. VPC-A is peered with VPC-B, and VPC-B is peered with VPC-C. Resources in VPC-A need to communicate with resources in VPC-C. The administrator configured route tables in VPC-A to send traffic destined for VPC-C through VPC-B, but the traffic is not flowing. What is the root cause?

**A)** The security groups in VPC-B are blocking the traffic
**B)** VPC peering does not support transitive routing between VPCs
**C)** The CIDR blocks of VPC-A and VPC-C are overlapping
**D)** VPC peering connections have a maximum of two VPCs

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** VPC peering does not support transitive routing. Even though VPC-A is peered with VPC-B and VPC-B is peered with VPC-C, traffic cannot transit through VPC-B to reach VPC-C. Each VPC peering connection is a one-to-one relationship, and traffic can only flow directly between the two peered VPCs. To enable communication between VPC-A and VPC-C, the administrator must create a direct peering connection between them or use AWS Transit Gateway, which supports transitive routing. Option A is incorrect because the traffic never reaches VPC-B's security groups due to the transitive routing limitation. Option C could be a valid issue, but the question states the root cause is the routing configuration through VPC-B. Option D is incorrect because there is no limit of two VPCs; a VPC can have multiple peering connections.
</details>

---
### Question 6

A company needs to protect its web applications from common exploits such as SQL injection and cross-site scripting (XSS). The operations team also wants to implement rate limiting to block IP addresses that send more than 2,000 requests per 5 minutes. Which combination of AWS services should the team use?

**A)** AWS Shield Standard and Amazon GuardDuty
**B)** AWS WAF with managed rules and rate-based rules
**C)** AWS Network Firewall with stateful rules
**D)** Amazon Route 53 Resolver DNS Firewall and AWS Shield Advanced

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS WAF (Web Application Firewall) is designed to protect web applications from common web exploits. AWS Managed Rules for WAF include rule groups that address SQL injection, XSS, and other OWASP Top 10 vulnerabilities. Additionally, AWS WAF supports rate-based rules that automatically block IP addresses that exceed a specified request threshold within a 5-minute window. This combination addresses both requirements in a single service. AWS Shield Standard provides DDoS protection but does not inspect application-layer content for SQL injection or XSS. Network Firewall operates at the VPC level for network-layer traffic inspection, not specifically for web application attacks behind an ALB or CloudFront. Route 53 Resolver DNS Firewall is for filtering DNS queries, not HTTP/HTTPS traffic.
</details>

---
### Question 7

A SysOps administrator is comparing NAT gateways and NAT instances for providing internet access to private subnet resources. The company requires high availability, minimal operational overhead, and support for more than 10 Gbps of bandwidth. Which solution meets these requirements?

**A)** A NAT instance deployed in an Auto Scaling group across multiple Availability Zones
**B)** A single NAT gateway, as it automatically spans multiple Availability Zones
**C)** A NAT gateway deployed in each Availability Zone with route table entries for each
**D)** A NAT instance using a c5n.18xlarge instance type for high bandwidth

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** NAT gateways are managed AWS resources that provide automatic high availability within a single Availability Zone, support up to 100 Gbps of bandwidth, and require no operational overhead for patching or maintenance. However, a NAT gateway is deployed in a single AZ. For a highly available architecture, a NAT gateway should be placed in each AZ, with the private subnet route tables in each AZ pointing to their local NAT gateway. Option A would require managing EC2 instances, OS patching, and custom health-check scripts, increasing operational overhead. Option B is incorrect because a NAT gateway does not automatically span multiple AZs — it resides in a single AZ. Option D would require managing an EC2 instance and would still not match a NAT gateway's managed availability and bandwidth capabilities.
</details>

---
### Question 8

A company wants to use Amazon Route 53 Resolver DNS Firewall to block DNS queries to known malicious domains from resources within their VPC. Which steps must the administrator perform? (Select TWO.)

**A)** Create a firewall rule group with rules that reference domain lists and associate the rule group with the VPC
**B)** Enable DNS query logging on the VPC's DHCP options set
**C)** Deploy Route 53 Resolver endpoints in the VPC
**D)** Configure domain lists containing the malicious domains to block
**E)** Attach an AWS WAF web ACL to the Route 53 hosted zone

<details>
<summary>Show Answer</summary>

**Correct Answers: A, D**

**Explanation:** To use Route 53 Resolver DNS Firewall, the administrator must first create domain lists that specify the domains to allow or block (Option D). Then, the administrator creates a firewall rule group containing rules that reference these domain lists and specify actions (ALLOW, BLOCK, or ALERT). The rule group must be associated with the VPC to take effect (Option A). Route 53 Resolver DNS Firewall works with the built-in Route 53 Resolver that is automatically available in every VPC — no additional Resolver endpoints need to be deployed (Option C is incorrect). DNS query logging is a separate feature and is not required for DNS Firewall to function (Option B). AWS WAF cannot be attached to Route 53 hosted zones (Option E is incorrect).
</details>

---
### Question 9

An operations team is evaluating VPC endpoint options for connecting to Amazon DynamoDB from a private subnet. The team wants the lowest-cost solution that does not require changes to the application code. Which type of VPC endpoint should they use?

**A)** Interface endpoint with private DNS enabled
**B)** Gateway endpoint with route table updates
**C)** Gateway Load Balancer endpoint
**D)** Interface endpoint without private DNS, using endpoint-specific DNS names

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** For DynamoDB, a VPC gateway endpoint is the lowest-cost option because gateway endpoints have no hourly charge and no data processing charge. The gateway endpoint is configured by adding a prefix list entry to the route table, and traffic to DynamoDB is automatically routed through the endpoint. No application code changes are needed because the public DynamoDB endpoint DNS names continue to work — traffic is simply routed differently at the network level. Interface endpoints for DynamoDB are also available but incur hourly charges (per AZ) and per-GB data processing fees. Gateway Load Balancer endpoints are used for third-party network appliance inspection, not for AWS service connectivity. When cost is the primary concern for S3 or DynamoDB access, gateway endpoints are always preferred.
</details>

---
### Question 10

A company is running workloads in a VPC and must ensure that all traffic between the VPC and supported AWS services stays within the AWS network. The security team requires that VPC endpoint policies be applied to restrict which S3 buckets can be accessed. Which configuration meets this requirement?

**A)** Create a gateway endpoint for S3 and attach a VPC endpoint policy that restricts access to specific S3 bucket ARNs
**B)** Configure a NAT gateway with an S3 bucket policy to restrict access
**C)** Use AWS PrivateLink with an interface endpoint and a bucket policy only
**D)** Create a gateway endpoint for S3 without any endpoint policy and rely on IAM policies alone

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** VPC gateway endpoints for S3 support endpoint policies, which are resource-based policies that control which AWS principals can use the endpoint and which S3 buckets they can access. By attaching a VPC endpoint policy to the gateway endpoint, the administrator can explicitly restrict access to only the specified S3 bucket ARNs. All traffic through the gateway endpoint stays within the AWS network and does not traverse the internet. Option B is incorrect because NAT gateways do not support endpoint policies, and traffic through a NAT gateway goes to the internet. Option C is partially correct in that interface endpoints also support policies, but the question asks about keeping traffic within the AWS network at the lowest cost, and gateway endpoints are free. Option D would not meet the requirement of restricting which S3 buckets can be accessed through the endpoint.
</details>

---
### Question 11

A company has IPv6-enabled resources in a private subnet that need to initiate outbound connections to IPv6 addresses on the internet, but the resources must not be reachable from the internet. Which component should the administrator add to the VPC?

**A)** A NAT gateway configured for IPv6
**B)** An internet gateway with IPv6 route restrictions
**C)** An egress-only internet gateway
**D)** A VPC endpoint for IPv6 internet access

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** An egress-only internet gateway is a horizontally scaled, redundant VPC component that allows outbound communication over IPv6 from instances in a VPC to the internet, while preventing the internet from initiating IPv6 connections to the instances. It is the IPv6 equivalent of a NAT gateway for IPv4. The administrator adds a route in the subnet's route table pointing ::/0 (all IPv6 traffic) to the egress-only internet gateway. Option A is incorrect because NAT gateways handle IPv4 traffic, not IPv6 (IPv6 addresses are globally unique and do not need NAT translation). Option B is incorrect because a standard internet gateway allows bidirectional traffic and cannot be restricted to outbound-only at the gateway level. Option D is incorrect because VPC endpoints are for accessing AWS services privately, not for general internet access.
</details>

---
### Question 12

A SysOps administrator needs to implement network-level filtering for a VPC that inspects traffic entering and leaving the VPC, including deep packet inspection of TLS-encrypted traffic. Which AWS service should the administrator use?

**A)** Security groups with detailed inspection rules
**B)** Network access control lists with stateful rules
**C)** AWS Network Firewall with TLS inspection configured
**D)** AWS WAF deployed at the VPC level

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** AWS Network Firewall is a managed service that provides network-level protection for VPCs. It supports stateful and stateless inspection, intrusion prevention (IPS), and TLS inspection, which allows it to decrypt, inspect, and re-encrypt TLS-encrypted traffic. Network Firewall is deployed in its own subnet within the VPC and inspects traffic as it passes through firewall endpoints. Security groups (Option A) operate at the instance level and only support allow rules based on IP, port, and protocol — they cannot perform deep packet inspection. NACLs (Option B) are stateless (not stateful) and only filter based on IP, port, and protocol. AWS WAF (Option D) operates at the application layer for HTTP/HTTPS traffic behind CloudFront or ALB, not at the VPC network level for all traffic types.
</details>

---
### Question 13

A company is deciding between AWS Shield Standard and AWS Shield Advanced. Their critical web application faces frequent DDoS attacks, and the team needs DDoS cost protection, access to the AWS Shield Response Team (SRT), and advanced attack visibility. Which statement about Shield Advanced is correct?

**A)** Shield Advanced is included at no additional cost with all AWS accounts
**B)** Shield Advanced provides cost protection for scaling charges resulting from DDoS attacks and access to the SRT for 24/7 support
**C)** Shield Advanced only protects Amazon CloudFront distributions and cannot be applied to Application Load Balancers
**D)** Shield Advanced replaces the need for AWS WAF entirely

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Shield Advanced provides enhanced DDoS protection with several key benefits beyond Shield Standard. These include DDoS cost protection, which provides credits for charges that result from DDoS-related scaling (such as EC2, ELB, CloudFront, Route 53, and Global Accelerator), and 24/7 access to the AWS Shield Response Team (SRT), who can assist during active attacks. Shield Advanced also provides near-real-time visibility into attacks and advanced attack diagnostics. Option A is incorrect because Shield Standard is free with all accounts, but Shield Advanced requires a monthly subscription fee. Option C is incorrect because Shield Advanced protects CloudFront, ALB, Elastic IP, Route 53, and Global Accelerator resources. Option D is incorrect because Shield Advanced complements WAF; in fact, Shield Advanced includes AWS WAF at no additional cost for protected resources.
</details>

---

## Task 5.2: DNS and Content Delivery

---
### Question 14

A company hosts its primary web application in the us-east-1 Region and a disaster recovery environment in eu-west-1. The operations team wants Route 53 to automatically route traffic to the DR environment if the primary environment becomes unhealthy. Which Route 53 routing policy should the administrator configure?

**A)** Weighted routing with 100% weight to the primary
**B)** Failover routing with health checks
**C)** Latency-based routing
**D)** Simple routing with multiple values

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Route 53 failover routing is specifically designed for active-passive disaster recovery scenarios. The administrator creates a primary record pointing to the us-east-1 environment and a secondary record pointing to the eu-west-1 environment. A health check is associated with the primary record, and when Route 53 detects that the primary is unhealthy, it automatically begins responding to DNS queries with the secondary record. Option A with weighted routing could work but requires manual intervention or complex health-check configurations to shift 100% of traffic. Option C with latency-based routing directs users to the lowest-latency Region but does not inherently provide failover behavior without additional configuration. Option D with simple routing returns all values randomly and does not support health-check-based failover between a primary and secondary.
</details>

---
### Question 15

A company wants to distribute traffic evenly between a blue and green deployment during a canary release. Initially, 90% of traffic should go to the blue environment and 10% to the green environment. Which Route 53 routing policy should the administrator use?

**A)** Simple routing
**B)** Geolocation routing
**C)** Weighted routing
**D)** Multivalue answer routing

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Route 53 weighted routing allows the administrator to assign relative weights to DNS records, which determines the proportion of traffic each record receives. For a canary deployment, the administrator creates two weighted records — one for the blue environment with a weight of 90 and one for the green environment with a weight of 10. Route 53 then responds to approximately 90% of DNS queries with the blue environment's IP and 10% with the green environment's IP. As confidence in the green deployment grows, the weights can be adjusted. Simple routing does not support traffic distribution between multiple endpoints. Geolocation routing directs traffic based on the geographic location of users, not traffic percentages. Multivalue answer routing returns up to eight random healthy records and does not support weighted distribution.
</details>

---
### Question 16

An operations team needs to configure a Route 53 DNS record that points to an Application Load Balancer. The team wants to use the zone apex (example.com) without a "www" prefix. Which record type should the administrator use?

**A)** A CNAME record pointing to the ALB DNS name
**B)** An alias A record pointing to the ALB
**C)** A TXT record with the ALB DNS name
**D)** An NS record delegating to the ALB

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Route 53 alias records are a special extension that allows the zone apex (also called the naked domain or root domain) to point to AWS resources such as ALBs, CloudFront distributions, S3 static website endpoints, and other Route 53 records. A CNAME record (Option A) cannot be used at the zone apex according to DNS standards (RFC 1034) — CNAME records can only be created for subdomains like www.example.com. Alias records also have the advantage of being free of charge for queries that resolve to AWS resources, whereas standard record queries incur charges. Additionally, alias records automatically reflect changes in the target resource's IP addresses. TXT records (Option C) store text information and cannot route traffic. NS records (Option D) delegate DNS authority and are not used for pointing to load balancers.
</details>

---
### Question 17

A global media company wants to serve content to users with the lowest possible latency. They have deployments in us-east-1, eu-west-1, and ap-southeast-1. Which Route 53 routing policy should the administrator configure to route each user to the nearest Region based on network conditions?

**A)** Geolocation routing
**B)** Geoproximity routing
**C)** Latency-based routing
**D)** Failover routing

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Latency-based routing in Route 53 measures the latency between the user and each AWS Region and directs the user to the Region that provides the lowest latency. Route 53 uses a latency database that is updated regularly based on network measurements between end users and AWS Regions. This is ideal for a multi-Region deployment where the goal is to minimize response time. Geolocation routing (Option A) routes traffic based on the geographic location of the user (continent or country), but this does not always correspond to the lowest-latency Region. Geoproximity routing (Option B) routes based on geographic distance and optional bias values, which is useful when you want to shift traffic between Regions but may not reflect actual network latency. Failover routing (Option D) is for DR scenarios, not for latency-based optimization.
</details>

---
### Question 18

A SysOps administrator has configured a CloudFront distribution with an S3 bucket as the origin. Users are receiving 403 Access Denied errors when trying to access objects. The administrator wants to ensure that only CloudFront can access the S3 bucket. What should the administrator configure?

**A)** Enable S3 Block Public Access and use a CloudFront Origin Access Control (OAC) with a bucket policy
**B)** Make the S3 bucket public and add a CloudFront header check
**C)** Create a VPC endpoint for S3 and route CloudFront through the VPC
**D)** Use signed URLs for every request to the S3 bucket

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** CloudFront Origin Access Control (OAC) is the recommended mechanism for restricting access to S3 origins. OAC uses AWS Signature Version 4 (SigV4) to sign requests from CloudFront to S3. The administrator creates an OAC, associates it with the CloudFront distribution's origin, and updates the S3 bucket policy to allow access only from the CloudFront distribution's service principal with the specific distribution ID. S3 Block Public Access remains enabled, ensuring no direct public access to the bucket. Option B is insecure because making the bucket public exposes it to direct access. Option C is incorrect because CloudFront does not operate within a VPC; it is an edge service. Option D would add unnecessary complexity because signed URLs are for restricting access to CloudFront content for specific users, not for securing the origin connection.
</details>

---
### Question 19

A company has a CloudFront distribution with an ALB origin in us-east-1 and wants to configure automatic failover to an ALB in eu-west-1 if the primary origin becomes unavailable. Which CloudFront feature should the administrator use?

**A)** Route 53 failover routing in front of CloudFront
**B)** CloudFront origin groups with a primary and secondary origin
**C)** CloudFront Lambda@Edge to detect failures and redirect
**D)** CloudFront multiple cache behaviors with path-based routing

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFront origin groups allow the administrator to configure origin failover by specifying a primary origin and a secondary origin. When CloudFront receives an error response from the primary origin (such as HTTP 500, 502, 503, or 504 status codes, or a connection timeout), it automatically routes the request to the secondary origin. This failover happens at the CloudFront edge, making it faster than DNS-based failover approaches. The administrator creates an origin group, designates the us-east-1 ALB as the primary and the eu-west-1 ALB as the secondary, and specifies which HTTP error codes trigger failover. Option A would work but introduces DNS TTL delays. Option C adds unnecessary complexity. Option D routes based on URL paths, not health status of origins.
</details>

---
### Question 20

A company needs to restrict access to premium video content served through CloudFront. Only authenticated users who have paid for a subscription should be able to stream the content. The content consists of hundreds of files served through HLS streaming. Which approach should the administrator use?

**A)** CloudFront signed URLs for each video file
**B)** CloudFront signed cookies
**C)** S3 pre-signed URLs bypassing CloudFront
**D)** Security groups on CloudFront to restrict access by IP

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFront signed cookies are ideal when you need to provide access to multiple restricted files, such as all the files in an HLS video stream. With signed cookies, the user's browser sends the cookie with every request to CloudFront, and CloudFront validates the signature before serving the content. This is more practical than signed URLs for streaming scenarios where content consists of many small segment files. Signed URLs (Option A) would require generating a unique URL for each segment file in the HLS stream, which is impractical for video streaming with hundreds or thousands of segments. S3 pre-signed URLs (Option C) would bypass CloudFront, losing the benefits of edge caching and CDN delivery. Option D is incorrect because CloudFront does not have security groups — it is not an EC2-based resource. Signed cookies support both canned and custom policies for controlling access duration and IP restrictions.
</details>

---
### Question 21

A company has enabled Route 53 query logging to monitor DNS queries for their hosted zone. Where are the query logs delivered?

**A)** Amazon S3 bucket in the same Region as the hosted zone
**B)** Amazon CloudWatch Logs in the us-east-1 Region
**C)** AWS CloudTrail in the Region where Route 53 is configured
**D)** Amazon Kinesis Data Firehose delivery stream

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Route 53 query logging sends DNS query logs to Amazon CloudWatch Logs. For public hosted zones, the log group must be in the us-east-1 Region, regardless of where the DNS queries originate. The logs include information such as the domain or subdomain that was requested, the date and timestamp, the DNS record type (A, AAAA, etc.), the Route 53 edge location that responded to the query, and the DNS response code. This is different from Route 53 Resolver query logging, which can log queries from resources within VPCs and can send logs to CloudWatch Logs, S3, or Kinesis Data Firehose. Option A is incorrect because query logs for public hosted zones go to CloudWatch Logs, not directly to S3. Option C is incorrect because CloudTrail logs Route 53 API calls, not DNS queries. Option D is available for Resolver query logging, but not for hosted zone query logging.
</details>

---
### Question 22

A company wants to improve the availability and performance of a TCP-based application that runs in a single AWS Region. The application serves users globally. Users in distant regions report high latency. The team wants to use static IP addresses for the application endpoint. Which AWS service should the administrator use?

**A)** Amazon CloudFront with TCP support
**B)** AWS Global Accelerator
**C)** Amazon Route 53 with latency-based routing
**D)** Elastic Load Balancing with cross-Region support

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Global Accelerator provides two static anycast IP addresses that serve as a fixed entry point to the application. It uses the AWS global network to route traffic from users to the optimal AWS endpoint, reducing latency by avoiding the congestion of the public internet. Global Accelerator supports TCP and UDP protocols, making it suitable for non-HTTP applications. Traffic enters the AWS network at the edge location nearest to the user and is routed over the AWS backbone to the application endpoint. CloudFront (Option A) is a CDN optimized for HTTP/HTTPS content delivery and does not support arbitrary TCP protocols. Route 53 latency-based routing (Option C) directs users to the lowest-latency Region but does not provide static IP addresses or optimize the network path. ELB (Option D) does not provide cross-Region load balancing natively.
</details>

---
### Question 23

A SysOps administrator is configuring a Route 53 routing policy and needs to return multiple IP addresses to DNS queries while also performing health checks to exclude unhealthy endpoints from responses. The application does not require geographic or latency-based routing. Which routing policy should the administrator use?

**A)** Simple routing with multiple values in a single record
**B)** Multivalue answer routing
**C)** Weighted routing with equal weights
**D)** Geoproximity routing with zero bias

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** Multivalue answer routing allows Route 53 to return up to eight healthy records in response to each DNS query. Each record can have an associated health check, and Route 53 only returns records that pass their health checks. This makes it a good fit when you want to return multiple IP addresses while ensuring that unhealthy endpoints are excluded from responses. Simple routing (Option A) can return multiple values in a single record, but it does not support health checks on individual values — if you associate a health check, it applies to the entire record. Weighted routing (Option C) with equal weights would work, but Route 53 would return only one record per query rather than multiple records. Geoproximity routing (Option D) is designed for geographic traffic shifting and is unnecessarily complex for this use case.
</details>

---
### Question 24

A company uses geolocation routing in Route 53 to direct European users to an EU-based endpoint and US users to a US-based endpoint. A user in Japan reports that they cannot access the application. What is the MOST likely cause?

**A)** Route 53 geolocation routing does not support Asian regions
**B)** There is no default geolocation record configured
**C)** The health check for the US endpoint is failing
**D)** The TTL on the DNS record is too high

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** With Route 53 geolocation routing, DNS queries are answered based on the geographic location of the user. If a user's location does not match any configured geolocation record, Route 53 returns no answer unless a default record is configured. Since the company only configured records for Europe and the US, and there is no default record, users from other locations (like Japan) receive no DNS response, making the application inaccessible. The solution is to create a default geolocation record that serves as a catch-all for locations not explicitly mapped. Option A is incorrect because geolocation routing supports all geographic locations. Option C would affect US users, not specifically Japanese users. Option D would affect DNS caching but would not prevent resolution entirely.
</details>

---
### Question 25

A SysOps administrator is configuring a CloudFront distribution with multiple origins. Static assets (images, CSS, JavaScript) should be served from an S3 bucket, while API requests should be forwarded to an Application Load Balancer. How should the administrator configure this?

**A)** Create two separate CloudFront distributions, one for each origin
**B)** Configure a single CloudFront distribution with multiple cache behaviors using path patterns
**C)** Use Route 53 weighted routing to split traffic between S3 and ALB
**D)** Use Lambda@Edge to inspect requests and redirect to the appropriate origin

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFront cache behaviors allow the administrator to configure how CloudFront processes requests based on URL path patterns. The administrator creates a single distribution with two origins (S3 bucket and ALB) and defines cache behaviors with path patterns such as /api/* for the ALB origin and a default behavior (*) for the S3 origin. Each behavior can have its own caching settings, allowed HTTP methods, and viewer protocol policy. This is the standard and recommended approach for serving content from multiple origins through a single CloudFront domain. Option A would require users to use different domain names for different content types. Option C would randomly distribute all requests without path-based logic. Option D adds unnecessary complexity when native cache behaviors provide this functionality.
</details>

---
### Question 26

A company wants to use Route 53 to route users to the geographically closest endpoint but also wants the ability to shift more traffic to a newly expanded Region by using a bias value. Which Route 53 routing policy supports this?

**A)** Geolocation routing
**B)** Latency-based routing
**C)** Geoproximity routing
**D)** Weighted routing

<details>
<summary>Show Answer</summary>

**Correct Answer: C**

**Explanation:** Route 53 geoproximity routing routes traffic based on the geographic location of users and resources, with the ability to shift traffic by applying a bias value. A positive bias expands the geographic area from which traffic is routed to a resource, effectively attracting more traffic. A negative bias shrinks the area, sending less traffic to that resource. This makes geoproximity routing ideal when you want geographic-based routing with the flexibility to gradually shift traffic to specific Regions. Geolocation routing (Option A) uses fixed country or continent boundaries and does not support bias-based traffic shifting. Latency-based routing (Option B) routes based on measured network latency, not geography. Weighted routing (Option D) distributes traffic by percentage but does not consider geographic proximity.
</details>

---

## Task 5.3: Troubleshoot Connectivity

---
### Question 27

An operations engineer is troubleshooting connectivity issues where an EC2 instance in a private subnet cannot reach an instance in another private subnet within the same VPC. The security groups allow the necessary traffic. Which should the engineer check NEXT?

**A)** The VPC peering connection between the subnets
**B)** The network access control lists (NACLs) on both subnets
**C)** The internet gateway attached to the VPC
**D)** The NAT gateway configuration

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** When security groups are confirmed to allow the necessary traffic, the next layer of network filtering to check is the NACLs. Since both subnets are within the same VPC, no VPC peering is needed (Option A is incorrect — peering is for inter-VPC communication). NACLs are stateless and evaluated at the subnet level. The engineer must verify that inbound rules on the destination subnet's NACL allow the traffic AND that outbound rules on both subnets' NACLs allow the traffic, including ephemeral port ranges for return traffic. Unlike security groups, which are stateful and automatically allow return traffic, NACLs require explicit rules for both directions. The internet gateway (Option C) and NAT gateway (Option D) are not involved in intra-VPC private subnet communication.
</details>

---
### Question 28

A SysOps administrator needs to analyze network traffic patterns and identify which EC2 instances are communicating with unexpected IP addresses. The administrator wants to capture metadata about all network flows without impacting application performance. Which solution should the administrator implement?

**A)** Enable VPC Flow Logs at the VPC level and send them to CloudWatch Logs
**B)** Install packet capture agents on all EC2 instances
**C)** Enable CloudTrail logging for EC2 network API calls
**D)** Configure an Application Load Balancer to log all traffic

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** VPC Flow Logs capture metadata about IP traffic going to and from network interfaces in a VPC. When enabled at the VPC level, flow logs capture traffic for all network interfaces in all subnets. Flow logs record information including source and destination IP addresses, source and destination ports, protocol, number of packets, number of bytes, start and end times, and the action taken (ACCEPT or REJECT). Flow logs do not capture actual packet content, so they do not impact application performance or introduce security concerns about capturing sensitive data. They can be sent to CloudWatch Logs, S3, or Kinesis Data Firehose. Option B would impact performance and require agent management. Option C only logs API calls, not network traffic. Option D only captures HTTP/HTTPS traffic that passes through the ALB, missing instance-to-instance traffic.
</details>

---
### Question 29

A company's CloudFront distribution is serving stale content after the origin was updated. The operations team confirmed the origin has the new content. The cache behavior has a default TTL of 86400 seconds (24 hours). What is the FASTEST way to serve the updated content to all users?

**A)** Wait for the TTL to expire naturally
**B)** Create a CloudFront invalidation for the affected paths
**C)** Delete and recreate the CloudFront distribution
**D)** Modify the security group to block old cached content

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFront invalidation removes cached copies of objects from CloudFront edge locations before the TTL expires. The administrator can invalidate specific file paths (e.g., /images/logo.png) or use wildcard paths (e.g., /images/*) to invalidate multiple objects. This forces CloudFront to fetch fresh content from the origin on the next request. The first 1,000 invalidation paths per month are free; additional paths incur a charge. Option A would work eventually but would take up to 24 hours given the TTL. Option C is extremely disruptive and unnecessary — it would cause downtime and require DNS propagation. Option D is incorrect because CloudFront is not controlled by security groups. For long-term cache management, consider using versioned file names (e.g., logo-v2.png) to avoid cache invalidation needs entirely.
</details>

---
### Question 30

A company has a Site-to-Site VPN connection from their on-premises data center to AWS. The VPN connection is experiencing intermittent connectivity issues and throughput is limited to approximately 1.25 Gbps. The company needs a more reliable connection with up to 10 Gbps of dedicated bandwidth. Which solution should the administrator recommend?

**A)** Add a second Site-to-Site VPN connection for redundancy
**B)** Migrate to AWS Direct Connect with a dedicated connection
**C)** Increase the VPN tunnel MTU to improve throughput
**D)** Use a larger EC2 instance as a VPN endpoint

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Direct Connect provides a dedicated private connection between an on-premises data center and AWS. Direct Connect offers dedicated connections of 1 Gbps, 10 Gbps, or 100 Gbps, providing consistent network performance, reduced bandwidth costs for large data transfers, and lower latency compared to VPN over the internet. Site-to-Site VPN connections are limited to approximately 1.25 Gbps per tunnel and depend on public internet quality, leading to variable performance. Option A would add redundancy but would not significantly increase single-flow throughput. Option C is incorrect because MTU adjustments do not increase the fundamental throughput limit of VPN tunnels. Option D is incorrect for a managed Site-to-Site VPN, which does not use customer EC2 instances as endpoints. Direct Connect also supports private virtual interfaces for VPC connectivity and public virtual interfaces for AWS public services.
</details>

---
### Question 31

A SysOps administrator is analyzing VPC Flow Logs and sees the following log entry:

`2 123456789012 eni-abc123 10.0.1.5 10.0.2.10 49152 3306 6 20 4000 1620000000 1620000060 ACCEPT OK`

Which statement correctly describes this traffic flow?

**A)** A database server at 10.0.2.10 is sending data to 10.0.1.5 on port 49152
**B)** An application at 10.0.1.5 is connecting to a MySQL database at 10.0.2.10 on port 3306
**C)** Traffic is being rejected between the two IP addresses
**D)** The flow log is capturing ICMP ping traffic

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** The VPC Flow Log format in this entry shows: version (2), account ID, ENI ID, source IP (10.0.1.5), destination IP (10.0.2.10), source port (49152), destination port (3306), protocol (6 = TCP), packets (20), bytes (4000), start time, end time, action (ACCEPT), and log status (OK). Port 3306 is the default MySQL port, indicating that the instance at 10.0.1.5 is connecting to a MySQL database at 10.0.2.10. The source port 49152 is an ephemeral port used by the client. The action is ACCEPT, meaning the traffic was allowed (not rejected as Option C states). Protocol 6 is TCP, not ICMP (which would be protocol 1), so Option D is incorrect. Option A reverses the direction — the source IP initiates the connection to the destination IP.
</details>

---
### Question 32

An operations team is troubleshooting an issue where an application behind an Application Load Balancer intermittently returns HTTP 502 Bad Gateway errors. CloudWatch metrics show the ALB is healthy, but the target group shows some targets as unhealthy. Which log source should the team analyze to identify the root cause?

**A)** VPC Flow Logs
**B)** ALB access logs
**C)** CloudTrail logs
**D)** Route 53 query logs

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** ALB access logs provide detailed information about every request processed by the load balancer, including the target IP address, response code from the target, processing times, and error reasons. For HTTP 502 errors, the ALB access logs will show which backend targets are returning errors and the specific error codes, helping the team identify whether targets are crashing, timing out, or returning malformed responses. The logs include fields like target_processing_time, elb_status_code, target_status_code, and actions_executed. VPC Flow Logs (Option A) only capture layer-3/4 metadata and do not include HTTP-level information. CloudTrail (Option C) logs AWS API calls, not application traffic. Route 53 query logs (Option D) capture DNS queries and are not relevant to HTTP errors at the load balancer level.
</details>

---
### Question 33

A company has a Transit Gateway connecting five VPCs and an on-premises network via a VPN attachment. Resources in VPC-A can communicate with all other VPCs but cannot reach the on-premises network. What should the administrator check?

**A)** The VPC peering connection between VPC-A and the on-premises network
**B)** The Transit Gateway route table for a route to the on-premises CIDR via the VPN attachment
**C)** The internet gateway in VPC-A for on-premises routing
**D)** The NAT gateway for translating on-premises IP addresses

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** AWS Transit Gateway uses route tables to control traffic flow between its attachments (VPCs and VPN connections). If VPC-A can reach other VPCs but not the on-premises network, the Transit Gateway route table associated with VPC-A's attachment likely does not have a route to the on-premises CIDR pointing to the VPN attachment. The administrator should check: (1) the Transit Gateway route table for a route to the on-premises CIDR via the VPN attachment, (2) that the VPN attachment is associated with the correct route table, and (3) that route propagation is enabled for the VPN attachment. Option A is incorrect because VPC peering is not used with Transit Gateway — Transit Gateway provides transitive connectivity. Option C is incorrect because internet gateways are for internet access, not on-premises connectivity. Option D is incorrect because NAT is not needed for Transit Gateway routing.
</details>

---
### Question 34

A SysOps administrator notices that a CloudFront distribution is returning a high cache miss rate. The distribution serves dynamic content with query strings that include user-specific parameters (e.g., ?userId=123&sessionId=abc&page=products). Only the "page" query string affects the actual content returned. How should the administrator optimize the cache hit ratio?

**A)** Disable query string forwarding entirely
**B)** Configure the cache behavior to forward only the "page" query string parameter to the origin and include it in the cache key
**C)** Increase the TTL to 7 days
**D)** Enable compression on the CloudFront distribution

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** CloudFront uses cache keys to identify cached objects. By default, if all query strings are forwarded and included in the cache key, each unique combination of query parameters creates a separate cached object. Since userId and sessionId are unique per user, every request generates a different cache key even if the content (based on "page") is the same. By configuring the cache behavior to include only the "page" query string in the cache key (using a cache policy), CloudFront will cache one copy of each page regardless of user-specific parameters. The other query strings can still be forwarded to the origin using an origin request policy if needed. Option A would break the application if the origin needs the query strings. Option C would increase cache duration but not fix the fundamental cache key problem. Option D helps with transfer size but does not improve cache hit ratios.
</details>

---
### Question 35

A company has established an AWS Direct Connect connection with a private virtual interface to their VPC. The connection was working but suddenly lost connectivity. The Direct Connect console shows the connection state as "available" but the virtual interface BGP status is "down." What should the administrator investigate?

**A)** The VPC internet gateway configuration
**B)** The BGP session parameters, including ASN, authentication key, and peer IP addresses on both sides
**C)** The Route 53 health checks for the Direct Connect endpoint
**D)** The CloudFront origin configuration

<details>
<summary>Show Answer</summary>

**Correct Answer: B**

**Explanation:** When a Direct Connect connection is "available" but the BGP status is "down," it means the physical connection is healthy but the BGP peering session between the customer router and the AWS Direct Connect router has failed. The administrator should investigate the BGP configuration parameters on the customer router, including the BGP Autonomous System Number (ASN), the BGP authentication key (MD5 password), the BGP peer IP addresses, and whether the customer router is advertising the correct prefixes. Common causes include a router reboot that did not restore BGP configuration, an expired BGP authentication key, or network interface issues on the customer router. Option A is irrelevant because Direct Connect with a private virtual interface does not use an internet gateway. Option C is incorrect because Route 53 health checks do not monitor Direct Connect BGP sessions. Option D has no relation to Direct Connect connectivity.
</details>

---
### Question 36

A SysOps administrator is troubleshooting why an EC2 instance can send traffic to the internet but cannot receive inbound traffic from the internet. The instance has a public IP address, and the internet gateway is attached to the VPC. The security group allows inbound HTTP traffic on port 80. What should the administrator check?

**A)** The subnet's NACL inbound rules for port 80 and outbound rules for ephemeral ports
**B)** Whether the instance has an Elastic IP instead of a public IP
**C)** The NAT gateway configuration
**D)** The VPC peering connection

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** Since the security group allows inbound HTTP on port 80, the next layer to investigate is the subnet's NACL. NACLs are stateless, so even if an inbound rule allows port 80 traffic, the return traffic must be explicitly allowed by an outbound rule. The outbound NACL rule must permit traffic on ephemeral ports (1024-65535) for the response traffic. The administrator should verify: (1) the inbound NACL rule allows traffic on port 80 from 0.0.0.0/0, (2) the outbound NACL rule allows traffic on ephemeral ports to 0.0.0.0/0, and (3) the route table has a route for 0.0.0.0/0 pointing to the internet gateway. Option B is unlikely to be the issue since both Elastic IPs and auto-assigned public IPs allow inbound traffic. Option C is for outbound internet access from private subnets. Option D is for inter-VPC connectivity, not internet access.
</details>

---
### Question 37

A company uses AWS WAF on their Application Load Balancer and wants to review detailed logs of all web requests evaluated by WAF rules, including which rules were triggered. Where should the administrator send WAF logs for analysis?

**A)** Send WAF logs to an Amazon S3 bucket with the prefix aws-waf-logs-
**B)** WAF logs are only available in the WAF console dashboard
**C)** Send WAF logs to Amazon RDS for SQL-based analysis
**D)** WAF logs are automatically stored in CloudTrail

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** AWS WAF full logging can be configured to send detailed logs to Amazon S3 (with a bucket name that must start with aws-waf-logs-), Amazon CloudWatch Logs (with a log group name that must start with aws-waf-logs-), or Amazon Kinesis Data Firehose (with a delivery stream name that must start with aws-waf-logs-). The logs contain detailed information about each request, including the client IP, URI, HTTP method, headers, and which WAF rules were matched. S3 is a common destination for long-term storage and analysis using Amazon Athena. Option B is incorrect because while the WAF console provides sampled requests, full detailed logging requires configuration. Option C is incorrect because WAF cannot send logs directly to RDS. Option D is incorrect because CloudTrail logs WAF API configuration changes, not web request evaluations.
</details>

---
### Question 38

An operations team needs to monitor the network performance of their VPC, including packet loss and latency between Availability Zones. Which CloudWatch feature should the administrator use?

**A)** CloudWatch custom metrics with a cron-based ping script
**B)** CloudWatch Internet Monitor
**C)** VPC Flow Logs with CloudWatch Logs Insights queries
**D)** CloudWatch Network Monitor or use Transit Gateway network manager for inter-AZ metrics

<details>
<summary>Show Answer</summary>

**Correct Answer: D**

**Explanation:** For monitoring network performance metrics such as packet loss and latency within AWS, CloudWatch Network Monitor and Transit Gateway Network Manager provide network performance insights. CloudWatch Network Monitor can measure network health metrics including latency and packet loss for AWS network paths. Additionally, if using Transit Gateway, the Network Manager provides a centralized view of network metrics across the global network. VPC Flow Logs (Option C) capture metadata about traffic flows (IPs, ports, bytes, accept/reject) but do not measure latency or packet loss between AZs. CloudWatch Internet Monitor (Option B) measures internet-facing performance from end users to AWS services, not inter-AZ VPC performance. While Option A could work with custom scripts, it adds operational overhead and is not the AWS-native approach. Native AWS monitoring tools provide these metrics without custom implementation.
</details>

---
### Question 39

A company has a hybrid architecture with an AWS Site-to-Site VPN as the primary connection and AWS Direct Connect being provisioned as the future primary. The VPN uses static routing. The operations team reports that failover between VPN tunnels is not automatic when one tunnel goes down. What should the administrator do to enable automatic failover?

**A)** Configure the VPN connection to use dynamic routing with BGP instead of static routing
**B)** Increase the VPN tunnel timeout values
**C)** Create a second VPN connection to a different virtual private gateway
**D)** Enable Enhanced Networking on the customer gateway device

<details>
<summary>Show Answer</summary>

**Correct Answer: A**

**Explanation:** AWS Site-to-Site VPN connections have two tunnels for redundancy. With static routing, the customer gateway device must be manually configured to detect tunnel failure and switch to the secondary tunnel, which may not happen automatically depending on the device. With dynamic routing using BGP (Border Gateway Protocol), the VPN endpoints automatically detect tunnel failures through BGP keepalive mechanisms and route traffic through the healthy tunnel. BGP provides automatic failover because when a BGP session on one tunnel goes down, routes are withdrawn and traffic shifts to the other tunnel. Option B does not address the fundamental routing issue. Option C adds complexity but does not solve the static routing failover problem. Option D relates to EC2 networking performance, not VPN tunnel failover. For production environments, dynamic BGP routing is strongly recommended for Site-to-Site VPN.
</details>

---
### Question 40

A SysOps administrator is troubleshooting a CloudFront distribution that returns HTTP 504 Gateway Timeout errors. The origin is an Application Load Balancer. CloudFront logs show that the errors occur during peak traffic periods. The origin's EC2 instances are at high CPU utilization. Which actions should the administrator take to resolve this issue? (Select TWO.)

**A)** Increase the CloudFront origin response timeout value
**B)** Scale the backend EC2 instances behind the ALB to handle the load
**C)** Change the Route 53 record TTL to a lower value
**D)** Disable CloudFront caching entirely
**E)** Increase the CloudFront cache TTL and optimize cache hit ratio to reduce origin load

<details>
<summary>Show Answer</summary>

**Correct Answers: B, E**

**Explanation:** HTTP 504 errors from CloudFront indicate that the origin did not respond within the configured timeout period. Since the errors correlate with peak traffic and high CPU on origin instances, two complementary approaches are needed. First, scale the backend instances (Option B) by adding more instances to the Auto Scaling group or increasing instance sizes to handle peak load, directly addressing the capacity constraint. Second, optimize the CloudFront cache (Option E) by increasing TTL values and improving the cache hit ratio through proper cache key configuration, which reduces the number of requests that reach the origin. Option A would only mask the symptom by waiting longer for slow responses without addressing the root cause. Option C (Route 53 TTL) has no impact on origin response times. Option D would worsen the problem by sending all requests to the already overloaded origin.
</details>

---

## Summary: Domain 5 Task Coverage

| Task | Focus Area | Questions |
|------|-----------|-----------|
| **5.1** | Networking features and connectivity | Q1-Q13 |
| **5.2** | DNS and content delivery | Q14-Q26 |
| **5.3** | Troubleshoot connectivity | Q27-Q40 |

### Key Concepts Covered:
- NACL stateless vs Security Group stateful behavior (Q2, Q27, Q36)
- Ephemeral ports in NACLs (Q2, Q36)
- VPC peering limitations — no transitive routing (Q5)
- Transit Gateway for transitive connectivity (Q33)
- NAT gateway vs NAT instance (Q7)
- VPC Flow Log format and fields (Q28, Q31)
- Route 53 alias vs CNAME records (Q16)
- CloudFront signed URLs vs signed cookies (Q20)
- CloudFront origin failover (Q19)
- WAF rate-based rules (Q6)
- Shield Advanced protections (Q13)
- Direct Connect vs Site-to-Site VPN (Q30, Q35, Q39)
- VPC endpoint policies (Q10)
- IPv6 egress-only internet gateway (Q11)
- Gateway vs Interface endpoints (Q3, Q4, Q9)
- CloudFront caching and invalidation (Q29, Q34)
- Route 53 routing policies (Q14-Q17, Q23, Q24, Q26)
