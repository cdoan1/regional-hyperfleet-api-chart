# regional-hyperfleet-api-chart

A Helm chart that deploys an ArgoCD Application resource to manage the installation of the [hyperfleet-api](https://github.com/openshift-hyperfleet/hyperfleet-api) Helm chart.

## Prerequisites

- ArgoCD installed in your Kubernetes cluster
- Helm 3.x
- AWS Secrets Store CSI Driver (if using SecretProviderClass for AWS Pod Identity)
- Properly configured IAM roles for service accounts (IRSA) or EKS Pod Identity

## Installation

```bash
helm install regional-hyperfleet-api . -n argocd
```

Or with custom values:

```bash
helm install regional-hyperfleet-api . -n argocd -f custom-values.yaml
```

## Configuration

The following table lists the configurable parameters:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `hyperfleet-api-chart.applicationName` | Name of the ArgoCD Application | `hyperfleet-api` |
| `hyperfleet-api-chart.namespace` | Namespace where ArgoCD Application will be created | `argocd` |
| `hyperfleet-api-chart.targetNamespace` | Namespace where hyperfleet-api will be deployed | `hyperfleet-system` |
| `hyperfleet-api-chart.project` | ArgoCD Project | `default` |
| `hyperfleet-api-chart.source.repoURL` | Git repository URL | `https://github.com/openshift-hyperfleet/hyperfleet-api.git` |
| `hyperfleet-api-chart.source.targetRevision` | Branch, tag, or commit | `v0.1.1` |
| `hyperfleet-api-chart.source.path` | Path to the chart in the repository | `charts` |
| `hyperfleet-api-chart.syncPolicy.automated.prune` | Enable automatic pruning | `true` |
| `hyperfleet-api-chart.syncPolicy.automated.selfHeal` | Enable self-healing | `true` |
| `hyperfleet-api-chart.helmValues` | Additional values to pass to hyperfleet-api chart | See below |
| `secretProviderClass.enabled` | Enable SecretProviderClass for AWS secrets | `true` |
| `secretProviderClass.name` | Name of the SecretProviderClass | `hyperfleet-api-aws-secrets` |
| `secretProviderClass.region` | AWS region for Secrets Manager | `us-east-1` |
| `secretProviderClass.secretName` | AWS Secrets Manager secret name/ARN | `hyperfleet-api/rds-credentials` |
| `secretProviderClass.objects` | Objects to retrieve from AWS Secrets Manager | See values.yaml |
| `secretProviderClass.secretObjects` | Kubernetes secrets to create from retrieved data | See values.yaml |

## AWS Pod Identity and RDS Access

This chart includes support for AWS Pod Identity to securely access RDS database credentials from AWS Secrets Manager. The `SecretProviderClass` resource is automatically created and configured to:

1. Retrieve RDS credentials from AWS Secrets Manager
2. Create a Kubernetes secret with the credentials
3. Mount the secret in the hyperfleet-api pods
4. Inject database connection details as environment variables

### Setup Requirements

1. **Install AWS Secrets Store CSI Driver**:
   ```bash
   helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
   helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system
   ```

2. **Install AWS Secrets Manager and Config Provider**:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
   ```

3. **Configure IAM Role for Service Account**:
   - Create an IAM role with permissions to access the Secrets Manager secret
   - Annotate the service account with the IAM role ARN
   - Ensure the trust policy allows the service account to assume the role

4. **Create Secret in AWS Secrets Manager**:
   The secret should be JSON formatted with the following structure:
   ```json
   {
     "username": "dbuser",
     "password": "dbpassword",
     "host": "db.example.com",
     "port": "5432",
     "database": "hyperfleet"
   }
   ```

## Example Custom Values

```yaml
hyperfleet-api-chart:
  targetNamespace: my-namespace
  source:
    targetRevision: v1.0.0
  helmValues:
    serviceAccount:
      name: custom-sa
      annotations:
        eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/hyperfleet-api-role
    image:
      tag: latest
    replicas: 3

secretProviderClass:
  enabled: true
  region: us-west-2
  secretName: my-app/rds-credentials
```

### Disabling SecretProviderClass

If you don't need AWS Pod Identity integration, you can disable it:

```yaml
secretProviderClass:
  enabled: false
```

## Uninstallation

```bash
helm uninstall regional-hyperfleet-api -n argocd
```
