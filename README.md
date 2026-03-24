# Secure AWS Organization Architecture

Welcome to the Secure AWS Organization architecture and governance documentation. This directory contains the blueprints for deploying a highly secure, airgapped-ready, and audit-compliant AWS Organization.

To fully understand the architecture, execution strategy, and security constraints, please review the documents in the following order:

### 1. The Big Picture (Start Here)
* **[`SecureOrg_End_to_End_Execution_Flow.md`](./SecureOrg_End_to_End_Execution_Flow.md)**
  Provides a chronological, high-level map of the entire build (Phases 0 through 4). Read this first for context.

### 2. Architecture & Design
* **[`AWS_SecureOrg_LandingZone.md`](./AWS_SecureOrg_LandingZone.md)**
  Explains *what* is being built: the core account structure, the "Control First, Network Later" philosophy, and the organizational tree (OUs).

### 3. Implementation Strategy
* **[`AWS_Organizations_Setup_with_CFN.md`](./AWS_Organizations_Setup_with_CFN.md)**
  Explains *how* the landing zone will be deployed, clearly defining the boundary between manual trust bootstrapping and CloudFormation automation.

### 4. Technical Details (Deployments & Networking)
* **[`Cloudformation_Stacksets_Networking_requirements.md`](./Cloudformation_Stacksets_Networking_requirements.md)**
  Details the engineering nuances of the implementation, including how StackSets operate without manual IAM roles and how baseline services are deployed in a fully airgapped manner.

### 5. Security Governance & Operations
* **[`root_strategy_across_org.md`](./root_strategy_across_org.md)**
  The operational rulebook. It introduces the advanced "Two-Tier Root Model" (centralized root removal for member accounts) and the strict "Break-Glass" protocol for the Management Account.
* **[`scp-deny-root-actions.json`](./scp-deny-root-actions.json)**
  A supporting code snippet showing the Service Control Policy (SCP) enforcement for root users in member accounts.

---
*Note: Infrastructure as Code (IaC) templates correspond directly to the phases outlined in the execution flow.*