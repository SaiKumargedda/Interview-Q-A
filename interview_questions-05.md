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
