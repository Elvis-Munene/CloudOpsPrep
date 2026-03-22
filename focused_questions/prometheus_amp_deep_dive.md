# Amazon Managed Service for Prometheus (AMP) - Deep Dive Questions (SOA-C03)

### Question 1
A company is migrating from self-managed Prometheus to Amazon Managed Service for Prometheus (AMP). Their EKS clusters currently scrape metrics from 500+ pods across 10 namespaces. Which component is responsible for sending metrics from EKS to AMP, and what protocol does it use?

A) AMP automatically discovers and scrapes metrics from EKS pods using service discovery annotations
B) Prometheus remote write protocol from an in-cluster Prometheus server or AWS Distro for OpenTelemetry (ADOT) collector to AMP workspace
C) CloudWatch Container Insights agent forwards metrics from EKS to AMP using the PutMetricData API
D) VPC PrivateLink connection allows AMP to directly access pod endpoints and scrape metrics using HTTP

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- AMP is a managed Prometheus-compatible service that stores and queries metrics but does NOT scrape metrics directly
- Ingestion uses Prometheus remote write protocol (HTTP-based API)
- Two main ingestion patterns:
  1. **In-cluster Prometheus**: Traditional Prometheus server scrapes pods locally, then remote-writes to AMP
  2. **ADOT Collector**: AWS Distro for OpenTelemetry collector scrapes metrics and sends to AMP (recommended approach)

**ADOT Collector Deployment:**
```yaml
# Deployed as DaemonSet or Deployment in EKS
# Scrapes /metrics endpoints from pods
# Uses Prometheus receiver to scrape
# Uses remote write exporter to send to AMP
```

**Why other options are incorrect:**
- **A**: AMP does not scrape metrics directly from sources. It's a storage and query service only. Scraping happens in-cluster
- **C**: Container Insights sends metrics to CloudWatch, not AMP. PutMetricData is CloudWatch API, not Prometheus-compatible
- **D**: AMP does not directly access pod endpoints. VPC PrivateLink is used for secure remote write ingestion, not scraping

**Key Concepts:**
- AMP = Prometheus-compatible storage + query service (no scraping)
- Remote write is the ingestion mechanism
- ADOT Collector is the AWS-recommended ingestion agent
- Authentication uses AWS SigV4 for remote write endpoint
</details>

---

### Question 2
A CloudOps engineer needs to query metrics from an AMP workspace to create custom dashboards. They write a PromQL query: `rate(http_requests_total[5m])` but are unsure where to execute it. What are the valid options for querying AMP? (Choose TWO)

A) Use AWS CLI command `aws amp query-workspace` with PromQL query parameter
B) Configure Amazon Managed Grafana data source pointing to the AMP workspace query endpoint
C) Use the AMP query API endpoint directly with HTTP POST requests including AWS SigV4 authentication
D) Use AWS CloudWatch Insights with PromQL query language
E) Configure CloudWatch Dashboard with AMP as a data source using PromQL queries

<details>
<summary>Answer</summary>

**Correct Answers: B and C**

**Explanation:**

**B is correct:**
- Amazon Managed Grafana (AMG) natively integrates with AMP
- Add AMP workspace as a Prometheus data source in Grafana
- Grafana translates dashboard queries to AMP API calls automatically
- AWS handles authentication between AMG and AMP
- This is the most common and user-friendly querying method

**C is correct:**
- AMP exposes Prometheus-compatible query API endpoints
- Endpoint format: `https://aps-workspaces.{region}.amazonaws.com/workspaces/{workspace-id}/api/v1/query`
- Requires AWS SigV4 authentication (IAM-based)
- Supports standard Prometheus query API: `/api/v1/query` (instant) and `/api/v1/query_range` (range queries)
- Can be used programmatically from applications, scripts, or tools like curl with aws-sigv4-proxy

**Example curl with SigV4:**
```bash
curl -X POST \
  --aws-sigv4 "aws:amz:us-east-1:aps" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d 'query=up' \
  https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-xxx/api/v1/query
```

**Why other options are incorrect:**
- **A**: There is no `aws amp query-workspace` command. The AWS CLI for AMP focuses on workspace management, not querying metrics
- **D**: CloudWatch Insights uses its own query language (not PromQL). CloudWatch and AMP are separate services
- **E**: CloudWatch Dashboards cannot use AMP as a data source. They display CloudWatch metrics only

**Key Concepts:**
- AMP query endpoint is Prometheus-compatible
- Amazon Managed Grafana is the primary visualization tool for AMP
- Authentication requires AWS SigV4 signing
- PromQL queries work exactly as in open-source Prometheus
</details>

---

### Question 3
A fintech company runs containerized microservices on EKS and needs to monitor them with AMP. Compliance requires that all metrics data remain within their VPC and never traverse the public internet. How should they configure AMP ingestion?

A) Deploy ADOT collector with VPC endpoints for AMP remote write, using AWS PrivateLink for secure private connectivity
B) Enable AMP VPC peering to directly connect the EKS VPC with the AMP service VPC
C) Configure NAT Gateway with security groups to restrict AMP traffic to private subnets only
D) Use AWS Direct Connect to route metrics from EKS to AMP through dedicated private connections

<details>
<summary>Answer</summary>

**Correct Answer: A**

**Explanation:**
- AWS PrivateLink allows private connectivity to AMP without using public internet
- VPC endpoints for AMP enable remote write traffic to stay within AWS network
- ADOT collector (or Prometheus server) sends metrics to VPC endpoint instead of public AMP endpoint

**Configuration Steps:**
1. Create VPC endpoint for AMP in your VPC
   - Service name: `com.amazonaws.{region}.aps-workspaces`
   - Type: Interface endpoint (PrivateLink)
2. Update ADOT collector remote write configuration to use VPC endpoint DNS
3. Ensure security groups allow HTTPS (443) traffic to VPC endpoint
4. Metrics flow: EKS pods → ADOT collector → VPC endpoint → AMP workspace

**Remote Write Config with VPC Endpoint:**
```yaml
remote_write:
  - url: https://vpce-xxx.aps-workspaces.us-east-1.vpce.amazonaws.com/workspaces/ws-yyy/api/v1/remote_write
    sigv4:
      region: us-east-1
```

**Why other options are incorrect:**
- **B**: AMP is a fully managed service without a VPC that can be peered. VPC peering connects two customer VPCs, not AWS service networks
- **C**: NAT Gateway routes traffic to the internet. Even with security groups, traffic would traverse public internet to reach AMP public endpoints
- **D**: Direct Connect is for on-premises to AWS connectivity. While it could work, PrivateLink is simpler and specifically designed for AWS service access

**Key Concepts:**
- PrivateLink provides private connectivity to AWS services
- VPC endpoints eliminate internet gateway/NAT requirements
- AMP supports interface VPC endpoints
- Data stays within AWS network backbone
- Cost: VPC endpoint charges apply (per hour + per GB processed)
</details>

---

### Question 4
A CloudOps team is evaluating when to use Amazon Managed Service for Prometheus versus CloudWatch for monitoring their containerized applications on EKS. Which scenarios favor AMP over CloudWatch? (Choose TWO)

A) Applications already expose Prometheus-format metrics at /metrics endpoints with custom business metrics
B) Need to monitor AWS service metrics like EC2 CPU, ELB request counts, and RDS connections
C) Require complex PromQL queries with functions like `rate()`, `histogram_quantile()`, and multi-dimensional aggregations
D) Primary use case is setting simple threshold-based alarms on standard EC2 and RDS metrics
E) Need to retain metrics for 15 months for compliance and historical analysis

<details>
<summary>Answer</summary>

**Correct Answers: A and C**

**Explanation:**

**A is correct (AMP advantage):**
- Prometheus is the de facto standard for Kubernetes and container monitoring
- Most cloud-native applications expose Prometheus-format metrics (/metrics endpoint)
- AMP natively ingests Prometheus metrics without transformation
- Custom application metrics (business KPIs, latency percentiles) are easily exposed in Prometheus format
- CloudWatch Container Insights can collect some metrics but requires agent translation

**C is correct (AMP advantage):**
- PromQL is a powerful query language designed for time-series data
- Functions like `rate()` (change over time), `histogram_quantile()` (percentile calculations), and multi-dimensional aggregations are built-in
- CloudWatch Insights query language is less specialized for metric analysis
- Complex queries like SLI/SLO calculations, RED metrics, and advanced aggregations are easier in PromQL
- Example: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))` for p95 latency

**Why other options are incorrect:**
- **B**: CloudWatch is purpose-built for AWS service metrics and integrates natively. This is a CloudWatch strength, not AMP
- **D**: CloudWatch excels at simple threshold alarms for AWS services. AMP requires Prometheus Alertmanager configuration, which is more complex for basic use cases
- **E**: AMP retention is only 150 days by default. CloudWatch can retain metrics indefinitely with extended retention settings. For 15 months, CloudWatch is better

**Key Decision Criteria:**
- **Use AMP:** Kubernetes/container workloads, Prometheus-format metrics, complex PromQL queries, existing Prometheus expertise
- **Use CloudWatch:** AWS service monitoring, simple alarms, long retention, AWS-native integrations
- **Use Both:** Hybrid approach - CloudWatch for infrastructure, AMP for application/container metrics

**Key Concepts:**
- AMP and CloudWatch serve different monitoring needs
- They can coexist in the same architecture
- Consider metric format, query complexity, and retention requirements
- Open-source Grafana can query both AMP and CloudWatch
</details>

---

### Question 5
A company's EKS cluster runs 200 microservices, each exposing custom Prometheus metrics. The AMP workspace shows unexpectedly high ingestion costs. Upon investigation, they discover many high-cardinality metrics (thousands of unique label combinations). What is the BEST approach to reduce costs while maintaining observability?

A) Reduce the scrape interval from 15 seconds to 60 seconds to decrease data points ingested
B) Implement metric relabeling in the ADOT collector to drop high-cardinality labels or metrics that aren't critical for monitoring
C) Switch from AMP to CloudWatch Container Insights to reduce costs
D) Enable AMP data compression and aggregation features to reduce storage size

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- High cardinality = many unique time series (combination of metric name + labels)
- Common culprits: request IDs, user IDs, timestamps as labels
- AMP charges based on samples ingested and queried
- Metric relabeling allows filtering/dropping metrics and labels before ingestion

**ADOT Collector Relabeling Configuration:**
```yaml
processors:
  metricstransform:
    transforms:
      - include: http_requests_total
        match_type: strict
        action: update
        operations:
          - action: delete_label_value
            label: user_id  # Drop high-cardinality label
          - action: delete_label_value
            label: request_id

  filter:
    metrics:
      exclude:
        match_type: regexp
        metric_names:
          - 'debug_.*'  # Drop debug metrics
```

**Best Practices:**
- Keep cardinality under 10 labels per metric
- Use labels for aggregation dimensions (service, environment, region)
- Avoid labels with unbounded values (IDs, timestamps)
- Drop unnecessary metrics entirely
- Sample high-frequency metrics if exact precision isn't needed

**Why other options are incorrect:**
- **A**: Reducing scrape interval helps but doesn't address the root cause (high cardinality). You still ingest many unique time series per scrape
- **C**: CloudWatch Container Insights has similar cost challenges with high cardinality. The problem is metric design, not the platform
- **D**: AMP doesn't have configurable "data compression" features. Prometheus uses efficient storage but doesn't aggregate raw metrics (that would lose fidelity)

**Key Concepts:**
- AMP pricing: samples ingested + samples queried + storage
- High cardinality is the #1 cost driver in Prometheus-based monitoring
- Metric relabeling is performed at collection time (before remote write)
- Use recording rules for expensive aggregate queries
</details>

---

### Question 6
A DevOps team wants to set up alerting for their AMP-monitored EKS cluster. They need to receive Slack notifications when pod memory usage exceeds 80% for more than 5 minutes. What components are required to implement this alerting workflow?

A) Configure CloudWatch Alarms to query AMP metrics and send notifications to SNS, which triggers Lambda to post to Slack
B) Deploy Prometheus Alertmanager (self-managed or using AWS Partner solutions), configure alert rules in AMP, and integrate Alertmanager with Slack webhook
C) Use Amazon Managed Grafana alerts configured against AMP data source, with SNS notification channel to Lambda for Slack integration
D) Enable AMP native alerting feature with direct Slack integration using OAuth tokens

<details>
<summary>Answer</summary>

**Correct Answer: C**

**Explanation:**
- AMP does NOT include managed Alertmanager (as of current features)
- Amazon Managed Grafana (AMG) has built-in alerting capabilities
- AMG can alert on queries against AMP data sources
- AMG supports notification channels including SNS, which can trigger Lambda for Slack

**Implementation Steps:**
1. Create AMP workspace with metrics ingestion
2. Create Amazon Managed Grafana workspace
3. Add AMP as data source in Grafana
4. Create alert rule in Grafana:
   - Query: `avg(container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.8`
   - Evaluate every 1m for 5m
5. Configure SNS notification channel in Grafana
6. Create Lambda function to receive SNS and post to Slack webhook
7. Subscribe Lambda to SNS topic

**Why other options are incorrect:**
- **B**: While technically possible (self-managed Alertmanager), this is NOT the AWS-managed solution and requires operational overhead. You'd need to deploy/manage Alertmanager in EKS. AMP doesn't natively support alert rules configuration
- **A**: CloudWatch Alarms cannot directly query AMP metrics. CloudWatch and AMP are separate data stores
- **D**: AMP does not have native alerting features. Alert evaluation must be done externally (Grafana, self-managed Alertmanager, or custom solutions)

**Alternative Approaches:**
- Self-managed Prometheus Alertmanager deployed in EKS (queries AMP via API)
- AWS Partner solutions like Grafana Enterprise (includes alerting)
- Custom Lambda functions querying AMP API periodically (not recommended for real-time alerts)

**Key Concepts:**
- AMP = storage + query only (no alerting, no visualization)
- Amazon Managed Grafana provides visualization + alerting
- For production alerting, AMG is the AWS-native solution
- Self-managed Alertmanager is an option but adds operational burden
</details>

---

### Question 7
A company has multiple EKS clusters across dev, staging, and production environments, each sending metrics to separate AMP workspaces. The CloudOps team needs to query metrics across all environments simultaneously to compare performance. What is the correct approach?

A) Enable AMP workspace federation to create a unified query endpoint across multiple workspaces
B) Configure Amazon Managed Grafana with multiple AMP data sources, and create dashboards using Grafana's mixed data source feature
C) Use AWS Glue to ETL metrics from multiple AMP workspaces into a single consolidated workspace
D) Configure cross-workspace remote read in AMP to query multiple workspaces with a single PromQL query

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- AMP workspaces are isolated and don't support native federation or cross-workspace queries
- Amazon Managed Grafana can connect to multiple data sources simultaneously
- Grafana dashboards can query multiple AMP workspaces in a single view
- Use Grafana variables and templating to switch between environments or overlay data

**Grafana Configuration:**
1. Add each AMP workspace as a separate data source:
   - Data Source 1: "AMP-Dev" (dev workspace)
   - Data Source 2: "AMP-Staging" (staging workspace)
   - Data Source 3: "AMP-Production" (prod workspace)
2. Create dashboard panels that query specific or multiple data sources
3. Use Grafana's mixed data source feature to overlay metrics from different sources
4. Use dashboard variables for dynamic environment selection

**Example Panel:**
```
Query A (Data Source: AMP-Dev): rate(http_requests_total[5m])
Query B (Data Source: AMP-Prod): rate(http_requests_total[5m])
Display both on same graph with different colors/labels
```

**Why other options are incorrect:**
- **A**: AMP does not support workspace federation. Each workspace is independent
- **C**: AWS Glue is for data ETL, not real-time metrics. Consolidating workspaces would lose environment isolation and add complexity
- **D**: Prometheus remote read is for federating external Prometheus servers. AMP doesn't expose cross-workspace remote read capabilities

**Key Concepts:**
- AMP workspaces are isolated tenants
- Isolation provides security, cost allocation, and environment separation
- Amazon Managed Grafana is the integration layer for multi-workspace visibility
- Consider workspace design: per-environment, per-team, per-cluster, etc.
</details>

---

### Question 8
An application team wants to monitor their Go application running on EKS using AMP. The application needs to expose custom business metrics like "orders_processed_total" and "payment_failure_count." What is required to make these metrics available in AMP?

A) Install CloudWatch agent in the application container to collect metrics and forward to AMP
B) Instrument the application with Prometheus client library to expose metrics at /metrics endpoint, then configure ADOT collector to scrape and remote-write to AMP
C) Send metrics directly to AMP API using AWS SDK PutMetricData calls
D) Configure EKS service discovery annotations to automatically export application metrics to AMP

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- Applications expose Prometheus metrics by instrumenting code with Prometheus client libraries
- Prometheus client libraries available for: Go, Java, Python, Node.js, Ruby, .NET, etc.
- Metrics are exposed via HTTP endpoint (typically `/metrics`) in Prometheus text format
- ADOT collector or Prometheus server scrapes the endpoint and sends to AMP via remote write

**Implementation Steps:**

1. **Instrument Go Application:**
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    ordersProcessed = prometheus.NewCounter(prometheus.CounterOpts{
        Name: "orders_processed_total",
        Help: "Total number of orders processed",
    })
    paymentFailures = prometheus.NewCounter(prometheus.CounterOpts{
        Name: "payment_failure_count",
        Help: "Total number of payment failures",
    })
)

func init() {
    prometheus.MustRegister(ordersProcessed)
    prometheus.MustRegister(paymentFailures)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    // Application logic that increments counters
    ordersProcessed.Inc()
}
```

2. **Kubernetes Service with Annotations:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

3. **ADOT Collector scrapes and sends to AMP**

**Why other options are incorrect:**
- **A**: CloudWatch agent is for CloudWatch metrics, not AMP. It doesn't understand Prometheus format or remote write
- **C**: AMP doesn't have a PutMetricData API. That's CloudWatch. AMP uses Prometheus remote write protocol
- **D**: Service discovery annotations help ADOT/Prometheus discover scrape targets, but the application must still expose metrics at /metrics

**Key Concepts:**
- Prometheus client libraries for instrumentation
- Metrics exposed in Prometheus text format at HTTP endpoint
- Four metric types: Counter, Gauge, Histogram, Summary
- ADOT collector uses Prometheus receiver for scraping
</details>

---

### Question 9
A company uses AMP to monitor EKS workloads and needs to calculate a complex Service Level Indicator (SLI): the 95th percentile request latency over 30-day windows. Executing this query in Grafana times out due to the large time range. What is the BEST solution to optimize this query?

A) Increase the AMP workspace query timeout setting to allow longer-running queries
B) Create a Prometheus recording rule that pre-aggregates the 95th percentile calculation at regular intervals and store the result as a new metric
C) Export AMP metrics to S3 and use Amazon Athena for long-range queries
D) Reduce the time range to smaller windows and manually aggregate results in a spreadsheet

<details>
<summary>Answer</summary>

**Correct Answer: B**

**Explanation:**
- Recording rules pre-calculate expensive queries and store results as new time series
- Queries against recorded metrics are fast because aggregation is already done
- Recording rules run at regular intervals (e.g., every 1 minute) in Prometheus
- For AMP, you deploy a Prometheus server (or use Prometheus in EKS) with recording rules that queries AMP and writes results back via remote write

**Recording Rule Configuration:**
```yaml
groups:
  - name: sli_rules
    interval: 1m
    rules:
      - record: http_request_duration_p95
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**Architecture:**
1. Deploy Prometheus with recording rules in EKS
2. Prometheus queries AMP workspace (remote read or query API)
3. Evaluates recording rules periodically
4. Writes resulting metrics back to AMP (remote write)
5. Grafana queries the pre-aggregated `http_request_duration_p95` metric

**Alternative (if AMP adds native recording rules in future):**
- Configure recording rules directly in AMP workspace (not currently available as managed feature)

**Why other options are incorrect:**
- **A**: AMP doesn't expose a configurable query timeout setting. Queries have service-side limits that can't be increased
- **C**: Exporting to S3 and using Athena is for long-term cold storage analysis, not operational monitoring. High latency and complexity
- **D**: Manual aggregation is not scalable, error-prone, and defeats the purpose of automated monitoring

**Key Concepts:**
- Recording rules optimize expensive queries by pre-aggregating
- Useful for SLIs, dashboards that must load quickly, and long-range queries
- Trade-off: increased ingestion cost (new metrics) for faster query performance
- Recording rules require a Prometheus instance to evaluate (not built into AMP as managed feature)
</details>

---

### Question 10
A financial services company must meet compliance requirements that mandate all monitoring data be encrypted at rest and in transit. They plan to use AMP for EKS monitoring. What encryption capabilities does AMP provide, and what additional configuration is needed?

A) AMP automatically encrypts data at rest using AWS-managed keys and in transit using TLS; no additional configuration required
B) Configure AMP workspace with customer-managed KMS key for at-rest encryption, and enable TLS 1.3 for remote write and query endpoints
C) Enable AMP encryption module and configure IPsec tunnels from EKS to AMP for end-to-end encryption
D) Deploy AWS Certificate Manager (ACM) certificates on ADOT collectors and configure mutual TLS authentication with AMP endpoints

<details>
<summary>Answer</summary>

**Correct Answer: A**

**Explanation:**
- AMP automatically encrypts all data at rest using AWS-managed encryption keys
- Cannot currently configure customer-managed KMS keys for AMP workspaces (as of latest features)
- All data in transit is encrypted using TLS 1.2+ automatically:
  - Remote write from collectors to AMP: HTTPS
  - Queries from Grafana/clients to AMP: HTTPS
  - Internal AWS service communication: encrypted
- AWS SigV4 authentication provides additional security for API calls

**Compliance Considerations:**
- At-rest encryption: Automatic, AWS-managed keys (meets most compliance standards)
- In-transit encryption: TLS 1.2+ (meets PCI-DSS, HIPAA, etc.)
- Access control: IAM-based authentication and authorization
- Audit logging: AWS CloudTrail logs AMP API calls
- Network isolation: VPC endpoints (PrivateLink) for private connectivity

**If customer-managed keys are required:**
- AMP does NOT currently support CMK for at-rest encryption
- For strict compliance requiring CMK: consider self-managed Prometheus with encrypted EBS volumes using CMK, or wait for AMP feature update

**Why other options are incorrect:**
- **B**: AMP doesn't support customer-managed KMS keys for workspace encryption. TLS version is managed by AWS, not user-configurable to specific versions
- **C**: AMP doesn't have an "encryption module" to enable. IPsec is not used; TLS handles in-transit encryption
- **D**: ADOT collectors use AWS SigV4 for authentication (not ACM certificates). Mutual TLS is not required or configurable for AMP

**Key Concepts:**
- AMP encryption is automatic and transparent
- AWS-managed keys used for at-rest encryption
- TLS for all in-transit communication
- VPC endpoints add network-level isolation
- No performance impact from encryption (handled transparently)
</details>
