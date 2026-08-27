# AWS Well-Architected Framework：Migration Lens

這是 AWS Migration Lens 的完整 practice catalog，從本 repository 的 `refs/migration-lens.pdf` 擷取並保留所有 107 個 `MIG-*-BP-*` identifiers 與原始 practice titles。它可作為 migration program、wave readiness review 與 remediation tracking 的工作清單，而不是高階摘要。

每個 checkbox 對應一個 AWS practice。將它轉為可稽核的工作項目時，至少補上：適用的 phase（Assess、Mobilize 或 Migrate and modernize）、workload/wave、owner、status、evidence link、risk decision 與 due date。標記完成代表 evidence 已由負責人驗證；未適用時必須記錄 rationale，而非直接勾選。

## 使用方式

1. 在 **Assess** 先以 Operational Excellence、Security、Reliability、Performance、Cost 與 Sustainability 的 applicable controls 建立 baseline、scope、business case、dependencies 與 risk register。
2. 在 **Mobilize** 把 open controls 轉成 landing zone、identity、network、operations、test 與 migration-factory 的 deliverables，並以 pilot 驗證。
3. 在 **Migrate and modernize** 以每個 wave 的 evidence、cutover result、rollback/DR test、actual KPI 與 lessons learned 重新 review；不要把先前 wave 的結果假設為後續 workload 已合規。

## Operational excellence practices

- [ ] **MIG-OPS-BP-1.1** - Your migration plan must be informed by and reflect technology, processes and business
- [ ] **MIG-OPS-BP-2.1** - Define and measure key performance indicators (KPIs) which can be shared with all teams involved in the migration
- [ ] **MIG-OPS-BP-3.1** - Invest time and effort to ensure the required migration and operations skills are captured, skills gaps identified, and training plans are implemented and managed
- [ ] **MIG-OPS-BP-4.1** - Build a comprehensive resource model for your migration that reflects the demands of the migration as well as the regular activities
- [ ] **MIG-OPS-BP-5.1** - Build a Cloud Center of Excellence (CCoE) team within your organization as part of your migration planning
- [ ] **MIG-OPS-BP-6.1** - Define Cloud Operations Strategy: understand your current operating model, processes and tools, and explore how to implement them efficiently, securely and reliably in the cloud to create your cloud operations strategy
- [ ] **MIG-OPS-BP-6.2** - Align AWS operational requirements with your existing tools and identify any gaps
- [ ] **MIG-OPS-BP-6.3** - Regularly test your operations in the cloud
- [ ] **MIG-OPS-BP-7.1** - Calculate your potential migration velocity using both technical and non-technical perspectives (like network bandwidth, team availability, volume of changes, and change freezes)
- [ ] **MIG-OPS-BP-8.1** - Ensure you have a testing strategy in place
- [ ] **MIG-OPS-BP-9.1** - Determine if your current CI/CD pipeline works on AWS
- [ ] **MIG-OPS-BP-9.2** - Provision resources through infrastructure as code (IaC) templates

## Security practices

- [ ] **MIG-SEC-BP-1.1** - Understand the security credentials needed by the discovery tool
- [ ] **MIG-SEC-BP-1.2** - Understand how the discovery tool works
- [ ] **MIG-SEC-BP-1.3** - Understand the discovery tool's data security and apply appropriate controls
- [ ] **MIG-SEC-BP-2.1** - Perform a tools mapping exercise
- [ ] **MIG-SEC-BP-3.1** - Understand, establish, and implement compliance framework
- [ ] **MIG-SEC-BP-4.1** - Implement strong identity and least privilege principles
- [ ] **MIG-SEC-BP-5.1** - Implement AWS multi-account structure
- [ ] **MIG-SEC-BP-6.1** - Establish secure connectivity to AWS
- [ ] **MIG-SEC-BP-6.2** - Establish network security controls
- [ ] **MIG-SEC-BP-7.1** - Establish security controls for protecting data at rest
- [ ] **MIG-SEC-BP-8.1** - Establish application layer security controls
- [ ] **MIG-SEC-BP-8.2** - Optimize application security with AWS Application Migration Service
- [ ] **MIG-SEC-BP-9.1** - Establish a data backup and restore strategy
- [ ] **MIG-SEC-BP-9.2** - Establish a Disaster recovery plan
- [ ] **MIG-SEC-BP-10.1** - Validate and use AWS native monitoring tools.
- [ ] **MIG-SEC-BP-10.2** - Explore cloud native AWS partner monitoring tools
- [ ] **MIG-SEC-BP-11.1** - Perform third-party integration due diligence
- [ ] **MIG-SEC-BP-12.1** - Understand the data security and compliance
- [ ] **MIG-SEC-BP-13.1** - Understand AWS service capabilities for event detection and investigation
- [ ] **MIG-SEC-BP-14.1** - Understand AWS best practices for incident response
- [ ] **MIG-SEC-BP-15.1** - Protect your network resources
- [ ] **MIG-SEC-BP-16.1** - Manage authentication for applications and databases

## Reliability practices

- [ ] **MIG-REL-BP-1.1** - Define SLAs across all applications or environments (like production, development, or test) and confirm them with your business team
- [ ] **MIG-REL-BP-1.2** - Define and automate runbooks and communicate them to your teams
- [ ] **MIG-REL-BP-1.3** - Map AWS Global Infrastructure to your business SLAs before migrations starts
- [ ] **MIG-REL-BP-1.4** - Select tools to monitor SLAs and notify you in case thresholds are exceeded
- [ ] **MIG-REL-BP-2.1** - Keep your business impact analysis up-to-date
- [ ] **MIG-REL-BP-2.2** - Update the risk assessment for the type of disaster events covered by your BCP
- [ ] **MIG-REL-BP-2.3** - Define the recovery point objective (RPO) and recovery time objective (RTO) targets
- [ ] **MIG-REL-BP-2.4** - Select a disaster recovery strategy based on cloud best practices
- [ ] **MIG-REL-BP-3.1** - Estimate the required maintenance window
- [ ] **MIG-REL-BP-3.2** - Test the migration window and impact
- [ ] **MIG-REL-BP-3.3** - Plan for failure
- [ ] **MIG-REL-BP-4.1** - Be aware of service quotas and constraints for migration services
- [ ] **MIG-REL-BP-4.2** - Estimate the impact of new workloads on existing service quotas across accounts and Regions
- [ ] **MIG-REL-BP-4.3** - Be aware of unchangeable service quotas and how you determine which accounts or VPCs workloads use
- [ ] **MIG-REL-BP-5.1** - Provide sufficient bandwidth for normal and traffic from data replication
- [ ] **MIG-REL-BP-5.2** - Assure that links and equipment to on-premises are highly available
- [ ] **MIG-REL-BP-5.3** - Verify that your network design enables communication between on-premises and cloud networks
- [ ] **MIG-REL-BP-5.4** - Use an IP scheme that allows for sufficient growth within cloud workloads and burst auto-scaling
- [ ] **MIG-REL-BP-5.5** - Complete a reliable DNS design that enables resolutions to existing domains, plus new domains in AWS
- [ ] **MIG-REL-BP-5.6** - Test network performance prior to migration
- [ ] **MIG-REL-BP-5.7** - Test network component failure
- [ ] **MIG-REL-BP-6.1** - Identify and back up all data that needs to be backed up, or reproduce the data from sources
- [ ] **MIG-REL-BP-7.1** - Deploy the workload to multiple locations
- [ ] **MIG-REL-BP-8.1** - Before the cut-over, test HA and FT for the migrated workloads, and perform a DR dry-run after the migration

## Performance efficiency practices

- [ ] **MIG-PERF-BP-1.1** - Understand the performance characteristics of your current infrastructure to select the best performant optimized cloud infrastructure
- [ ] **MIG-PERF-BP-2.1** - Evaluate operating systems and versions that are running in your environment
- [ ] **MIG-PERF-BP-3.1** - Evaluate the different methods to migrate data and select the one best for you use case: online mode, oﬄine mode, or hybrid approach
- [ ] **MIG-PERF-BP-4.1** - Select the storage solution based on the characteristics of your workloads
- [ ] **MIG-PERF-BP-4.2** - Choose the optimal storage solutions for specialized workloads, such as SAP and VMware cloud on AWS
- [ ] **MIG-PERF-BP-4.3** - Evaluate the different storage tiers at prices to meet your migrated workload's performance
- [ ] **MIG-PERF-BP-5.1** - Establish a reliable network connectivity from on-premises to AWS to ensure performance
- [ ] **MIG-PERF-BP-5.2** - Assure that network performance is not impacted by external factors
- [ ] **MIG-PERF-BP-6.1** - Identify a migration strategy for your network components (DNS, IP addressing, and DHCP) migration
- [ ] **MIG-PERF-BP-7.1** - Identify the right CloudWatch metrics to capture or detect anomaly and identify performance blockers for shared services
- [ ] **MIG-PERF-BP-7.2** - Select the best performing cloud infrastructure that can scale for additional workloads in future without any performance impact
- [ ] **MIG-PERF-BP-7.3** - Reduce the blast radius for performance impact into a single account
- [ ] **MIG-PERF-BP-7.4** - Benchmark existing workloads for performance
- [ ] **MIG-PERF-BP-8.1** - Perform stress and user acceptance tests on migrated workloads before the actual cutover.
- [ ] **MIG-PERF-BP-8.2** - Review and implement the lessons learned from previous migration waves
- [ ] **MIG-PERF-BP-8.3** - Perform a Well-Architected Framework Review on each iteration of the migrated workload.
- [ ] **MIG-PERF-BP-9.1** - Generate alarm-based notifications for metric's threshold breach
- [ ] **MIG-PERF-BP-9.2** - Determine the need for a real-time or a near real-time monitoring solution
- [ ] **MIG-PERF-BP-9.3** - Implement CloudWatch or a Quicksight dashboard as a single pane view for visualizing all metrics
- [ ] **MIG-PERF-BP-9.4** - Set up automated testing for your application metrics
- [ ] **MIG-PERF-BP-9.5** - Re-evaluate your compute usage with AWS Trusted Advisor, AWS Compute Optimizer, or partner tools

## Cost optimization practices

- [ ] **MIG-COST-BP-1.1** - Thoroughly assess existing infrastructure usage and application dependencies prior to migration
- [ ] **MIG-COST-BP-1.2** - Leverage AWS programs and workshops designed to remove common blockers and accelerate migrations
- [ ] **MIG-COST-BP-2.1** - Leverage existing tools to automate your migration
- [ ] **MIG-COST-BP-2.2** - Minimize the number of applications and the amount of data that is migrated
- [ ] **MIG-COST-BP-2.3** - Right-size your replication servers to prevent bottlenecks without over-provisioning
- [ ] **MIG-COST-BP-3.1** - Plan and set up cost and usage governance of AWS resources with help of IAM policies
- [ ] **MIG-COST-BP-3.2** - Define a cost allocation strategy that meets your organizations specific financial management process
- [ ] **MIG-COST-BP-3.3** - Design a strategy to monitor, track and analyze your AWS cost and usage as you move resources to AWS
- [ ] **MIG-COST-BP-4.1** - Create a deliberate metrics strategy to help demystify cloud economics
- [ ] **MIG-COST-BP-4.2** - Monitor spend and limit unintended or unnecessary costs with budgeting and forecasting tools
- [ ] **MIG-COST-BP-4.3** - Use AWS Cost Anomaly Detection in Cost Explorer to quickly improve cost controls
- [ ] **MIG-COST-BP-4.4** - Use dashboards that provide pre-built visualizations to help you get a detailed view of your AWS usage and costs as you move resources to AWS
- [ ] **MIG-COST-BP-5.1** - Leverage the right purchase options and scalable architecture for your workloads
- [ ] **MIG-COST-BP-5.2** - Identify resources during migration that are likely candidates for cost optimizations later
- [ ] **MIG-COST-BP-6.1** - Use automation to re-evaluate your compute usage periodically
- [ ] **MIG-COST-BP-7.1** - Create a plan early to optimize after the initial migration

## Sustainability practices

- [ ] **MIG-SUS-BP-1.1** - Include sustainability considerations as part of your migration business case and preliminary assessments
- [ ] **MIG-SUS-BP-2.1** - Choose a Region for the workloads you plan to migrate based on your business requirements and your sustainability goals
- [ ] **MIG-SUS-BP-3.1** - Focus on efficiency across all aspects of infrastructure
- [ ] **MIG-SUS-BP-4.1** - Adopt metrics that can signal the sustainability of your application
- [ ] **MIG-SUS-BP-4.2** - Include sustainability metrics in the application portfolio analysis to drive migration and modernization initiatives
- [ ] **MIG-SUS-BP-5.1** - Implement efficient workload design by leveraging the underlying infrastructure.
- [ ] **MIG-SUS-BP-6.1** - Identify environments and workloads that can be consolidated or retired
- [ ] **MIG-SUS-BP-6.2** - Identify workloads that can use efficient software and architecture patterns to maintain consistent high utilization of deployed resources
- [ ] **MIG-SUS-BP-6.3** - Analyze your data access patterns and data lifecycle processes, and evaluate how you can become more efficient and sustainable in your data management
- [ ] **MIG-SUS-BP-6.4** - Understand and influence business requirements, and optimize areas of code to reach your sustainability goals
- [ ] **MIG-SUS-BP-7.1** - Implement data management practices
- [ ] **MIG-SUS-BP-8.1** - Adopt methods that can reduce interim resource consumption during the migration

## References

- AWS Well-Architected Framework, [Migration Lens](../../../refs/migration-lens.pdf) (local source PDF; 138 pages; consulted 2026-08-26)
- AWS Well-Architected Framework, [Migration Lens documentation](https://docs.aws.amazon.com/wellarchitected/latest/migration-lens/welcome.html)
