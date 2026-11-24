# AWS & DevOps Real-World Hands-on Scenario Practice

> **Goal:** Build production-style AWS and DevOps troubleshooting,
> architecture, automation, security, and operations skills through
> hands-on labs.
>
> **Level:** Beginner → Intermediate → Advanced\
> **Format:** Scenario questions + implementation tasks + evidence to
> capture\
> **Total scenarios:** 130\
> **Suggested pace:** 1--2 scenarios per day

------------------------------------------------------------------------

## How to use this README

For every scenario:

1.  Explain the problem in your own words before touching AWS.
2.  Draw the architecture or request flow.
3.  Implement the solution in a **sandbox AWS account**.
4.  Test both the success path and failure path.
5.  Capture CLI output, CloudWatch metrics/logs, screenshots,
    Terraform/code, and a short README.
6.  Write down the root cause, fix, prevention, security impact, and
    cost impact.

### Definition of done

A scenario is complete only when you can demonstrate:

-   [ ] The problem and business impact.
-   [ ] The architecture / request flow.
-   [ ] The implementation (console, AWS CLI, Terraform, or code).
-   [ ] A positive test and a negative/failure test.
-   [ ] Monitoring, alerting, and rollback/recovery where relevant.
-   [ ] Least-privilege security and cost considerations.
-   [ ] A short explanation suitable for a production change review.

### Priority guide

  -----------------------------------------------------------------------
  Priority                Meaning                 Suggested target
  ----------------------- ----------------------- -----------------------
  **P0 -- Must do**       Common production       First
                          responsibilities and    
                          interview-critical      
                          foundations             

  **P1 -- High**          Frequently used         Second
                          organization-level      
                          operations and          
                          integrations            

  **P2 -- Advanced**      Resilience, scale,      Third
                          automation, security,   
                          and platform            
                          engineering             

  **P3 -- Specialist**    Complex architecture    Last
                          and Kubernetes/platform 
                          operations              
  -----------------------------------------------------------------------

### Lab safety rules

-   Use a separate sandbox account and never experiment in a production
    account.
-   Set an AWS Budget and billing alert before starting.
-   Prefer free-tier eligible / smallest practical resources; delete
    resources after each lab.
-   Do not expose SSH/RDP or database ports to `0.0.0.0/0`.
-   Never commit access keys, passwords, kubeconfigs, `.env` files, or
    Terraform state.
-   Use synthetic data only. Do not upload real customer or patient
    data.
-   Before deleting or changing resources, check dependencies and take
    backups where needed.

------------------------------------------------------------------------

# P0 -- Must-do production scenarios

## 1. IAM -- Identity and Access Management

1.  **New employee onboarding:** An organization hires a developer who
    needs read-only access to EC2, S3, and CloudWatch. How will you
    create access without giving AdministratorAccess?
2.  **Team-based access:** Developers, operations engineers, and
    auditors need different permissions. How will you design IAM
    groups/roles and policies for each team?
3.  **Cross-account deployment:** A CI/CD role in the tooling account
    must deploy to a workload account. How will you configure
    `AssumeRole`, trust policy, and least privilege?
4.  **EC2 application permissions:** An application on EC2 needs to read
    one S3 bucket. How will you use an instance profile instead of
    storing access keys on the server?
5.  **Access denied investigation:** A user can list an S3 bucket but
    cannot download an object. How will you identify whether the cause
    is an identity policy, bucket policy, KMS key policy, SCP, or
    explicit deny?
6.  **Credential exposure:** A developer accidentally commits an AWS
    access key to a public repository. What immediate containment,
    rotation, investigation, and prevention steps will you perform?
7.  **Temporary vendor access:** A vendor needs access for two days. How
    will you provide time-bound, auditable access without sharing a
    permanent user credential?
8.  **MFA enforcement:** How will you enforce MFA for human users while
    allowing workloads to use roles and temporary credentials?
9.  **Unused permissions review:** How will you identify stale users,
    unused access keys, and excessive permissions, then safely remove
    access?
10. **Break-glass access:** How will you design emergency administrator
    access with MFA, approval, alerting, and an audit trail?

## 2. EC2 -- Compute and operating systems

11. **Application server deployment:** Deploy a Node.js application on a
    private EC2 instance. How will a user reach it through a public
    entry point without exposing the instance directly?
12. **Instance unreachable:** An EC2 instance is running, but SSH is
    timing out. How will you troubleshoot route tables, security groups,
    NACLs, public IP, key pair, OS firewall, and system status checks?
13. **High CPU incident:** A production EC2 instance has sustained 95%
    CPU utilization. How will you identify the process, inspect logs,
    determine whether scaling or optimization is needed, and prevent
    recurrence?
14. **Disk full:** A Linux application server reports
    `No space left on device`. How will you find large files, inspect
    logs, clean safely, expand EBS, and prevent another incident?
15. **Boot failure:** An instance fails system status checks after a
    configuration change. What recovery options will you use, and how
    will you preserve the root volume for investigation?
16. **Patch management:** How will you patch a fleet of Linux EC2
    instances with controlled rollout, maintenance windows, and rollback
    planning?
17. **Instance metadata security:** How will you enforce IMDSv2 and
    verify that applications still retrieve role credentials correctly?
18. **Golden AMI:** How will you create a hardened, repeatable AMI for
    application servers and distribute it across environments?
19. **EBS performance:** An application has high disk latency. How will
    you inspect EBS volume type, IOPS, throughput, queue length, and
    filesystem utilization?
20. **EC2 cost reduction:** An organization has idle and oversized
    instances. How will you identify candidates and right-size them
    without disrupting production?

## 3. VPC -- Networking foundations

21. **Three-tier network:** Design a VPC for public load balancers,
    private application servers, and isolated database subnets across
    two Availability Zones. Which routes and security controls are
    required?
22. **Private instance internet access:** Private EC2 instances need
    outbound package downloads but must not accept inbound internet
    traffic. How will you design NAT access and routes?
23. **No internet connectivity:** An instance in a private subnet cannot
    reach the internet. How will you troubleshoot route tables, NAT
    gateway state, subnet association, NACLs, and DNS?
24. **Overlapping CIDRs:** Two business units need VPC connectivity but
    their CIDR ranges overlap. What options can you evaluate, and what
    are the limitations?
25. **VPC peering:** Two VPCs need private communication. How will you
    configure peering, routes, security groups, and DNS---and what does
    peering not support?
26. **Private AWS service access:** Private instances need S3 access
    without traversing a NAT gateway. How will you configure and test an
    S3 gateway endpoint?
27. **Interface endpoint use case:** An application in private subnets
    needs Systems Manager or ECR APIs. When would you use interface
    endpoints, and how will you control endpoint access?
28. **Network ACL vs security group:** Traffic is failing even though
    the security group allows it. How will you inspect stateless NACL
    rules and ephemeral return ports?
29. **Hybrid connectivity:** An on-premises data center needs private
    connectivity to AWS. Compare Site-to-Site VPN and Direct Connect for
    this requirement.
30. **VPC traffic investigation:** A service cannot connect to another
    service on TCP 5432. How will you use VPC Flow Logs and
    packet/network checks to isolate the failure?

## 4. S3 -- Object storage

31. **Secure document bucket:** Create a bucket for company documents.
    How will you block public access, enable encryption, versioning, and
    least-privilege access?
32. **Accidental public access:** A bucket containing internal files
    becomes public. What containment, access investigation, data
    exposure assessment, and preventive controls will you apply?
33. **Cross-account file exchange:** One AWS account must upload files
    to another account's bucket. How will you configure bucket policy,
    role access, encryption, and ownership controls?
34. **Lifecycle management:** Logs older than 90 days should move to a
    cheaper storage class and expire after one year. How will you
    implement and validate lifecycle rules?
35. **Accidental deletion:** A user deletes an important object. How
    will versioning and delete markers help recover it, and what
    additional protection is appropriate?
36. **Large file upload:** Users upload multi-GB files. How will you use
    multipart upload and pre-signed URLs without exposing AWS
    credentials?
37. **Static website delivery:** Host static frontend assets in S3 and
    serve them securely through CloudFront. How will you prevent direct
    public bucket access?
38. **Encryption failure:** An application receives `AccessDenied` when
    uploading to an SSE-KMS bucket. How will you investigate IAM and KMS
    key permissions?
39. **Audit trail:** Compliance asks who accessed or changed objects in
    a sensitive bucket. Which S3 and CloudTrail data events would you
    enable, and what are the cost implications?
40. **Replication and recovery:** A business requires copies of objects
    in another Region/account. How will you design replication,
    versioning, ownership, and recovery testing?

## 5. Load Balancing and Auto Scaling

41. **High availability web tier:** Deploy an application across two
    Availability Zones behind an Application Load Balancer. How will you
    configure target groups and health checks?
42. **Targets unhealthy:** The ALB returns 503 although EC2 instances
    are running. How will you inspect listener rules, target health,
    application port, health-check path, and security groups?
43. **HTTPS enablement:** An application currently serves HTTP. How will
    you configure an ACM certificate, HTTPS listener, and HTTP-to-HTTPS
    redirect?
44. **Path-based routing:** Route `/api/*` to a backend service and `/`
    to a frontend service. How will you configure listener rules and
    priorities?
45. **Autoscaling policy:** Traffic increases during business hours. How
    will you build an Auto Scaling Group using a launch template and
    target-tracking policy?
46. **Bad deployment at scale:** A new AMI causes newly launched
    instances to fail health checks. How will you stop rollout, roll
    back, and keep capacity available?
47. **Scale-in protection:** A long-running job is interrupted when an
    instance is terminated. How will you use lifecycle hooks, draining,
    and termination protection appropriately?
48. **Uneven traffic:** One target receives more traffic or requests
    take too long. How will you investigate stickiness, slow-start,
    target response time, and application behavior?
49. **Multi-AZ resilience:** One Availability Zone becomes impaired. How
    will the load balancer and Auto Scaling Group maintain service, and
    what capacity limits must be considered?
50. **Capacity planning:** Design scaling thresholds for a service with
    predictable daily peaks and sudden bursts. Which metrics and
    cooldown/warm-up settings will you choose?

## 6. RDS -- Relational databases

51. **Production database setup:** Create an RDS database for a
    three-tier application. How will you place it in private subnets and
    restrict access to only the application tier?
52. **Database connection failure:** EC2 cannot connect to RDS. How will
    you check DNS endpoint, port, security groups, routes, NACLs,
    credentials, and database status?
53. **Multi-AZ requirement:** A business requires database availability
    during an AZ failure. Explain and configure Multi-AZ, and test what
    the application experiences during failover.
54. **Read-heavy application:** Reports are slowing down the primary
    database. How will you evaluate read replicas and route read-only
    queries safely?
55. **Storage full:** RDS storage is nearly exhausted. How will you
    configure storage autoscaling, monitor free storage, and plan
    capacity?
56. **Backup and restore:** A developer runs a destructive query. How
    will you restore to a point in time and validate the recovered
    database before cutover?
57. **Slow queries:** Application response time increases because of
    database queries. How will you use Performance Insights/Database
    Insights, slow query logs, and query plans?
58. **Credential rotation:** Database credentials are shared in
    application configuration. How will you move to Secrets Manager and
    implement rotation with minimal downtime?
59. **Maintenance planning:** A database engine patch is pending. How
    will you plan the maintenance window, test compatibility, take
    backups, and communicate impact?
60. **Database cost optimization:** An RDS instance is oversized and
    backups consume storage. How will you optimize instance class,
    storage, retention, and deployment options safely?

## 7. CloudWatch -- Monitoring and alerting

61. **No production visibility:** A service has no useful monitoring.
    Define the minimum dashboard for CPU, memory, disk, latency, error
    rate, request count, and database health.
62. **Memory alerting:** EC2 CPU metrics exist, but memory usage is
    unknown. How will you install/configure the CloudWatch Agent and
    publish memory metrics?
63. **Log investigation:** Users report intermittent 500 errors. How
    will you centralize application logs and use CloudWatch Logs
    Insights to find the pattern?
64. **Actionable alarms:** Create alarms for high CPU, low disk space,
    elevated 5xx responses, and low database free storage. How will you
    avoid noisy alerts?
65. **Incident notification:** An alarm must notify the on-call team.
    How will you connect CloudWatch alarms to SNS and test delivery?
66. **Log retention:** Application logs are retained forever. How will
    you set retention policies and export/archive logs when required?
67. **Synthetic monitoring:** How will you monitor a critical login or
    health-check endpoint from outside the application infrastructure?
68. **Deployment correlation:** Errors increase after a release. How
    will you correlate deployment timestamps, application logs, metrics,
    and alarms?
69. **Distributed tracing:** A request crosses API Gateway, Lambda, and
    a database. How will you use AWS X-Ray or supported tracing to find
    latency?
70. **Operational dashboard:** Build a dashboard that gives an on-call
    engineer a quick view of service health and links to the relevant
    logs and alarms.

------------------------------------------------------------------------

# P1 -- High-priority organization scenarios

## 8. Lambda and event-driven compute

71. **Scheduled cleanup:** Automatically remove expired temporary
    records every night. How will you schedule Lambda with EventBridge
    and make the function safe to retry?
72. **S3 event processing:** When a file is uploaded to S3, validate it
    and store metadata in a database. How will you configure event
    notifications and handle duplicate events?
73. **Lambda timeout:** A function regularly times out while processing
    files. How will you identify whether the cause is code, memory,
    network, downstream latency, or timeout configuration?
74. **Private database access:** Lambda must connect to RDS in private
    subnets. What VPC configuration, security groups, connection limits,
    and networking costs should you consider?
75. **Secrets management:** A Lambda function uses hardcoded
    credentials. How will you retrieve secrets securely and handle
    rotation/caching?
76. **Retry storm:** A downstream service is unavailable and Lambda
    retries increase load. How will you use bounded retries,
    destinations or DLQs, and idempotency?
77. **Cold starts:** A latency-sensitive API has inconsistent response
    times. How will you measure cold starts and evaluate memory tuning
    or provisioned concurrency?
78. **Concurrency control:** A function overwhelms a downstream database
    during a traffic spike. How will reserved concurrency and queue
    buffering protect the dependency?
79. **Safe deployment:** Deploy a new Lambda version with low risk. How
    will you use versions, aliases, weighted traffic, alarms, and
    rollback?
80. **Cost investigation:** Lambda costs rise sharply. How will you
    inspect invocations, duration, memory, retries, and architecture
    choices?

## 9. SQS and messaging

81. **Order processing:** An e-commerce application must process orders
    asynchronously. Design producer, SQS queue, consumer, retry, and
    status tracking.
82. **Poison messages:** Some messages fail repeatedly. How will you
    configure a redrive policy and DLQ, then inspect and replay messages
    safely?
83. **Duplicate processing:** A consumer receives the same message more
    than once. How will you make processing idempotent?
84. **Visibility timeout:** Messages are processed twice because
    processing takes longer than expected. How will you tune visibility
    timeout and extend it when needed?
85. **Backlog growth:** Queue depth keeps increasing. How will you
    inspect message age, consumer throughput, concurrency, and
    downstream bottlenecks?
86. **FIFO requirement:** A payment workflow requires ordering per
    customer. When would you choose FIFO queues and message group IDs?
87. **Secure queue access:** Only one producer role and one consumer
    role should access a queue. How will you implement IAM and queue
    policies?
88. **Queue monitoring:** Define alarms for visible messages, age of
    oldest message, DLQ depth, and failed processing.

## 10. API Gateway

89. **Serverless API:** Build a REST API using API Gateway and Lambda.
    How will you define routes, methods, integration, stages, and
    deployment?
90. **Authentication:** A public API must authenticate users. Compare
    IAM auth, Cognito/JWT authorizers, and Lambda authorizers for the
    use case.
91. **Abuse protection:** An endpoint is receiving excessive requests.
    How will you configure throttling, quotas, and WAF where
    appropriate?
92. **CORS failure:** Browser clients receive CORS errors while API
    calls work in Postman. How will you troubleshoot preflight requests
    and response headers?
93. **API versioning:** Existing mobile clients must continue using an
    older API while a new version is released. How will you manage
    stages, paths, or versioned deployments?
94. **Request validation:** Invalid payloads reach backend code. How
    will you validate request schemas and return consistent 4xx
    responses?
95. **API observability:** How will you trace a failed request across
    API Gateway, Lambda, and downstream services using access logs,
    execution logs, metrics, and correlation IDs?

## 11. CloudFront and Route 53

96. **Global frontend delivery:** A website hosted in S3 is slow for
    overseas users. How will you use CloudFront, caching, compression,
    and an appropriate origin configuration?
97. **Private S3 origin:** Users must access content only through
    CloudFront. How will you use Origin Access Control and block direct
    S3 access?
98. **Cache invalidation:** A deployment publishes updated JavaScript
    but users still see old files. How will you handle cache-control
    headers, versioned assets, and invalidations?
99. **Custom domain HTTPS:** Configure `app.example.com` with Route 53
    and CloudFront. Which region must the CloudFront ACM certificate be
    in?
100. **DNS migration:** Move a domain to a new load balancer with
     minimal downtime. How will you plan TTL, record changes,
     validation, and rollback?
101. **Health-based routing:** A service runs in two Regions. How will
     you use Route 53 health checks and routing policies, and what are
     their limitations?
102. **Redirect requirement:** Redirect the apex domain to `www` (or the
     reverse). Which DNS and HTTP-layer components are needed, and why
     is DNS alone not an HTTP redirect?
103. **Intermittent DNS issue:** Some users resolve an old IP. How will
     you inspect TTL, resolver caching, authoritative records, and
     propagation?

## 12. ECR, ECS, and containers

104. **Private image registry:** Build a Docker image and push it to
     ECR. How will you authenticate, tag immutable releases, and scan
     images?
105. **Image pull failure:** An ECS task cannot pull its ECR image. How
     will you check execution role permissions, network endpoints/NAT,
     repository policy, and image tag?
106. **ECS service deployment:** Deploy a containerized API on ECS
     Fargate behind an ALB. How will you configure task definition,
     service, networking, health checks, and scaling?
107. **Task keeps stopping:** An ECS task exits shortly after starting.
     How will you inspect stopped reason, container exit code, logs,
     health checks, and resource limits?
108. **Secrets in containers:** A Docker image contains credentials in
     an environment file. How will you move secrets to Secrets Manager
     or Parameter Store and rotate them?
109. **Rolling release:** Deploy a new container image without taking
     the service offline. Explain desired count, deployment
     configuration, health checks, and rollback.
110. **ECS capacity and cost:** Compare Fargate and EC2 launch types for
     a workload with variable traffic, persistent capacity needs, and
     specialized host requirements.

------------------------------------------------------------------------

# P2 -- Advanced reliability, security, and delivery

## 13. CI/CD and source control

111. **Build and deploy pipeline:** Create a GitHub Actions pipeline
     that tests a Node.js application, builds a Docker image, pushes it
     to ECR, and deploys to ECS.
112. **No long-lived AWS keys:** A CI pipeline currently stores AWS
     access keys as secrets. How will you configure GitHub OIDC
     federation and an IAM role?
113. **Pipeline failure:** A deployment job fails after the image is
     pushed. How will you make the workflow resumable and prevent a
     failed deploy from corrupting the running service?
114. **Artifact promotion:** The same tested artifact must move from dev
     to staging to production. How will you avoid rebuilding different
     artifacts for each environment?
115. **Approval gate:** Production deployment requires a human approval.
     How will you implement environment protection, approvals, and
     auditability?
116. **Rollback strategy:** A release causes elevated errors. How will
     the pipeline roll back to the last known-good image or application
     version?
117. **Repository migration:** Your organization needs to move from a
     discontinued or unsuitable source-control service. How will you
     migrate repositories, branches, tags, permissions, webhooks, and CI
     integrations?
118. **Pipeline security:** How will you scan dependencies, source code,
     secrets, and container images, and decide which findings should
     block a release?

## 14. Kubernetes / Amazon EKS

119. **First EKS application:** Deploy a containerized application to
     EKS with Deployment, Service, ConfigMap, Secret, and Ingress.
     Explain each resource's role.
120. **CrashLoopBackOff:** A pod repeatedly restarts. How will you use
     `kubectl describe`, `kubectl logs --previous`, events, probes, and
     resource limits to find the root cause?
121. **Pending pods:** Pods remain Pending. How will you inspect
     scheduler events, node capacity, taints/tolerations, affinity,
     PVCs, and subnet/IP capacity?
122. **ImagePullBackOff:** A pod cannot pull an image from ECR. How will
     you verify image name/tag, node or pod IAM permissions, registry
     access, and network connectivity?
123. **Service unreachable:** Pods are Running, but the application is
     not reachable through a Kubernetes Service. How will you inspect
     labels/selectors, endpoints, ports, NetworkPolicy, and Ingress?
124. **Autoscaling:** Pods need to scale with CPU/request load and nodes
     need to scale when capacity is exhausted. Explain HPA and
     cluster/node autoscaling options.
125. **Zero-downtime deployment:** Design readiness/liveness/startup
     probes, rolling updates, PodDisruptionBudgets, and graceful
     shutdown for a critical service.
126. **Secrets security:** Developers store credentials in plain
     Kubernetes Secrets. How will you improve encryption, RBAC, external
     secret integration, and rotation?
127. **Node failure:** A worker node becomes NotReady. How will you
     identify the reason, protect workloads, drain safely, and restore
     capacity?
128. **EKS access control:** Developers should access only one
     namespace. How will you configure IAM-to-Kubernetes access and
     namespace-scoped RBAC?
129. **Cluster observability:** How will you monitor node, pod,
     container, and application health, and centralize logs and metrics?
130. **EKS upgrade:** Plan a Kubernetes version upgrade. How will you
     check API deprecations, add-on compatibility, node groups,
     disruption budgets, and rollback options?

------------------------------------------------------------------------

# P3 -- Specialist / architecture challenge scenarios

131. **Multi-account landing zone:** Design separate security, log
     archive, shared services, development, staging, and production
     accounts. How will you use Organizations, SCPs, centralized
     logging, and delegated administration?
132. **Disaster recovery:** A business requires recovery in another
     Region. Define RTO/RPO, backup strategy, replication, DNS failover,
     runbooks, and a recovery exercise.
133. **Centralized security monitoring:** Aggregate CloudTrail, Config,
     GuardDuty, Security Hub, and findings across multiple accounts. How
     will the security team investigate and respond?
134. **Infrastructure as Code:** Build a reusable Terraform module for
     VPC, subnets, route tables, security groups, and outputs. How will
     you manage remote state, locking, environments, and drift?
135. **Terraform drift:** Someone changes a production security group
     manually. How will you detect drift, assess impact, reconcile code
     and reality, and prevent recurrence?
136. **Private platform architecture:** Design a private EKS/ECS
     platform with VPC endpoints, controlled egress, private DNS,
     centralized logging, and least-privilege access.
137. **Resilience game day:** Simulate an AZ-level failure in a
     non-production environment. What signals prove the system
     recovered, and what gaps did you find?
138. **Cost allocation:** Finance wants cost by
     product/team/environment. How will you define tags, activate cost
     allocation tags, use Cost Explorer/CUR, and identify unallocated
     spend?
139. **Incident response:** A production service is down after a network
     change. Describe incident roles, triage, mitigation, communication,
     rollback, evidence preservation, and postmortem.
140. **Architecture review:** Design a highly available web application
     with Route 53, CloudFront, WAF, ALB, ECS/EKS, RDS, S3, SQS,
     CloudWatch, and CI/CD. Defend each service choice and identify
     failure modes.

------------------------------------------------------------------------

## Weekly practice roadmap

  Week   Focus                                                  Target scenarios
  ------ ------------------------------------------------------ ------------------
  1      IAM, EC2, Linux troubleshooting                        1--20
  2      VPC, S3, networking                                    21--40
  3      ALB, Auto Scaling, RDS                                 41--60
  4      CloudWatch, Lambda, SQS                                61--88
  5      API Gateway, CloudFront, Route 53                      89--103
  6      ECR, ECS, CI/CD                                        104--118
  7      Kubernetes / EKS                                       119--130
  8      Multi-account, Terraform, DR, security, architecture   131--140

## Scenario submission template

Copy this template into a separate file for each lab:

``` text
# Scenario: [ID - Title]

## 1. Business problem
What happened? Who is affected? What is the impact?

## 2. Requirements and constraints
Availability:
Security:
Performance:
Cost:
Recovery / rollback:

## 3. Architecture
Draw the architecture and explain the request/data flow.

## 4. Implementation
Commands, Terraform, YAML, application code, and AWS configuration.

## 5. Validation
Success test:
Failure test:
Evidence / screenshots / logs:

## 6. Troubleshooting
Symptoms:
Checks performed:
Root cause:
Fix:

## 7. Production readiness
Monitoring and alerts:
Security / least privilege:
Backup and recovery:
Cost considerations:
Rollback plan:

## 8. Explanation
Explain the solution in 2 minutes as if you are presenting it in a change review.
```

## Progress tracker

Track completion in your own copy of this README. Mark a scenario
complete only after you can explain and demonstrate it.

-   [ ] P0 -- IAM (1--10)
-   [ ] P0 -- EC2 (11--20)
-   [ ] P0 -- VPC (21--30)
-   [ ] P0 -- S3 (31--40)
-   [ ] P0 -- Load Balancing / Auto Scaling (41--50)
-   [ ] P0 -- RDS (51--60)
-   [ ] P0 -- CloudWatch (61--70)
-   [ ] P1 -- Lambda (71--80)
-   [ ] P1 -- SQS (81--88)
-   [ ] P1 -- API Gateway (89--95)
-   [ ] P1 -- CloudFront / Route 53 (96--103)
-   [ ] P1 -- ECR / ECS (104--110)
-   [ ] P2 -- CI/CD (111--118)
-   [ ] P2 -- Kubernetes / EKS (119--130)
-   [ ] P3 -- Advanced architecture (131--140)

------------------------------------------------------------------------

## Final challenge

Build one end-to-end production-style platform:

-   A frontend delivered through CloudFront with a private S3 origin.
-   Route 53 for DNS and ACM for TLS.
-   An ALB in public subnets.
-   An application tier on ECS or EKS in private subnets.
-   RDS in isolated private subnets.
-   SQS for asynchronous background jobs.
-   Lambda for a scheduled or event-driven task.
-   ECR for container images.
-   CI/CD using GitHub Actions with OIDC (no permanent AWS access keys).
-   CloudWatch dashboards, logs, alarms, and notifications.
-   Terraform for repeatable infrastructure.
-   A written backup, rollback, and disaster-recovery plan.

**Your goal is not just to make it work. Your goal is to explain why it
is secure, observable, resilient, recoverable, and cost-aware.**
