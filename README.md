# 🚀 PayMyBuddy - Kubernetes Deployment

A complete guide to deploy the PayMyBuddy application on Kubernetes with MySQL database, MetalLB load balancer, and Nginx Ingress Controller.

---

## 📋 Table of Contents

- [Prerequisites](#-prerequisites)
- [Architecture](#-architecture)
- [Installation](#-installation)
  - [Step 1: Build Docker Image](#step-1-build-docker-image)
  - [Step 2: Create Namespace](#step-2-create-namespace)
  - [Step 3: Configure Secrets](#step-3-configure-secrets)
  - [Step 4: Install MetalLB](#step-4-install-metallb)
  - [Step 5: Install Ingress Controller](#step-5-install-ingress-controller)
  - [Step 6: Deploy Resources](#step-6-deploy-resources)
- [Verification](#-verification)
- [Troubleshooting](#-troubleshooting)

---

## 🔧 Prerequisites

| Requirement | Version |
|-------------|---------|
| Kubernetes | 1.19+ |
| kubectl | Latest |
| Docker | Latest |
| Docker Hub Account | Required |

> 💡 **Tip:** For this project, a small VM from [KillerCoda](https://killercoda.com/) with Kubernetes pre-installed was used.

---

## 🏗️ Architecture
```
                    Internet / Browser
                           │
                           ▼
                ┌─────────────────────┐
                │      MetalLB        │
                │   Load Balancer     │
                └─────────────────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │  Ingress Controller │
                │       (Nginx)       │
                └─────────────────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │      Ingress        │
                │  paymybuddy.local   │
                └─────────────────────┘
                           │
                           ▼
        ┌──────────────────┴──────────────────┐
        │                                      │
        ▼                                      ▼
┌───────────────┐                    ┌───────────────┐
│  PayMyBuddy   │ ──────────────────▶│    MySQL      │
│   Service     │                    │   Service     │
│  (ClusterIP)  │                    │  (ClusterIP)  │
└───────────────┘                    └───────────────┘
        │                                      │
        ▼                                      ▼
┌───────────────┐                    ┌───────────────┐
│  PayMyBuddy   │                    │    MySQL      │
│     Pod       │                    │     Pod       │
└───────────────┘                    └───────────────┘
```

---

## 🚀 Installation

### Step 1: Build Docker Image

Follow the instructions in [build_docker.md](./build_docker.md) to build and push the Docker image to your private registry.

---
### Step 2: Create Namespace
```bash
cd k8s/
kubectl apply -f namespace.yml
```

![namespace](imgs/namespace.png)

✅ **Expected result:** Namespace `paymybuddy` created successfully.

---

### Step 3: Configure Secrets

#### 🔐 Docker Hub Secret

Create a secret to pull images from your private Docker Hub repository:
```bash
kubectl create secret docker-registry dockerhub-secret \
  --namespace=paymybuddy \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOUR_DOCKER_USERNAME \
  --docker-password=YOUR_PASSWORD \
  --docker-email=YOUR_EMAIL_ADDRESS
```

![docker secret](imgs/docker-secret.png)

#### 🔐 MySQL Secrets
```bash
kubectl apply -f mysql-secrets.yml
```

![mysql secret](imgs/secrets-creation.png)

> 💡 **Alternative:** You can also create MySQL secrets via CLI:
> ```bash
> kubectl create secret generic mysql-secret \
>   --namespace=paymybuddy \
>   --from-literal=mysql-root-password=rootpassword \
>   --from-literal=mysql-database=paymybuddy \
>   --from-literal=mysql-user=paymybuddy \
>   --from-literal=mysql-password=paymybuddy \
>   --from-literal=spring-datasource-url=jdbc:mysql://mysql:3306/paymybuddy
> ```

---

### Step 4: Install MetalLB

MetalLB provides external IP addresses for LoadBalancer services in bare-metal environments.
```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# Wait for pods to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=120s
```

![metallb](imgs/metallb-install.png)

#### 📍 Configure IP Range

Find your machine's IP address to configure the IP pool:
```bash
hostname -I | awk '{print $1}'
```

![hostname ip](imgs/hostname.png)

> ⚠️ **Important:** Update the IP range in `paymybuddy-lb.yml` based on your network.
> 
> Example: If your IP is `10.0.29.5`, use range `10.0.29.200-10.0.29.250`

---

### Step 5: Install Ingress Controller

The Nginx Ingress Controller manages external access to services via HTTP/HTTPS routing.
```bash
# Install Nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml

# Wait for the pod to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

![ingress controller](imgs/ingress-controller.png)

---

### Step 6: Deploy Resources

Deploy all Kubernetes resources:
```bash
kubectl apply -f .
```

---

## ✅ Verification

### 🔍 Check Secrets
```bash
# List all secrets
kubectl get secret -n paymybuddy
```

![show secrets](imgs/show-secrets.png)
```bash
# View secret content (base64 encoded)
kubectl get secret mysql-secret -n paymybuddy -o yaml

# Decode a secret value
kubectl get secret mysql-secret -n paymybuddy -o jsonpath='{.data.mysql-root-password}' | base64 --decode
```

---

### 🔍 Check Resources
```bash
# List all resources in the namespace
kubectl get all -n paymybuddy
```

![all resources](imgs/get-all-resources.png)

---

### 🔍 Check Database Connection
```bash
# Connect to MySQL pod
kubectl exec -it -n paymybuddy deployment/mysql -- mysql -u paymybuddy -ppaymybuddy -e "SHOW DATABASES;"

# Verify Spring datasource configuration
kubectl exec -it -n paymybuddy deployment/paymybuddy -- env | grep SPRING
```

![database connection](imgs/test-mysql-connection.png)

---

### 🔍 Check Application Logs
```bash
kubectl logs -n paymybuddy -l app=paymybuddy
```

---

### 🔍 Check Ingress Configuration
```bash
# Verify IP attribution
kubectl get svc -n ingress-nginx
```

![nginx ingress](imgs/ip-attribution.png)
```bash
# Verify Ingress
kubectl get ingress -n paymybuddy
```

![ingress service](imgs/ingress-service.png)

---

### 🌐 Access the Application
```bash
# Get Ingress Controller IP
INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Ingress IP: $INGRESS_IP"

# Add hostname to /etc/hosts
echo "$INGRESS_IP paymybuddy.local" | sudo tee -a /etc/hosts
```

![hosts file](imgs/hosts-file.png)
```bash
# Test the application
curl -v --connect-timeout 10 http://paymybuddy.local
```

![curl test](imgs/curl-test.png)

✅ **Expected result:** HTTP 302 redirect to `/login` page.

---

## 🛠️ Troubleshooting

| Issue | Solution |
|-------|----------|
| `ImagePullBackOff` | Check Docker Hub credentials and secret |
| `EXTERNAL-IP` pending | Verify MetalLB configuration and IP range |
| Ingress class `NONE` | Add `ingressClassName: nginx` to Ingress manifest |
| 404 Not Found | Verify Ingress rules and service endpoints |
| Connection timeout | Check pod status and application logs |

### Useful Commands
```bash
# Check pod status
kubectl get pods -n paymybuddy

# View pod logs
kubectl logs -n paymybuddy -l app=paymybuddy

# Describe pod for events
kubectl describe pod -n paymybuddy -l app=paymybuddy

# Check endpoints
kubectl get endpoints -n paymybuddy
```

---

## 📁 Project Structure
```
k8s/
├── namespace.yml
├── mysql-secrets.yml
├── mysql-deployment.yml
├── mysql-service.yml
├── paymybuddy-deployment.yml
├── paymybuddy-service.yml
├── paymybuddy-ingress.yml
└── paymybuddy-lb.yml
```

---

## 📚 Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [MetalLB Documentation](https://metallb.universe.tf/)
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)

---

## 👨‍💻 Author

**Kevin Lagaza**

---

## 📄 License

This project is licensed under the MIT License.