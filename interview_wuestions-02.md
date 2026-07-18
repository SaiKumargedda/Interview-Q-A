# DevOps Interview Notes (This Chat Only)

# 1. What is Apache Tomcat Server? What is its use?

## Answer

Apache Tomcat is an open-source Java Application Server (Servlet Container) used to run Java web applications developed using Servlets and JSP.

### Uses
- Hosts Java web applications
- Deploys WAR files
- Receives HTTP/HTTPS requests from users
- Executes the application and returns the response

### Example

Developer builds a Spring Boot application:

```bash
mvn clean package
```

This generates:

```text
app.war
```

Deploy the WAR into:

```text
/opt/tomcat/webapps/
```

Tomcat automatically deploys the application.

**Interview Tip:**
In modern microservices, Spring Boot usually uses Embedded Tomcat, so we deploy a JAR instead of a WAR.

---

# 2. What is the purpose of WORKDIR in Docker?

## Answer

WORKDIR sets the default working directory inside the container.

Instead of repeatedly using:

```dockerfile
RUN cd /app
RUN mvn clean package
```

(which doesn't work because each RUN creates a new layer)

we use:

```dockerfile
WORKDIR /app

COPY . .

RUN mvn clean package
```

### Benefits

- No need to use `cd`
- Cleaner Dockerfile
- Relative paths work properly
- Easier maintenance

---

# 3. Purpose of ENTRYPOINT in Docker

## Answer

ENTRYPOINT defines the main process that starts whenever the container runs.

Example:

```dockerfile
ENTRYPOINT ["java","-jar","app.jar"]
```

Running:

```bash
docker run myimage
```

actually executes:

```bash
java -jar app.jar
```

### ENTRYPOINT vs CMD

| ENTRYPOINT | CMD |
|------------|-----|
| Main executable | Default arguments |
| Difficult to override | Easy to override |
| Container behaves like an application | Supplies default parameters |

Example:

```dockerfile
ENTRYPOINT ["java","-jar","app.jar"]
CMD ["--server.port=8080"]
```

Runs:

```bash
java -jar app.jar --server.port=8080
```

---

# 4. Difference between Maven and Gradle

| Maven | Gradle |
|--------|---------|
| Uses XML (pom.xml) | Uses Groovy/Kotlin DSL |
| Convention based | Highly customizable |
| Slower | Faster due to caching & incremental builds |
| Fixed lifecycle | Flexible task execution |
| Easier to learn | Better for very large projects |

### Which one did we use?

We used Maven because:

- Spring Boot project
- Easy dependency management
- SonarQube integration
- Enterprise standard

---

# 5. Azure provides an HTTP endpoint. How do you enable HTTPS?

## Answer

HTTPS is generally terminated at Azure Application Gateway or Azure Front Door.

### Flow

```
Internet
     │
Azure Front Door (Optional)
     │
Application Gateway (HTTPS Listener)
     │
AKS Ingress
     │
Pods
```

### Steps

1. Obtain an SSL certificate.
2. Store it in Azure Key Vault or directly in Application Gateway.
3. Create an HTTPS Listener.
4. Bind the certificate to the listener.
5. Configure TLS in the Kubernetes Ingress.

Example:

```yaml
tls:
- hosts:
  - app.company.com
  secretName: tls-secret
```

6. Configure HTTP to HTTPS redirection.

---

# 6. What dependencies are defined in pom.xml?

## Answer

The pom.xml file contains:

- Project Information
- Dependencies
- Plugins
- Build Configuration
- Java Version
- Repositories
- Profiles

Example:

```xml
<dependencies>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>

<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
</dependency>

</dependencies>
```

Dependencies are external libraries required by the application during build and runtime.

---

# 7. Pipeline succeeded but the application is running the old version. How do you troubleshoot?

## Answer

I troubleshoot layer by layer.

### Step 1 – Verify Pipeline Build

- Verify the build completed successfully.
- Check the Build ID.
- Verify the Docker image tag.
- Confirm the correct artifact was generated.

---

### Step 2 – Verify Image in Azure Container Registry (ACR)

Ensure the latest image was pushed successfully.

---

### Step 3 – Verify Kubernetes Deployment

```bash
kubectl describe deployment <deployment-name>
```

Confirm the deployment references the latest image tag.

---

### Step 4 – Verify Rollout

```bash
kubectl rollout status deployment <deployment-name>
```

Ensure Kubernetes completed the rollout successfully.

---

### Step 5 – Verify Running Pods

```bash
kubectl get pods
```

Ensure newly created pods are running and old pods are terminated.

---

### Step 6 – Verify Image Pull Policy

If using:

```yaml
image: latest
imagePullPolicy: IfNotPresent
```

the node may reuse an old cached image.

**Best Practice**

- Never use the `latest` tag.
- Use immutable image tags such as Build ID or Git Commit.
- Prefer `imagePullPolicy: Always` if required.

---

### Step 7 – Verify Service or Ingress

Ensure traffic is routed to the newly created pods instead of old pods.

---

### Step 8 – Check Application Logs

```bash
kubectl logs <pod-name>
```

Look for application startup failures or runtime errors.

---

### Step 9 – Restart Deployment if Required

```bash
kubectl rollout restart deployment <deployment-name>
```

---

# 8. How do you deploy using Blue-Green Deployment?

## Answer

Assume:

```
Blue = Current Production Version

Green = New Version
```

### Step 1

Deploy the Green version while Blue continues serving production traffic.

```
Blue Pods (v1)

Green Pods (v2)
```

---

### Step 2

Validate the Green environment by testing it internally.

---

### Step 3

Switch Traffic

If using Kubernetes Service:

Before:

```yaml
selector:
  version: blue
```

After:

```yaml
selector:
  version: green
```

If using Azure Application Gateway or Azure Front Door:

- Update the backend pool or routing rule to point to the Green deployment.

---

### Step 4

Monitor:

- Application Logs
- CPU Usage
- Memory Usage
- Error Rate
- Health Checks

---

### Step 5

If everything is healthy:

Remove the Blue deployment after confirmation.

---

### Step 6

Rollback

If an issue occurs, immediately switch traffic back to the Blue deployment.

Rollback is completed within seconds because both environments already exist.

---

# 9. Does Azure provide SSL certificates? How did you manage them?

## Answer

Azure does **not automatically issue public SSL certificates** for your applications.

Certificates are generally obtained from:

- DigiCert
- GlobalSign
- Sectigo
- Let's Encrypt
- Internal Enterprise PKI

### How We Managed Certificates

1. The Security team procured the SSL certificate.
2. We stored it securely in Azure Key Vault.
3. Azure Application Gateway accessed the certificate using Managed Identity.
4. The certificate was bound to the HTTPS Listener.
5. During certificate renewal, we updated the certificate in Azure Key Vault and associated the updated certificate with the HTTPS Listener, eliminating manual certificate deployment on servers.

---

# 10. Can we add a validation stage so the deployment pipeline succeeds only after the application is deployed successfully?

## Answer

Yes. This is a best practice followed in enterprise CI/CD pipelines.

Instead of marking the deployment successful immediately after applying the Kubernetes manifests, we add a **Post-Deployment Validation Stage**.

### Validation Steps

#### Wait for Rollout Completion

```bash
kubectl rollout status deployment/my-app --timeout=300s
```

---

#### Verify Pods are Ready

```bash
kubectl wait --for=condition=Ready pod -l app=my-app --timeout=300s
```

---

#### Verify the Deployed Image Version

```bash
kubectl get deployment my-app -o=jsonpath='{.spec.template.spec.containers[0].image}'
```

Compare the deployed image tag with the image built in the pipeline.

---

#### Perform Health Check

Example:

```
https://app.company.com/actuator/health
```

The endpoint should return **HTTP 200 OK**.

---

#### Run Smoke Tests (Optional)

Execute a few API calls to verify that the application is functioning correctly.

---

### Azure DevOps Example

```yaml
- bash: |
    kubectl rollout status deployment/my-app --timeout=300s
    kubectl wait --for=condition=Ready pod -l app=my-app --timeout=300s
```

If any validation fails, the stage fails and the deployment pipeline is marked as **Failed**.

---

# Interview-Ready Answer

> "In our Azure DevOps deployment pipeline, we don't consider the deployment complete immediately after applying the Kubernetes manifests. We include a post-deployment validation stage that waits for the rollout to complete, verifies that the expected image version is deployed, confirms all pods are in the Ready state, performs health checks (and smoke tests if applicable), and only then marks the pipeline as successful. This ensures the application is actually running the new version before reporting a successful deployment."
