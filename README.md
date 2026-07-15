# Internal Developer Platform (IDP) Core

This repository contains the core configuration and platform definitions for the Internal Developer Platform (IDP) lab. It leverages **Crossplane (v2.x)** for infrastructure orchestration and custom resource provisioning, **Argo CD** for GitOps application deployment, and **Helm** for templating team applications.

---

## Repository Structure

```text
├── bootstrap/
│   └── argocd-apps/
│       └── alpha-frontend.yaml      # Argo CD application configuration for the Alpha team
├── charts/
│   └── platform-application/        # Generic Helm chart for deploying developer applications
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── ingress.yaml
│           ├── service.yaml
│           └── crossplane-function.yaml
├── crossplane/
│   └── apis/
│       ├── definition.yaml          # CompositeResourceDefinition (XRD) for DatabaseEnvironment
│       ├── composition.yaml         # Composition implementing the DatabaseEnvironment logic
│       └── function.yaml            # function-patch-and-transform declaration
├── providers/
│   ├── crossplane-provider.yaml     # provider-kubernetes installation manifest
│   └── crossplane-provider-config.yaml # ProviderConfig using InjectedIdentity authentication
└── examples/
    └── developer-db-request.yaml    # Example DatabaseEnvironment claim for application teams
```

---

## Architecture Overview

1. **Control Plane (Crossplane)**:
   - Defines a self-service abstraction: `DatabaseEnvironment` (API Group: `platform.example.com`).
   - The platform engineers manage the **CompositeResourceDefinition (XRD)** and the **Composition**.
   - When a developer requests a `DatabaseEnvironment` claim, Crossplane automatically provisions:
     - A dedicated Kubernetes Namespace (`db-env-<teamName>`).
     - A ConfigMap (`db-config`) inside that namespace containing database host details and size credentials.

2. **GitOps (Argo CD)**:
   - Automatically synchronizes applications using the templates under `charts/platform-application` to deploy deployments, services, and ingress rules.

---

## Installation & Bootstrapping

Follow these steps to bootstrap the platform in a Kubernetes cluster:

### Step 1: Install Crossplane Providers and Functions
Apply the provider and function definitions to the cluster:
```bash
# Install the Kubernetes provider
kubectl apply -f providers/crossplane-provider.yaml

# Install the patch-and-transform composition function
kubectl apply -f crossplane/apis/function.yaml
```

### Step 2: Configure Provider Authentication
Configure the provider to authenticate against the local cluster API server using its pod's service account identity (`InjectedIdentity`):
```bash
# Apply the ProviderConfig
kubectl apply -f providers/crossplane-provider-config.yaml

# Grant cluster-admin permissions to the provider's service account (required to manage namespaces/configmaps)
SA_NAME=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | cut -d'/' -f2)
kubectl create clusterrolebinding provider-kubernetes-admin-binding \
  --clusterrole cluster-admin \
  --serviceaccount=crossplane-system:${SA_NAME}
```

### Step 3: Apply the platform APIs
Apply the CompositeResourceDefinition (XRD) and the corresponding Composition:
```bash
# Establish the XRD (DatabaseEnvironment)
kubectl apply -f crossplane/apis/definition.yaml

# Apply the Composition (rules to provision Namespace & ConfigMap)
kubectl apply -f crossplane/apis/composition.yaml
```

---

## How to Use (Self-Service Database Environment)

Developers request a database environment by applying a claim.

### 1. Create a Claim
An example claim is provided in `examples/developer-db-request.yaml`:
```yaml
apiVersion: platform.example.com/v1alpha1
kind: DatabaseEnvironment
metadata:
  name: alpha-db-env
  namespace: default
spec:
  parameters:
    teamName: alpha
    dbSize: small
```

Apply the claim:
```bash
kubectl apply -f examples/developer-db-request.yaml
```

### 2. Verify Provisioning
Crossplane will bind the claim, select the composition, and provision the namespace and configmap:

```bash
# Check the status of the DatabaseEnvironment claim
kubectl get databaseenvironment

# Check the status of the generated Composite Resource (XR)
kubectl get composite

# Check the provisioned namespace
kubectl get ns db-env-alpha

# Check the generated ConfigMap containing credentials
kubectl get configmap -n db-env-alpha db-config -o yaml
```

---

## Author

- **Roberto Palacios** - [LinkedIn](https://www.linkedin.com/in/robpalacios1/)