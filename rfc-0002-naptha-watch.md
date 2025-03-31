# RFC-0002: naptha-watch Monitoring Solution

- **RFC Number**: 0002
- **Title**: naptha-watch Monitoring Solution
- **Author**: Madhur Tandon and Mohamed Arshath
- **Created**: 2025-03-31
- **Status**: Draft

## Summary

This RFC proposes "naptha-watch," a centralized monitoring solution for Naptha applications running across multiple EC2 instances and cloud environments, including nodes not directly hosted by Naptha. The solution will be based on VictoriaMetrics and Grafana with a PUSH-based architecture to provide unified metrics collection, health checking, job monitoring, and alerting capabilities for both public and private nodes.

## Motivation

At present, Naptha applications running across multiple EC2 instances and other cloud environments (8xA100 instances) lack a unified monitoring system. This creates several challenges:

1. No centralized view of application health across distributed deployments
2. Inconsistent health check mechanisms for detecting application failures
3. Limited visibility into system metrics (CPU, memory, disk) across instances
4. No standardized alerting system for critical failures or performance degradation
5. Difficulty tracking scheduled job executions and their statuses

Currently, no formalized monitoring architecture exists. All monitoring and health checking is performed manually. This manual approach is time-consuming, error-prone, and does not scale with the growing number of instances. It results in delayed incident detection and hampers the team's ability to proactively address potential issues.

## Detailed Proposal

### Tech Stack

1. **VictoriaMetrics**: For metrics collection, storage, and querying
   - Chosen over Prometheus for better scalability, storage efficiency, and performance
   - 100% Prometheus-compatible APIs for easy adoption
   - Independent scaling of read path, write path, and storage components
   - More efficient compression and storage utilization

2. **Grafana**: For visualization and dashboards
   - Will be self-hosted on the central monitoring server
   - Connects to VictoriaMetrics as a data source

3. **Alertmanager**: For alert routing, grouping, and notification
   - Routes notifications to Discord via webhooks

4. **Node Exporter**: For system metrics collection on each EC2 instance
   - Collects CPU, memory, disk, and network metrics

### Solution Components

#### Central Monitoring Server

A dedicated EC2 instance will host the core components:

1. **VictoriaMetrics Server**:
   - Receives metrics via PUSH from all instances
   - Stores time-series data efficiently
   - Provides querying capabilities for Grafana and alerting

2. **Grafana**:
   - Connects to VictoriaMetrics as data source
   - Hosts dashboards for different monitoring aspects
   - Provides authentication and authorization for users

3. **Alertmanager**:
   - Receives alerts from VictoriaMetrics
   - Groups and deduplicates alerts
   - Routes notifications to Discord via webhooks
   - Implements silencing and inhibition rules

#### Target Instance Components

On each monitored EC2 instance:

1. **Node Exporter** with **VictoriaMetrics Agent** or **vmagent**:
   - Collects system metrics (CPU, memory, disk, network)
   - Actively pushes metrics to the central VictoriaMetrics server
   - Minimal resource footprint
   - Handles local buffering in case of network issues

### Metrics Hierarchy

The monitoring system will collect metrics at three distinct levels:

1. **System Metrics**: Basic infrastructure metrics
   - CPU utilization, load averages
   - Memory usage and swap activity
   - Disk utilization and I/O operations
   - Network traffic and errors

2. **Application Metrics**: Service-specific metrics (if relevant)
   - Custom application metrics as needed (eg: HTTP status codes and response times)

3. **Business Metrics**: (if relevant)
   - Eg: Active users/sessions etc.

### Implementation Plan

1. **Setup Repository**
   - Create `naptha-watch` repository
   - Add configuration templates and deployment scripts

2. **Deploy Central Server**
   - Provision monitoring EC2 instance
   - Deploy VictoriaMetrics, Grafana, Alertmanager
   - Configure security and retention policies

3. **Pilot Integration**
   - Set up one Naptha node with Node Exporter and vmagent
   - Verify metric collection and data flow

4. **Create Dashboards/Alerts**
   - Build system metric dashboards
   - Configure alerts with Discord integration
   - Develop alert runbooks

5. **Scale to Hosted Nodes**
   - Expand to all Naptha-hosted nodes
   - Automate deployment
   - Optimize configurations

6. **Support External Nodes**
   - Create installation package for non-Naptha nodes
   - Implement authentication for external nodes

## Rationale and Alternatives

### Why PUSH-based vs PULL-based Architecture?

We have decided to implement a PUSH-based architecture where nodes actively push their metrics to the central server, rather than having the central server pull metrics from each node. This decision was made for several reasons:

1. **Support for Non-Naptha-Hosted Nodes**: The primary rationale is to enable monitoring of Naptha nodes that are not hosted directly by us and may not have public IPs. A PUSH-based approach allows these private nodes to initiate connections to our central monitoring server, which would be difficult with a PULL-based approach.

2. **Firewall Considerations**: Many of our nodes may be behind firewalls or NAT, making it difficult for a central server to reach them directly for metrics scraping.

3. **Simplified Configuration**: No need to maintain a central configuration of all endpoints to scrape, which can become unwieldy as the number of instances grows.

4. **Better for Dynamic Environments**: Instances can self-register by simply pushing their metrics, without requiring service discovery mechanisms or configuration updates on the central server.

5. **Local Buffering**: Agents can buffer metrics locally during network outages or when the central server is temporarily unavailable, preventing data loss.

6. **Reduced Central Server Load**: Distributes the work of initiating connections across all nodes rather than concentrating it on the central server.

### Why VictoriaMetrics over Prometheus?

VictoriaMetrics was chosen over Prometheus for several key reasons:

1. **PUSH-based Architecture Support**: VictoriaMetrics has excellent support for push-based metrics collection through vmagent, which aligns with our architectural decision.

2. **Scalability**: VictoriaMetrics offers independent scaling of read path, write path, and storage components. This makes it more adaptable to growing monitoring needs.

3. **Resource Efficiency**: VictoriaMetrics requires significantly less storage space due to better compression algorithms, reducing infrastructure costs.

4. **Performance**: Faster query execution for high-cardinality metrics, which becomes increasingly important as the number of monitored instances grows.

5. **Compatibility**: 100% compatibility with Prometheus query language (PromQL) and APIs, allowing easy migration and use of existing Prometheus tooling.

6. **Future-proofing**: Many major companies have already transitioned from Prometheus to VictoriaMetrics, indicating this is likely the direction the industry is moving.

### Alternatives Considered

1. **Hosted Prometheus/Grafana**: Services like Grafana Cloud offer managed monitoring. While this would reduce operational overhead, it would increase ongoing costs and potentially limit customization options.

2. **Standard Prometheus**: Would work for our current scale but might require migration to a more scalable solution as we grow.

3. **ELK Stack**: More complex to set up and maintain for metrics-based monitoring, though excellent for log analysis which could be a future addition.

4. **AWS CloudWatch**: Native to AWS but has higher costs at scale and would limit our flexibility if we expand to other cloud providers.

## Impact and Trade-offs

### Benefits

1. Centralized visibility into all Naptha applications and infrastructure
2. Proactive detection of issues before they impact users
3. Consistent alerting mechanisms across all environments
4. Better understanding of system behavior and performance patterns
5. Reduced manual monitoring overhead for operations teams

### Trade-offs

1. **Increased Infrastructure Costs**: Additional EC2 instance for the central monitoring server
2. **Learning Curve**: Team members will need to become familiar with VictoriaMetrics and Grafana
3. **Maintenance Overhead**: The monitoring system itself will require maintenance and updates

## Unresolved Questions

1. Should we consider a high-availability setup for the monitoring system itself?
2. What are the appropriate retention periods for different types of metrics?
3. Should we integrate with PagerDuty or similar services for critical alerts in addition to Discord?
4. How should we handle authentication and authorization for accessing Grafana dashboards?
5. What specific application metrics should we prioritize implementing?
6. What push interval should we set for the agents on each node to balance monitoring freshness with network and processing load?
7. What local buffering capacity should we configure for agents to handle temporary central server unavailability?
8. Should we implement server-side duplicate detection to handle potential duplicate metric submissions?

## References

1. [VictoriaMetrics Case Studies](https://docs.victoriametrics.com/casestudies/)
2. [VictoriaMetrics Cluster Installation](https://docs.victoriametrics.com/cluster-victoriametrics/)
3. [VictoriaMetrics vmagent Documentation](https://docs.victoriametrics.com/vmagent.html)
4. [Prometheus Instrumentation Best Practices](https://prometheus.io/docs/practices/instrumentation/)
5. [Grafana Documentation](https://grafana.com/docs/)
6. [Node Exporter Documentation](https://prometheus.io/docs/guides/node-exporter/)
