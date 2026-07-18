==========================
DEVOPS AZURE AKS INTERVIEW NOTES
PART 1 – NETWORKING, SECURITY & TERRAFORM
==========================

1. How can two VNets communicate without VNet Peering?

Answer:
Azure provides multiple ways to connect VNets without VNet Peering:

• VPN Gateway
  - IPSec/IKE tunnel between VNets.
  - Used when peering is not possible.

• Azure Virtual WAN
  - Microsoft-managed global networking service.
  - Connects multiple VNets, branch offices and remote users through Virtual Hubs.

• ExpressRoute
  - Private dedicated connection to Azure.
  - Used for hybrid enterprise connectivity.

• Private Link
  - Provides private access to Azure PaaS services without exposing them to the Internet.

• Application-level connectivity
  - Azure Front Door
  - Azure Application Gateway
  - Azure API Management

Use Case:
If applications only need to communicate through APIs, Front Door or API Management can be used instead of network-level connectivity.

------------------------------------------------------------

2. Azure Virtual WAN

Purpose:
Azure Virtual WAN simplifies enterprise networking by centrally managing connectivity between Azure, branches and remote users.

Components:
• Virtual WAN
• Virtual Hub
• VPN Gateway
• ExpressRoute Gateway
• Point-to-Site VPN
• Azure Firewall
• Route Tables
• Routing Intent
• Secure Hub

Benefits:
• Centralized routing
• Multi-region connectivity
• Branch office connectivity
• Remote user connectivity
• ExpressRoute integration
• Simplified network management

Interview Answer:
Azure Virtual WAN is a Microsoft-managed networking service used to connect multiple VNets, branch offices and remote users through Virtual Hubs. It simplifies routing, improves scalability and provides centralized connectivity for enterprise environments.

------------------------------------------------------------

3. User cannot access Azure resources after assigning the same role

Troubleshooting Steps:

1. Verify Azure Entra ID account.
2. Verify correct Azure Tenant.
3. Check Azure RBAC assignment.
4. Verify RBAC scope:
   • Subscription
   • Resource Group
   • Resource
5. Verify Azure DevOps permissions:
   • Repository
   • Pipelines
   • Variable Groups
   • Environments
   • Service Connections
6. Verify Azure AD Group membership.
7. Check MFA.
8. Check Conditional Access policies.
9. Wait for RBAC propagation if recently assigned.
10. Verify resource-specific permissions:
    • AKS
    • Key Vault
    • Storage
11. Review Activity Logs.
12. Review Sign-in Logs.

Interview Answer:
Whenever a user cannot access Azure resources, I first verify Azure Entra ID, RBAC assignment and its scope, Azure DevOps permissions, resource-specific access, Conditional Access policies and Activity Logs before troubleshooting further.

------------------------------------------------------------

4. AKS Security

Enterprise Security Layers:

• Azure Entra ID Integration
• Azure RBAC
• Kubernetes RBAC
• Private AKS
• Network Policies
• NSGs
• Azure Firewall
• WAF
• AGIC
• Azure Key Vault
• Managed Identity
• Secrets Store CSI Driver
• Prisma Cloud Image Scanning
• Microsoft Defender for Cloud
• Azure Policy
• Namespace Isolation
• Resource Quotas
• Requests & Limits
• Azure Monitor
• Container Insights
• Log Analytics

Interview Answer:
Our AKS clusters are secured using Azure Entra ID, Azure RBAC, Kubernetes RBAC, Private AKS, Azure Firewall, WAF, Key Vault, Managed Identity, Network Policies, Azure Policy, Defender for Cloud and Azure Monitor.

------------------------------------------------------------

5. Private AKS

Private AKS exposes the Kubernetes API Server through a Private Endpoint instead of a Public IP.

API Server Access:

VPN
      ↓
Private Endpoint
      ↓
AKS API Server

or

ExpressRoute
      ↓
Private Endpoint
      ↓
AKS API Server

Advantages:
• API Server not exposed to Internet
• Better security
• Reduced attack surface

Azure DevOps Deployment:

Microsoft Hosted Agent
        ↓
Cannot access Private AKS

Self Hosted Agent
        ↓
Inside Azure VNet
        ↓
Deploys successfully

Interview Answer:
For Private AKS we deploy using self-hosted agents because Microsoft-hosted agents cannot access the private Kubernetes API Server.

------------------------------------------------------------

6. Network Security Group (NSG)

Purpose:
Controls traffic at Subnet and NIC level.

Filters:
• Source IP
• Destination IP
• Port
• Protocol

Example:

Allow:
Application Gateway Subnet
        ↓
AKS Subnet

Block:
All other inbound traffic.

NSG is Stateful.

------------------------------------------------------------

7. Application Security Group (ASG)

Purpose:
Logical grouping of Virtual Machines or NICs.

Instead of writing rules for multiple IPs:

10.0.1.10
10.0.1.11
10.0.1.12

Create:

Frontend-ASG

↓

Backend-ASG

Then create one NSG rule:

Allow Frontend-ASG → Backend-ASG

Advantages:
• Simplifies NSG rules
• Easier maintenance

Note:
ASGs are generally used for Azure VM/NIC traffic, not Kubernetes pod-to-pod communication.

------------------------------------------------------------

8. Kubernetes Network Policies

Purpose:
Controls Pod-to-Pod communication inside Kubernetes.

Example:

Frontend Pod
      ↓
Backend Pod

Allowed

Backend Pod
      ↓
Database Pod

Allowed

Any other Pod
      ↓
Backend Pod

Denied

Default Deny Policy is commonly implemented.

Interview Answer:
Network Policies provide micro-segmentation by restricting pod communication based on labels instead of IP addresses.

------------------------------------------------------------

9. Web Application Firewall (WAF)

Layer:
OSI Layer 7

Protects against:
• SQL Injection
• Cross Site Scripting (XSS)
• Command Injection
• Remote Code Execution
• OWASP Top 10 attacks

Modes:
• Detection
• Prevention

Usually integrated with Azure Application Gateway.

------------------------------------------------------------

10. Azure Firewall

Stateful managed firewall.

Supports:
• Network Rules
• Application Rules
• DNAT Rules

Application Rule Example:

Allow:
github.com
mcr.microsoft.com
management.azure.com

Block:
All other Internet traffic.

Benefits:
• Centralized outbound inspection
• Threat intelligence
• Logging
• High availability

------------------------------------------------------------

11. Stateful vs Stateless

Stateful:
• Azure Firewall
• NSG

Automatically allows return traffic.

Stateless:
Return traffic requires separate rules.

------------------------------------------------------------

12. FQDN

Fully Qualified Domain Name.

Examples:
• github.com
• login.microsoftonline.com
• management.azure.com
• mcr.microsoft.com

Azure Firewall Application Rules commonly use FQDNs instead of IP addresses.

------------------------------------------------------------

13. User Defined Route (UDR) & Route Tables

Azure provides default routing.

In enterprise environments:

AKS Subnet

↓

Route Table

↓

0.0.0.0/0

↓

Azure Firewall

↓

Internet

Benefits:
• Centralized Security
• Traffic Inspection
• Logging
• Compliance

------------------------------------------------------------

14. Terraform Resources Discussed

Resources:
• azurerm_network_security_group
• azurerm_subnet_network_security_group_association
• azurerm_firewall
• azurerm_firewall_policy
• Firewall Rule Collections
• azurerm_route_table
• azurerm_route
• azurerm_subnet_route_table_association

Recommended Folder Structure:

terraform/
├── environments/
├── modules/
│   ├── aks/
│   ├── firewall/
│   ├── nsg/
│   ├── route-table/
│   ├── appgw/
│   └── monitoring/

Deployment Order:

Resource Group
↓
VNet
↓
Subnets
↓
NSGs
↓
Azure Firewall
↓
Route Tables
↓
AKS
↓
Application Gateway
↓
Applications

------------------------------------------------------------

FINAL INTERVIEW SUMMARY

"Our enterprise AKS platform is secured using Azure Entra ID, Azure RBAC, Kubernetes RBAC, Private AKS, NSGs, Azure Firewall, WAF, AGIC, Azure Key Vault, Managed Identity, Network Policies, Defender for Cloud, Azure Policy and Azure Monitor. We use Route Tables and UDRs to route all outbound traffic through Azure Firewall for centralized inspection and security. Infrastructure is provisioned using Terraform with reusable modules following enterprise folder structures."
