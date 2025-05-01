# Ksau-Backend-DevOps

DevOps configurations and deployment tools for the ksau-web-backend application on Kubernetes.

## Prerequisites

Before deploying the application, ensure you have the following:

- AWS Account with appropriate permissions
- AWS CLI installed and configured
- eksctl installed
- kubectl installed
- Helm v3 installed (for Helm deployment method)
- Access to the Docker image repository (ksauraj/ksau-web-backend)

## Setting Up the Infrastructure

### Creating an EKS Cluster with eksctl

1. Use the provided EKS cluster configuration file:

```bash
eksctl create cluster -f eks-cluster-config.yaml
```

The configuration uses t3.small instances to optimize costs while providing sufficient resources for the application.

This process will take approximately 15-20 minutes to complete.

2. Verify that your cluster is created and configured correctly:

```bash
kubectl get nodes
```

### Installing NGINX Ingress Controller for AWS

1. Install the NGINX Ingress Controller using the AWS-specific manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.2/deploy/static/provider/aws/deploy.yaml
```

2. Verify the installation:

```bash
kubectl get pods -n ingress-nginx
```

3. Get the External IP/DNS of the NGINX Ingress Controller:

```bash
kubectl get service ingress-nginx-controller -n ingress-nginx
```

The output will include an EXTERNAL-IP field with a value like `a1b2c3d4e5f6g7h8i9j0.us-east-1.elb.amazonaws.com`. This is the AWS Load Balancer DNS name that will be used for DNS configuration.

### Configuring DNS

#### If You Own a Domain

1. Go to your domain registrar's DNS management page.

2. Add a new DNS record:
   - Type: CNAME
   - Name: aws (or your preferred subdomain)
   - Value: The EXTERNAL-IP from the previous step
   - TTL: 300 (or as low as possible for faster propagation)

3. Wait for DNS propagation (can take up to 24-48 hours, but often much faster).

#### For Testing Without a Domain

If you don't own a domain or want to test quickly, you can use `/etc/hosts` file (or Windows equivalent):

1. Edit your hosts file:
   - On Linux/Mac: `sudo nano /etc/hosts`
   - On Windows: Edit `C:\Windows\System32\drivers\etc\hosts` as Administrator

2. Add a line with the following format:
   ```
   <Load Balancer IP> aws.ksauraj.eu.org
   ```

3. Save the file.

This method only works for local testing on your machine.

## Deployment Methods

### Method 1: Using Kubernetes Manifests

To deploy the application using raw Kubernetes manifests:

```bash
# Apply the deployment
kubectl apply -f k8s/manifests/deployment.yaml

# Apply the service
kubectl apply -f k8s/manifests/service.yaml

# Apply the ingress
kubectl apply -f k8s/manifests/ingress.yaml
```

Verify the deployment:

```bash
kubectl get deployments,services,ingress
```

### Method 2: Using Helm Chart

#### Installing the Chart

To install the chart with the release name `my-release`:

```bash
# Navigate to the Helm chart directory
cd helm/ksau-web-charts

# Install the chart
helm install my-release .
```

#### Installing with Custom Values

You can override the default configuration values during installation:

```bash
helm install my-release . --set service.type=NodePort --set ingress.host=example.com
```

Alternatively, you can provide a custom values file:

```bash
helm install my-release . -f custom-values.yaml
```

#### Verifying the Installation

After installation, verify that the resources have been created:

```bash
kubectl get deployments,services,ingress -l app=ksau-web-backend
```

## Helm Chart Configuration

The following table lists the configurable parameters of the ksau-web-charts chart and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `labels.app` | Application label | `ksau-web-backend` |
| `labels.env` | Environment label | `dev` |
| `container.name` | Container name | `ksau-web-backend` |
| `container.image.name` | Container image name | `ksauraj/ksau-web-backend` |
| `container.image.tag` | Container image tag | `v1-alpine` |
| `deployment.name` | Deployment name | `ksau-web-backend-svc` |
| `deployment.replicaCount` | Number of replicas | `1` |
| `service.name` | Service name | `ksau-web-backend-svc` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `80` |
| `service.targetPort` | Container port | `8080` |
| `ingress.name` | Ingress name | `ksau-web-backend-ing` |
| `ingress.host` | Ingress host | `aws.ksauraj.eu.org` |

To modify these parameters, either edit the `values.yaml` file or provide your own values file.

## Usage

### Accessing the Application

Once deployed, the application will be accessible at the URL specified in the ingress configuration:

```
http://aws.ksauraj.eu.org
```

### Scaling the Application

To scale the deployment to 3 replicas:

```bash
kubectl scale deployment ksau-web-backend --replicas=3
```

Or if using Helm, update the `values.yaml` file and upgrade the release:

```bash
helm upgrade my-release . --set deployment.replicaCount=3
```

### Upgrading the Helm Chart

To upgrade the chart with a new version or configuration:

```bash
helm upgrade my-release .
```

## Uninstallation

### Removing Kubernetes Resources

To remove the resources created with Kubernetes manifests:

```bash
kubectl delete -f k8s/manifests/ingress.yaml
kubectl delete -f k8s/manifests/service.yaml
kubectl delete -f k8s/manifests/deployment.yaml
```

### Removing Helm Release

To uninstall/delete the `my-release` deployment:

```bash
helm uninstall my-release
```

This will remove all the Kubernetes resources associated with the chart.

## Troubleshooting

### Common Issues

#### Ingress Not Working

1. Verify the NGINX Ingress Controller is running:
   ```bash
   kubectl get pods -n ingress-nginx
   ```

2. Check the ingress resource:
   ```bash
   kubectl describe ingress ksau-web-backend
   ```

3. Ensure your DNS is properly configured and has propagated.

#### Application Pod Not Starting

1. Check the pod status:
   ```bash
   kubectl get pods -l app=ksau-web-backend
   ```

2. View the pod logs:
   ```bash
   kubectl logs <pod-name>
   ```

3. Describe the pod for events:
   ```bash
   kubectl describe pod <pod-name>
   ```

### Cleaning Up Resources

To delete the EKS cluster when you're done:

```bash
eksctl delete cluster --name ksau-cluster --region us-east-1
```

This will remove all AWS resources created for the cluster, including the load balancer.
