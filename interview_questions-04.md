
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
==========================
DEVOPS AZURE AKS INTERVIEW NOTES
PART 2 – CERTIFICATES, KEY VAULT, MANAGED IDENTITY & CSI DRIVER
==========================

1. TLS vs SSL

SSL (Secure Sockets Layer)
• Older protocol.
• Deprecated due to security vulnerabilities.

TLS (Transport Layer Security)
• Successor to SSL.
• Used for HTTPS communication.
• Provides Encryption, Integrity and Authentication.

Interview Answer:
Today we use TLS, not SSL. Although people often say "SSL certificate," production environments use TLS certificates.

------------------------------------------------------------

2. What is a Digital Certificate?

A digital certificate is an electronic identity issued by a trusted Certificate Authority (CA). It proves the identity of a server or client and enables encrypted communication.

A certificate contains:
• Subject (Domain Name)
• Issuer (Certificate Authority)
• Public Key
• Valid From
• Valid To
• Serial Number
• Digital Signature
• Certificate Chain Information

------------------------------------------------------------

3. Certificate Authorities (CA)

Common Certificate Authorities:
• DigiCert
• GlobalSign
• Entrust
• Sectigo
• Let's Encrypt (commonly used for development/non-production)

The CA verifies domain ownership or organization identity and issues a signed certificate.

------------------------------------------------------------

4. TLS Handshake

Step 1:
Client (Browser/Application)
↓
Client Hello

Step 2:
Server
↓
Server Hello

Sends:
• Server Certificate
• Public Key
• Supported TLS Version
• Cipher Suite

Step 3:
Client validates:
• Hostname
• Expiry
• Certificate Chain
• Trusted Root CA
• Digital Signature
• Revocation Status (if available)

Step 4:
Session Keys are negotiated.

Step 5:
Encrypted HTTPS communication starts.

------------------------------------------------------------

5. Why does the Client validate the Server?

The client must ensure it is communicating with the genuine server and not an attacker.

The browser checks:
• Hostname matches the requested URL.
• Certificate is not expired.
• Certificate chain is valid.
• Certificate is signed by a trusted CA.
• Certificate has not been revoked (if checked).

Without these checks, Man-in-the-Middle (MITM) attacks would be possible.

------------------------------------------------------------

6. Does the Server validate the Client?

In Standard TLS:
• No.
• The server authenticates the client later using:
  - Username/Password
  - OAuth
  - JWT
  - API Keys
  - Session Cookies

In Mutual TLS (mTLS):
• Yes.
• Both client and server exchange and validate certificates.

Common mTLS Use Cases:
• Banking APIs
• Payment Gateways
• Government Systems
• Internal Microservices
• B2B Integrations

------------------------------------------------------------

7. Keystore vs Truststore

Keystore:
Contains:
• Private Key
• Server (or Client) Certificate
• Certificate Chain

Purpose:
Used to present the application's own identity during TLS.

Truststore:
Contains:
• Trusted Root CA Certificates
• Trusted Intermediate CA Certificates (if configured)

Purpose:
Used to validate certificates presented by the peer.

Important:
Keystore and Truststore are application-level concepts.

A server application may have both.
A client application may also have both (especially in mTLS).

------------------------------------------------------------

8. Root CA and Intermediate CA

Certificate Chain:

Root CA
   ↓
Intermediate CA
   ↓
Server Certificate

Why Intermediate CA?

• Root CA private key is highly sensitive.
• Root CA remains offline.
• Intermediate CAs perform day-to-day certificate issuance.
• If compromised, only the Intermediate CA is replaced.

------------------------------------------------------------

9. How does the Browser trust the Server Certificate?

The browser or operating system maintains a Trusted Root Certificate Store.

When the server presents its certificate:

Browser validates:
• Hostname
• Expiry
• Revocation (if available)
• Digital Signature
• Certificate Chain

The browser verifies that the certificate chain ends at a trusted Root CA already present in its trust store.

If all validations succeed:
Secure TLS connection is established.

------------------------------------------------------------

10. Does Google or Microsoft manually add every website certificate?

No.

Browsers and operating systems DO NOT store certificates for every website.

Instead, they store trusted Root CA certificates.

Examples:
• DigiCert Root CA
• GlobalSign Root CA
• Entrust Root CA
• Sectigo Root CA

These Root CAs are included through browser/OS updates after strict security audits and compliance reviews.

------------------------------------------------------------

11. Digital Signature Verification

Certificate Authority:
Signs the certificate using its Private Key.

Browser:
Uses the CA's Public Key (stored in the trusted Root CA certificate) to verify the signature.

If verification succeeds:
• Certificate has not been altered.
• Certificate was issued by a trusted CA.

------------------------------------------------------------

12. Certificate Lifecycle in Enterprise

Security/PKI Team:
• Purchase certificate.
• Generate CSR.
• Get certificate signed.
• Create PFX.

DevOps Team:
• Import PFX into Azure Key Vault.
• Configure Application Gateway.
• Configure AKS CSI Driver.
• Assign Managed Identity permissions.
• Validate HTTPS.
• Monitor expiry.
• Rotate certificates.

------------------------------------------------------------

13. Azure Key Vault

Purpose:
Securely stores:
• Secrets
• Keys
• Certificates

Certificate Formats:
• .pfx
• .crt
• .pem

Benefits:
• Centralized management.
• Versioning.
• Secure storage.
• RBAC integration.
• Managed Identity support.

------------------------------------------------------------

14. Application Gateway + Key Vault

Flow:

Security Team
↓
Certificate

↓

Azure Key Vault

↓

Managed Identity

↓

Application Gateway

↓

HTTPS Listener

↓

Client

Application Gateway retrieves certificates directly from Key Vault using Managed Identity.

No certificate files are manually copied to Application Gateway.

------------------------------------------------------------

15. AKS + Key Vault + CSI Driver

Flow:

Azure Key Vault

↓

Secrets Store CSI Driver

↓

SecretProviderClass

↓

Mounted Volume

↓

Application Pod

Applications read certificates or secrets from the mounted path.

No hardcoded secrets inside the application.

------------------------------------------------------------

16. Secrets Store CSI Driver

The Secrets Store CSI Driver is an open-source Kubernetes project.

In AKS:

Command:

az aks create --enable-addons azure-keyvault-secrets-provider

Automatically installs:
• Secrets Store CSI Driver
• Azure Key Vault Provider
• Required DaemonSets
• RBAC
• CRDs

Verification:

kubectl get pods -n kube-system

Expected Pods:
• secrets-store-csi-driver
• provider-azure

------------------------------------------------------------

17. Managed Identity

Managed Identity is an Azure AD identity automatically managed by Azure.

Advantages:
• No passwords.
• No client secrets.
• No certificate rotation.
• Azure automatically issues OAuth tokens.

Important:
Creating a Managed Identity DOES NOT automatically grant permissions.

Azure RBAC roles must still be assigned.

Examples:
• Key Vault Secrets User
• Key Vault Certificate User
• Reader
• Contributor

------------------------------------------------------------

18. System Assigned vs User Assigned Managed Identity

System Assigned:
• Created with the Azure resource.
• One identity per resource.
• Deleted automatically when the resource is deleted.

Example:
AKS → System Assigned Managed Identity

User Assigned:
• Independent Azure resource.
• Can be attached to multiple Azure resources.
• Survives even if attached resources are deleted.

Example:
One User Assigned Identity
↓
AKS
VM
Application Gateway
Function App

------------------------------------------------------------

19. Managed Identity vs Service Principal

Managed Identity:
• Azure-managed identity.
• No secrets.
• No password rotation.
• Preferred for Azure resources.

Service Principal:
• Client ID
• Client Secret or Certificate
• Used by workloads outside Azure.

Examples:
• Azure DevOps
• Jenkins
• Terraform on developer laptop
• Third-party CI/CD tools

------------------------------------------------------------
==========================
DEVOPS AZURE AKS INTERVIEW NOTES
PART 3 – INTERVIEW SCENARIOS & REAL-TIME EXPLANATIONS
==========================

1. AKS Security Architecture

Internet
    ↓
Azure Front Door (Optional)
    ↓
Application Gateway + WAF
    ↓
AGIC
    ↓
AKS Cluster
    ↓
Frontend Pod
    ↓
Backend Pod
    ↓
Azure SQL / Other Services

Security Layers:
• Azure Entra ID
• Azure RBAC
• Kubernetes RBAC
• Private AKS
• NSGs
• Azure Firewall
• Network Policies
• Azure Key Vault
• Managed Identity
• CSI Driver
• WAF
• Defender for Cloud
• Azure Policy
• Azure Monitor

------------------------------------------------------------

2. Complete Request Flow

User

↓

HTTPS

↓

Application Gateway

↓

TLS Certificate from Azure Key Vault

↓

AGIC

↓

Ingress

↓

Kubernetes Service

↓

Application Pod

↓

Database

------------------------------------------------------------

3. Complete Certificate Flow

Security Team

↓

Generate CSR

↓

Certificate Authority (DigiCert)

↓

Signed Certificate (.pfx)

↓

Azure Key Vault

↓

Managed Identity Permission

↓

Application Gateway

↓

HTTPS Listener

↓

Client

------------------------------------------------------------

4. Complete Secret Flow

Developer

↓

Stores Secret

↓

Azure Key Vault

↓

CSI Driver

↓

SecretProviderClass

↓

Mounted Volume

↓

Application Reads Secret

Advantages:
• No secrets in Git
• No secrets in Docker image
• No hardcoded passwords
• Centralized secret management

------------------------------------------------------------

5. Managed Identity Flow

AKS

↓

Managed Identity

↓

Azure AD

↓

OAuth Token

↓

Azure Key Vault

↓

Returns Secret

Important:
Managed Identity only authenticates.

Authorization is controlled by Azure RBAC.

------------------------------------------------------------

6. Service Principal Flow

Azure DevOps

↓

Service Connection

↓

Service Principal

↓

Azure AD

↓

OAuth Token

↓

Azure Resources

Used because Azure DevOps (Microsoft-hosted agents) is external to your Azure subscription.

------------------------------------------------------------

7. Browser Trust Flow

Browser

↓

Receives Server Certificate

↓

Checks:
• Hostname
• Expiry
• Revocation
• Digital Signature
• Certificate Chain

↓

Trusted Root CA Store

↓

TLS Connection Established

------------------------------------------------------------

8. Certificate Chain

Server Certificate

↓

Intermediate CA

↓

Root CA

The browser trusts the Root CA.

The Intermediate CA signs the Server Certificate.

------------------------------------------------------------

9. Why Root CA is kept Offline?

Reasons:
• Highest security.
• Prevent compromise.
• Root key is extremely valuable.
• Intermediate CA performs daily certificate issuance.

------------------------------------------------------------

10. Digital Signature Process

Certificate Authority

Private Key

↓

Signs Certificate

↓

Server Certificate

↓

Browser

↓

Uses CA Public Key

↓

Verifies Signature

If signature matches:
Certificate is trusted.

------------------------------------------------------------

11. Keystore vs Truststore

Keystore

Contains:
• Private Key
• Own Certificate
• Certificate Chain

Purpose:
Present application's identity.

Truststore

Contains:
• Trusted Root CA Certificates
• Trusted Intermediate CAs

Purpose:
Validate certificates presented by others.

Remember:

Keystore = "Who am I?"

Truststore = "Who do I trust?"

------------------------------------------------------------

12. Standard TLS vs Mutual TLS

Standard TLS

Client

↓

Validates Server

↓

Application Login

(username/password, JWT, OAuth)

Mutual TLS

Client

↓

Presents Certificate

↓

Server validates Client

AND

Server

↓

Presents Certificate

↓

Client validates Server

------------------------------------------------------------

13. AKS CSI Driver Interview Answer

"The Secrets Store CSI Driver is an open-source Kubernetes CSI project. AKS provides it as a managed add-on using the Azure Key Vault Provider. It securely mounts secrets and certificates from Azure Key Vault directly into pods without permanently storing them as Kubernetes Secrets unless secret synchronization is explicitly configured."

------------------------------------------------------------

14. Managed Identity Interview Answer

"Managed Identity is an Azure AD identity managed by Azure. It removes the need to store passwords or client secrets. Azure resources obtain OAuth tokens automatically from Azure AD and can access Azure services only after appropriate Azure RBAC permissions are assigned."

------------------------------------------------------------

15. Service Principal Interview Answer

"A Service Principal is an Azure AD application identity used by workloads running outside Azure or external automation tools. It authenticates using a Client ID and Client Secret (or certificate) and is commonly used by Azure DevOps, Jenkins and Terraform running outside Azure."

------------------------------------------------------------

16. Browser Trust Interview Answer

"Browsers and operating systems maintain a Trusted Root Certificate Store. During the TLS handshake, the server presents its certificate chain. The browser validates the hostname, expiry, digital signature, revocation status and verifies that the chain ends at a trusted Root CA already present in the trust store."

------------------------------------------------------------

17. Why Managed Identity over Service Principal?

Managed Identity:
• No password.
• No secret rotation.
• More secure.
• Azure-managed.
• Recommended for Azure resources.

Service Principal:
• Requires secret management.
• Used when workloads run outside Azure.

------------------------------------------------------------

18. Common Interview Questions & Answers

Q. Who installs the CSI Driver?

Answer:
AKS automatically installs it when the Azure Key Vault Secrets Provider add-on is enabled.

------------------------------------------------------------

Q. Does Managed Identity automatically get permissions?

Answer:
No. It must be assigned Azure RBAC roles such as Key Vault Secrets User or Reader.

------------------------------------------------------------

Q. Difference between System Assigned and User Assigned Managed Identity?

Answer:
System Assigned is tied to one resource and deleted with it.
User Assigned is an independent Azure resource that can be attached to multiple Azure resources.

------------------------------------------------------------

Q. Why does the client validate the server?

Answer:
To ensure it is communicating with the legitimate server and prevent man-in-the-middle attacks.

------------------------------------------------------------

Q. Why doesn't the server validate the client in HTTPS?

Answer:
In standard TLS, the server usually authenticates the client at the application layer using credentials such as usernames, passwords, OAuth tokens or JWTs. Certificate-based client validation is used only in Mutual TLS (mTLS).

------------------------------------------------------------

Q. Does the browser store every website certificate?

Answer:
No. It stores only trusted Root CA certificates. Website certificates are validated through the certificate chain.

------------------------------------------------------------

Q. Who adds DigiCert to the browser?

Answer:
Browser and operating system vendors (Microsoft, Apple, Mozilla, etc.) include trusted Root CA certificates after the CA passes strict security audits and complies with industry requirements.

------------------------------------------------------------

Q. What is stored in Azure Key Vault?

Answer:
• Secrets
• Keys
• Certificates
• Certificate versions

------------------------------------------------------------

Q. Who manages certificates in production?

Answer:
Typically, the Security/PKI team procures and renews certificates, while the DevOps team imports them into Azure Key Vault, configures Application Gateway/AKS, assigns permissions and validates secure communication.

------------------------------------------------------------

19. One-Minute Interview Summary

"Our AKS platform follows enterprise security best practices. User authentication is integrated with Azure Entra ID, authorization is controlled using Azure RBAC and Kubernetes RBAC, and the cluster is deployed as a Private AKS. Network traffic is protected using NSGs, Azure Firewall, WAF and Network Policies. Secrets and certificates are securely stored in Azure Key Vault and accessed through Managed Identity using the Secrets Store CSI Driver. Infrastructure is provisioned with Terraform, monitored using Azure Monitor and Defender for Cloud, and all outbound traffic is routed through Azure Firewall using User Defined Routes for centralized inspection and compliance."
