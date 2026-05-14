# Story 7 — PostgreSQL Connection Count Alert Integration

![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-0089D6?style=for-the-badge&logo=microsoftazure&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)
![Status](https://img.shields.io/badge/Status-In_Progress-yellow?style=for-the-badge)

## Ticket Reference
| Field | Value |
|---|---|
| **Story** | Story 7 |
| **Title** | Update PostgreSQL Flexible Server provisioning script to create connection count alerts |
| **Sprint** | EDAV PostgreSQL Alert Automation |
| **Assignee** | Austin Jones |
| **Reviewer** | Krishna |

---

## Purpose

Integrate connection count alerts into the PostgreSQL Flexible Server provisioning workflow. This covers two scenarios:

1. **Active Connections** — alert when connection count exceeds the pool limit threshold (`>343`)
2. 2. **Failed Connections** — alert when connection failures exceed tolerance threshold (`>5`)
  
   3. Both alerts use the `global_metric_alert` module (Story 2). If the `connections_failed` metric does not provide the needed granularity, the failed connections alert can be migrated to the `global_query_alert` module (Story 3) using a KQL-based approach.
  
   4. ---
  
   5. ## Alert Definitions
  
   6. | Alert Name | Metric | Threshold | Severity | Aggregation |
   7. |---|---|---|---|---|
   8. | `postgres-active-connections-{env}` | `active_connections` | > 343 | 2 (Warning) | Average |
   9. | `postgres-failed-connections-{env}` | `connections_failed` | > 5 | 1 (Error) | Total |
  
   10. > **Note on `connections_failed`:** This metric uses `Total` aggregation (sum over window) because each failed connection is a discrete event. Active connections uses `Average` because it represents a continuous pool state.
       >
       > ---
       >
       > ## Terraform Code
       >
       > ```hcl
       > # Active Connections Alert (> 343)
       > module "postgres_active_connections" {
       >   source = "../../modules/global_metric_alert"
       >
       >   alert_name          = "postgres-active-connections-${var.environment}"
       >   resource_group_name = var.resource_group_name
       >   description         = "PostgreSQL active connections exceeded 343 — approaching connection pool limit"
       >
       >   scopes = [var.postgres_server_resource_id]
       >
       >   severity         = 2
       >   frequency        = "PT1M"
       >   window_size      = "PT5M"
       >   metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
       >   metric_name      = "active_connections"
       >   aggregation      = "Average"
       >   operator         = "GreaterThan"
       >   threshold        = 343
       >   action_group_id  = var.action_group_id
       >
       >   tags = {
       >     Environment = var.environment
       >     Story       = "story-7-connection-alert-integration"
       >     ManagedBy   = "Terraform"
       >     Component   = "PostgreSQL"
       >     AlertType   = "Connections-Active"
       >   }
       > }
       >
       > # Failed Connections Alert (> 5)
       > module "postgres_failed_connections" {
       >   source = "../../modules/global_metric_alert"
       >
       >   alert_name          = "postgres-failed-connections-${var.environment}"
       >   resource_group_name = var.resource_group_name
       >   description         = "PostgreSQL failed connections exceeded 5 in window — check credentials or network"
       >
       >   scopes = [var.postgres_server_resource_id]
       >
       >   severity         = 1
       >   frequency        = "PT1M"
         window_size      = "PT5M"
         metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
         metric_name      = "connections_failed"
         aggregation      = "Total"
         operator         = "GreaterThan"
         threshold        = 5
         action_group_id  = var.action_group_id

         tags = {
           Environment = var.environment
           Story       = "story-7-connection-alert-integration"
           ManagedBy   = "Terraform"
           Component   = "PostgreSQL"
           AlertType   = "Connections-Failed"
         }
       }
       ```

       ---

       ## How to Run Locally

       ### Prerequisites
       - [Terraform >= 1.5](https://developer.hashicorp.com/terraform/downloads)
       - Azure CLI authenticated (`az login`)
       - `global_metric_alert` module available (Story 2)
       - PostgreSQL Flexible Server provisioned in DEV

       ### Steps

       ```bash
       # 1. Clone the repo
       git clone https://github.com/ausjones84/story-7-postgres-connection-alert-integration.git
       cd story-7-postgres-connection-alert-integration

       # 2. Navigate to DEV environment
       cd terraform-scripts/dev/postgres-alerts

       # 3. Copy and edit tfvars
       cp postgres-servers.tfvars.example postgres-servers.tfvars

       # 4. Init, validate, plan, apply
       terraform init -backend=false
       terraform validate
       terraform plan -var-file="postgres-servers.tfvars"
       terraform apply -var-file="postgres-servers.tfvars"
       ```

       ### Expected Output
       ```
       azurerm_monitor_metric_alert.postgres_active_connections: Creation complete
       azurerm_monitor_metric_alert.postgres_failed_connections: Creation complete

       Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
       ```

       ---

       ## Acceptance Criteria Checklist

       - [ ] Active connections alert added — metric: `active_connections`, threshold: `343`, aggregation: `Average`
       - [ ] - [ ] Failed connections alert added — metric: `connections_failed`, threshold: `5`, aggregation: `Total`
       - [ ] - [ ] Active connections severity set to **2 (Warning)**
       - [ ] - [ ] Failed connections severity set to **1 (Error)**
       - [ ] - [ ] Action group configured per environment
       - [ ] - [ ] `terraform plan` shows 2 new alert resources
       - [ ] - [ ] Alerts validated in Azure Portal or Log Analytics
       - [ ] - [ ] Decision documented: metric-based vs KQL for failed connections
       - [ ] - [ ] Ticket updated
      
       - [ ] ---
      
       - [ ] ## Decision: Metric vs. Query for Failed Connections
      
       - [ ] | Approach | Pros | Cons |
       - [ ] |---|---|---|
       - [ ] | Metric alert (`connections_failed`) | Simple, no Log Analytics needed | Less context in alert |
       - [ ] | Query alert (KQL via Story 3) | Full log context, richer queries | Requires Log Analytics + diagnostic settings |
      
       - [ ] **Recommendation:** Start with metric alert. If additional context or alerting granularity is needed, migrate to KQL via Story 3's `global_query_alert` module.
      
       - [ ] ---
      
       - [ ] ## Ticket Update (Copy-Paste Ready)
      
       - [ ] ```
       - [ ] Integrated PostgreSQL connection count alerts using the global metric alert module.
       - [ ] Added active_connections alert at >343 (severity 2/Warning, Average aggregation) and
       - [ ] connections_failed alert at >5 (severity 1/Error, Total aggregation).
      
       - [ ] Active connections uses Average aggregation to track ongoing pool state. Failed connections
       - [ ] uses Total aggregation since each failure is a discrete event counted over the window.
      
       - [ ] Decision: Metric-based alerts used for both. If more context is needed for failed connections,
       - [ ] can migrate to KQL-based alert using the global_query_alert module (Story 3).
      
       - [ ] Validated with terraform plan — 2 new alert resources will be created.
       - [ ] ```
      
       - [ ] ---
      
       - [ ] ## Related Stories
      
       - [ ] | Story | Repo | Description |
       - [ ] |---|---|---|
       - [ ] | Story 2 | [story-2-global-metric-alert-module](https://github.com/ausjones84/story-2-global-metric-alert-module) | Global metric alert module (dependency) |
       - [ ] | Story 3 | [story-3-global-query-alert-module](https://github.com/ausjones84/story-3-global-query-alert-module) | Global query alert module (fallback for failed connections) |
       - [ ] | Story 4 | [story-4-postgres-cpu-alert-integration](https://github.com/ausjones84/story-4-postgres-cpu-alert-integration) | CPU alert integration |
       - [ ] | Story 5 | [story-5-postgres-memory-alert-integration](https://github.com/ausjones84/story-5-postgres-memory-alert-integration) | Memory alert integration |
       - [ ] | Story 6 | [story-6-postgres-disk-alert-integration](https://github.com/ausjones84/story-6-postgres-disk-alert-integration) | Disk alert integration |
       - [ ] | Story 7 | **This repo** | Connection alert integration |
       - [ ] | Story 8 | [story-8-postgres-restore-alert-rules](https://github.com/ausjones84/story-8-postgres-restore-alert-rules) | Restore alert rules |
