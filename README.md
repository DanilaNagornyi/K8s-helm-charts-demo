# Helm Demo: MongoDB with Mongo Express on DigitalOcean

![Helm](https://img.shields.io/badge/helm-%230F1689.svg?style=for-the-badge&logo=helm&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![DigitalOcean](https://img.shields.io/badge/DigitalOcean-%230167ff.svg?style=for-the-badge&logo=digitalOcean&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-%234ea94b.svg?style=for-the-badge&logo=mongodb&logoColor=white)
![Nginx](https://img.shields.io/badge/nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)

A production-ready deployment of MongoDB ReplicaSet with Mongo Express web interface, deployed using Helm on DigitalOcean Kubernetes cluster with Nginx Ingress.

## 📋 Overview

This project demonstrates:
- Deploying MongoDB ReplicaSet (3 replicas) using Helm
- Using DigitalOcean Block Storage for persistent volumes
- Exposing Mongo Express web UI via Nginx Ingress
- Managing secrets and configuration with Helm values
- Setting up external access using nip.io DNS

## 🏗️ Architecture

```
Internet
    ↓
Nginx Ingress Controller
    ↓ (mongo-express.139.59.219.50.nip.io)
Mongo Express Service (8081)
    ↓
Mongo Express Pod
    ↓
MongoDB ReplicaSet
├── mongodb-0 (Primary)
├── mongodb-1 (Secondary)
└── mongodb-2 (Secondary)
    ↓
DigitalOcean Block Storage (Persistent Volumes)
```

## 📁 Project Structure

```
.
├── helm-mongodb.yaml              # Helm values for MongoDB chart
├── helm-mongo-express.yaml        # Mongo Express deployment & service
├── helm-ingress.yaml              # Ingress configuration
└── k8s-*-kubeconfig.yaml         # DigitalOcean cluster kubeconfig
```

## 🚀 Quick Start

### Prerequisites

- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
- [Helm](https://helm.sh/docs/intro/install/) v3+ installed
- [doctl](https://docs.digitalocean.com/reference/doctl/how-to/install/) (DigitalOcean CLI)
- DigitalOcean Kubernetes cluster provisioned

### 1. Configure kubectl for DigitalOcean

```bash
# Download kubeconfig from DigitalOcean
doctl kubernetes cluster kubeconfig save <cluster-name>

# Or use the provided kubeconfig file
export KUBECONFIG=/path/to/k8s-*-kubeconfig.yaml

# Verify connection
kubectl cluster-info
```

### 2. Install Nginx Ingress Controller

```bash
# Add Nginx Ingress Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install Nginx Ingress Controller
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --set controller.publishService.enabled=true
```

Wait for the LoadBalancer to get an external IP:
```bash
kubectl get svc nginx-ingress-ingress-nginx-controller
```

### 3. Add MongoDB Helm Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 4. Deploy MongoDB ReplicaSet

```bash
helm install mongodb bitnami/mongodb -f helm-mongodb.yaml
```

**Configuration details:**
- 3-node ReplicaSet for high availability
- DigitalOcean Block Storage for persistence
- Root password: `secret-root-pwd` (change in production!)
- Metrics disabled

### 5. Deploy Mongo Express

```bash
kubectl apply -f helm-mongo-express.yaml
```

**Default credentials:**
- Username: `admin`
- Password: `admin`

### 6. Configure Ingress

Update `helm-ingress.yaml` with your LoadBalancer IP:
```yaml
- host: mongo-express.<YOUR_LOADBALANCER_IP>.nip.io
```

Apply the Ingress:
```bash
kubectl apply -f helm-ingress.yaml
```

### 7. Access Mongo Express

Open your browser:
```
http://mongo-express.<YOUR_LOADBALANCER_IP>.nip.io
```

Login with credentials: `admin` / `admin`

## 🔍 Verification Commands

```bash
# Check Helm releases
helm list

# Check MongoDB pods
kubectl get pods -l app.kubernetes.io/name=mongodb

# Check MongoDB service
kubectl get svc mongodb-headless

# Check persistent volumes
kubectl get pvc

# Check Mongo Express deployment
kubectl get pods -l app=mongo-express

# Check Ingress
kubectl get ingress mongo-express

# View MongoDB logs
kubectl logs -l app.kubernetes.io/name=mongodb -f

# Check ReplicaSet status
kubectl exec mongodb-0 -- mongosh --eval "rs.status()"
```

## 📊 MongoDB Configuration

The MongoDB deployment uses the following configuration:

| Parameter | Value | Description |
|-----------|-------|-------------|
| Architecture | ReplicaSet | High availability setup |
| Replicas | 3 | Number of MongoDB instances |
| Storage Class | do-block-storage | DigitalOcean persistent storage |
| Root Password | secret-root-pwd | Admin password (stored in secret) |
| Image | bitnamisecure/mongodb:latest | Bitnami secure MongoDB image |
| Metrics | Disabled | Prometheus metrics integration |

## 🔐 Security Notes

**⚠️ IMPORTANT - Production Checklist:**

1. **Change default passwords** in `helm-mongodb.yaml`:
   ```yaml
   auth:
     rootPassword: <strong-password>
   ```

2. **Update Mongo Express credentials** in `helm-mongo-express.yaml`:
   ```yaml
   - name: ME_CONFIG_BASICAUTH_USERNAME
     value: <your-username>
   - name: ME_CONFIG_BASICAUTH_PASSWORD
     value: <strong-password>
   ```

3. **Use Kubernetes Secrets** instead of plain text:
   ```bash
   kubectl create secret generic mongo-express-auth \
     --from-literal=username=admin \
     --from-literal=password=<strong-password>
   ```

4. **Enable TLS/SSL** for Ingress:
   - Install cert-manager
   - Configure Let's Encrypt certificates
   - Update Ingress with TLS configuration

5. **Network Policies**: Restrict pod-to-pod communication

6. **RBAC**: Configure proper role-based access control

## 🛠️ Troubleshooting

### MongoDB pods not starting
```bash
# Check pod events
kubectl describe pod mongodb-0

# Check storage provisioning
kubectl get pvc
kubectl describe pvc <pvc-name>
```

### Mongo Express connection issues
```bash
# Check logs
kubectl logs -l app=mongo-express

# Verify MongoDB service
kubectl get svc mongodb-headless

# Test connection from Mongo Express pod
kubectl exec <mongo-express-pod> -- mongosh mongodb://root:secret-root-pwd@mongodb-0.mongodb-headless:27017
```

### Ingress not working
```bash
# Check Ingress controller
kubectl get pods -n ingress-nginx

# Check Ingress configuration
kubectl describe ingress mongo-express

# Verify LoadBalancer IP
kubectl get svc -n ingress-nginx
```

## 🧹 Cleanup

```bash
# Delete Ingress
kubectl delete -f helm-ingress.yaml

# Delete Mongo Express
kubectl delete -f helm-mongo-express.yaml

# Uninstall MongoDB (this will delete PVCs)
helm uninstall mongodb

# Uninstall Nginx Ingress
helm uninstall nginx-ingress

# Delete persistent volumes (if needed)
kubectl delete pvc --all
```

## 📚 Resources

- [Helm Documentation](https://helm.sh/docs/)
- [Bitnami MongoDB Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/mongodb)
- [DigitalOcean Kubernetes](https://docs.digitalocean.com/products/kubernetes/)
- [MongoDB ReplicaSet](https://www.mongodb.com/docs/manual/replication/)
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [nip.io DNS Service](https://nip.io/)

## 💡 Next Steps

- Configure automated backups for MongoDB
- Set up monitoring with Prometheus and Grafana
- Implement TLS/SSL certificates
- Configure network policies
- Set up RBAC and pod security policies
- Implement MongoDB user management

## 📄 License

This project is for educational and demonstration purposes.
