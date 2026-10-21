 # OpenShift TLS Certificate Management with Vault Secrets Operator (VSO)

  This repository provides an end-to-end guide, declarative manifests, and automation scripts for managing and auto-rotating **X.509 TLS Certificates** in **Red Ha
  t OpenShift** using **HashiCorp Vault Secrets Operator (`VaultPKISecret`)**.

  ---

  ## Table of Contents
  - [1. Architecture Overview](#1-architecture-overview)
  - [2. Prerequisites](#2-prerequisites)
  - [3. Repository Structure](#3-repository-structure)
  - [4. Step 1: Configure Vault PKI & Kubernetes Auth](#4-step-1-configure-vault-pki--kubernetes-auth)
  - [5. Step 2: Install Vault Secrets Operator on OpenShift](#5-step-2-install-vault-secrets-operator-on-openshift)
  - [6. Step 3: Deploy VSO Custom Resources](#6-step-3-deploy-vso-custom-resources)
    - [VaultConnection](#vaultconnection)
    - [VaultAuth & ServiceAccount](#vaultauth--serviceaccount)
    - [VaultPKISecret](#vaultpkisecret)
  - [7. Step 4: Deploy TLS-Enabled Workload & OpenShift Route](#7-step-4-deploy-tls-enabled-workload--openshift-route)
  - [8. Verification & Day-2 Operations](#8-verification--day-2-operations)
    - [Inspect TLS Certificate](#inspect-tls-certificate)
    - [Test Zero-Downtime Auto-Rotation](#test-zero-downtime-auto-rotation)
  - [9. Production Best Practices](#9-production-best-practices)

  ---

  ## 1. Architecture Overview
  ┌────────────────────────┐ ┌────────────────────────┐ ┌─────────────────────┐ ┌───────────────────────────┐ ┌────────────────────┐ ┌─────────────────┐
  │                        │ │                        │ │                     │ │                           │ │                    │ │                 │
  │                        │ │                        │ │                     │ │                           │ │                    │ │                 │
  │ ServiceAccount: app-sa │ │ Vault Secrets Operator │ │ CRD: VaultPKISecret │ │  Secret: my-app-tls-cert  │ │ Deployment: my-app │ │ OpenShift Route │
  │                        │ │                        │ │                     │ │ (type: kubernetes.io/tls) │ │                    │ │                 │
  │                        │ │                        │ │                     │ │                           │ │                    │ │                 │
  └────────────────────────┘ └────────────────────────┘ └─────────────────────┘ └───────────────────────────┘ └────────────────────┘ └─────────────────┘
  VPKI -->|Reconciles| VSO
      VSO -->|Mounts SA JWT token| SA
      VSO -->|Creates & Auto-Renews| K8S_SEC
      K8S_SEC -->|Mounted at /etc/tls| DEP
      DEP -->|Serves HTTPS| ROUTE
      VSO -.->|Triggers Rolling Restart on Renewal| DEP
  end
  subgraph Vault ["HashiCorp Vault"]
      PKI_ENG["PKI Secrets Engine (/pki)"]
      K8S_AUTH["Kubernetes Auth Engine (/auth/kubernetes)"]
      PKI_ROLE["PKI Role: app-tls-role"]

      K8S_AUTH -->|Validates SA Token with OpenShift API| OpenShift
      VSO -->|Authenticate| K8S_AUTH
      VSO -->|Issue / Rotate Certificate| PKI_ROLE
      PKI_ROLE --> PKI_ENG
  end


  ### Key Workflow:
  1. **Authentication**: OpenShift Pod / VSO uses `VaultAuth` with OpenShift's ServiceAccount JWT to authenticate against Vault's Kubernetes Auth backend.
  2. **Dynamic Issuance**: `VaultPKISecret` requests a short-lived TLS certificate from Vault's PKI engine.
  3. **Secret Generation**: VSO writes the certificate, private key, and CA chain into a native OpenShift `kubernetes.io/tls` secret (`my-app-tls-cert`).
  4. **Automated Lifecycle**: When the certificate reaches its renewal threshold (`expiryPageRatio: 70`), VSO requests a new certificate from Vault, updates the Op
  enShift Secret, and triggers a rolling restart (`rolloutRestartTargets`) of the application pods.

  ---

  ## 2. Prerequisites

  | Component | Minimum Version | Requirement |
  | :--- | :--- | :--- |
  | **Red Hat OpenShift** | 4.10+ | `cluster-admin` privileges |
  | **HashiCorp Vault** | 1.13+ | Reachable from OpenShift cluster |
  | **Vault Secrets Operator** | 0.4.0+ | Installed via OperatorHub / Helm |
  | **OpenShift CLI (`oc`)** | 4.10+ | Configured locally |
  | **Vault CLI (`vault`)** | 1.13+ | Admin access to Vault |

  ---

  ## 3. Repository Structure
  .
  ├── scripts/
  │   └── 01-vault-pki-setup.sh       # Automates Vault PKI & Auth configuration
  ├── manifests/
  │   ├── 02-vault-connection.yaml    # VaultConnection Custom Resource
  │   ├── 03-vault-auth.yaml          # ServiceAccount & VaultAuth CR
  │   ├── 04-vault-pki-secret.yaml    # VaultPKISecret CR with auto-renewal & pod rollout
  │   ├── 05-app-deployment.yaml      # Deployment & Service consuming TLS secret
  │   └── 06-openshift-route.yaml     # OpenShift Route with TLS Termination
  └── README.md                       # Documentation

  ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

  ## 4. Step 1: Configure Vault PKI & Kubernetes Auth

  Save this script as scripts/01-vault-pki-setup.sh and execute it against your Vault server:

  #!/usr/bin/env bash
  set -euo pipefail
  VAULT_ADDR="${VAULT_ADDR:-https://vault.example.com:8200}"
  OPENSHIFT_PROJECT="app-prod"
  OPENSHIFT_SA="app-sa"
  PKI_MOUNT="pki"
  PKI_ROLE="app-tls-role"
  CLUSTER_DOMAIN="apps.mycluster.example.com"
  echo "=== 1. Enable PKI Secrets Engine ==="
  vault secrets enable -path="${PKI_MOUNT}" pki || true
  vault secrets tune -max-lease-ttl=8760h "${PKI_MOUNT}"
  echo "=== 2. Generate Internal Root Certificate Authority ==="
  vault write -field=certificate "${PKI_MOUNT}/root/generate/internal"
      common_name="Enterprise OpenShift Root CA"
      ttl=8760h > root_ca.crt
  vault write "${PKI_MOUNT}/config/urls"
      issuing_certificates="${VAULT_ADDR}/v1/${PKI_MOUNT}/ca"
      crl_distribution_points="${VAULT_ADDR}/v1/${PKI_MOUNT}/crl"
  echo "=== 3. Create PKI Role for OpenShift Workloads ==="

  # Generates certificates with 24-hour TTL for demonstration of automated rotation

  vault write "${PKI_MOUNT}/roles/${PKI_ROLE}"
      allowed_domains="${CLUSTER_DOMAIN},svc.cluster.local,example.com"
      allow_subdomains=true
      max_ttl="720h"
      ttl="24h"
      require_cn=false
      generate_lease=true
  echo "=== 4. Create Vault Access Policy ==="
  cat <<EOF | vault policy write app-pki-policy -
  path "${PKI_MOUNT}/issue/${PKI_ROLE}" {
    capabilities = ["create", "update"]
  }
  path "${PKI_MOUNT}/cert/*" {
    capabilities = ["read"]
  }
  path "${PKI_MOUNT}/ca" {
    capabilities = ["read"]
  }
  path "${PKI_MOUNT}/ca_chain" {
    capabilities = ["read"]
  }
  EOF
  echo "=== 5. Configure Kubernetes Auth Backend in Vault ==="
  vault auth enable kubernetes || true

  K8S_HOST=$(oc whoami --show-server)
  vault write auth/kubernetes/config
      kubernetes_host="${K8S_HOST}"
  echo "=== 6. Bind Vault Policy to OpenShift ServiceAccount ==="
  vault write auth/kubernetes/role/app-pki-auth-role
      bound_service_account_names="${OPENSHIFT_SA}"
      bound_service_account_namespaces="${OPENSHIFT_PROJECT}"
      policies="app-pki-policy"
      ttl="1h"

  echo "=== Vault PKI Setup Completed Successfully! ==="
  ```
  Run the script:

  export VAULT_ADDR="https://vault.example.com:8200"
  export VAULT_TOKEN="<your-vault-token>"
  chmod +x scripts/01-vault-pki-setup.sh
  ./scripts/01-vault-pki-setup.sh

  ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

  ## 5. Step 2: Install Vault Secrets Operator on OpenShift

  Install VSO via Helm or OpenShift OperatorHub:
  helm repo add hashicorp https://helm.releases.hashicorp.com
  helm repo update
  helm install vault-secrets-operator hashicorp/vault-secrets-operator
      --namespace vault-secrets-operator-system
      --create-namespace


  ---

  ## 6. Step 3: Deploy VSO Custom Resources

  Create target OpenShift namespace:
  ```bash
  oc new-project app-prod
  ### VaultConnection

  manifests/02-vault-connection.yaml:

  apiVersion: secrets.hashicorp.com/v1beta1
  kind: VaultConnection
  metadata:
    name: default
    namespace: app-prod
  spec:
    address: "https://vault.example.com:8200"
  ### VaultAuth & ServiceAccount

  manifests/03-vault-auth.yaml:

  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: app-sa
    namespace: app-prod
  ---
  apiVersion: secrets.hashicorp.com/v1beta1
  kind: VaultAuth
  metadata:
    name: app-vault-auth
    namespace: app-prod
  spec:
    vaultConnectionRef: default
    method: kubernetes
    mount: kubernetes
    kubernetes:
      role: app-pki-auth-role
      serviceAccount: app-sa
  ### VaultPKISecret

  manifests/04-vault-pki-secret.yaml:

  apiVersion: secrets.hashicorp.com/v1beta1
  kind: VaultPKISecret
  metadata:
    name: my-app-tls-secret
    namespace: app-prod
  spec:
    vaultAuthRef: app-vault-auth
    mount: pki
    role: app-tls-role
    commonName: "my-app.apps.mycluster.example.com"
    altNames:
      - "my-app.app-prod.svc.cluster.local"
      - "my-app"
    format: pem
    ttl: 24h
    # Auto-rotate when certificate reaches 70% of lease TTL (~16.8 hours)
    expiryPageRatio: 70
    destination:
      create: true
      name: my-app-tls-cert
      type: kubernetes.io/tls
    # Automatically perform rolling restart of app pods on cert renewal
    rolloutRestartTargets:
      - kind: Deployment
        name: my-app
  Apply VSO resources:

  oc apply -f manifests/02-vault-connection.yaml
  oc apply -f manifests/03-vault-auth.yaml
  oc apply -f manifests/04-vault-pki-secret.yaml

  ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

  ## 7. Step 4: Deploy TLS-Enabled Workload & OpenShift Route
  ### Application Deployment & Service

  manifests/05-app-deployment.yaml:

  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: my-app
    namespace: app-prod
    labels:
      app: my-app
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: my-app
    template:
      metadata:
        labels:
          app: my-app
      spec:
        serviceAccountName: app-sa
        containers:
          - name: app
            image: registry.access.redhat.com/ubi9/ubi-minimal:latest
            command:
              - /bin/sh
              - -c
              - |
                echo "Starting TLS application..."
                ls -la /etc/tls/
                sleep infinity
            ports:
              - containerPort: 8443
                name: https
            volumeMounts:
              - name: tls-certs
                mountPath: /etc/tls
                readOnly: true
        volumes:
          - name: tls-certs
            secret:
              secretName: my-app-tls-cert
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: my-app-svc
    namespace: app-prod
  spec:
    selector:
      app: my-app
    ports:
      - name: https
        port: 8443
        targetPort: 8443
  ### OpenShift Route (Passthrough or Reencrypt)

  manifests/06-openshift-route.yaml:

  apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    name: my-app-route
    namespace: app-prod
  spec:
    host: my-app.apps.mycluster.example.com
    to:
      kind: Service
      name: my-app-svc
    port:
      targetPort: https
    tls:
      termination: passthrough
      insecureEdgeTerminationPolicy: Redirect
  Apply workload resources:

  oc apply -f manifests/05-app-deployment.yaml
  oc apply -f manifests/06-openshift-route.yaml

  ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

  ## 8. Verification & Day-2 Operations
  ### Inspect TLS Certificate


  1. Check the sync status of VaultPKISecret:

  oc get vaultpkisecret my-app-tls-secret -n app-prod
  2. Verify that the native OpenShift TLS secret was generated:

  oc get secret my-app-tls-cert -n app-prod
  3. Decode and inspect certificate validity dates, SANs, and Issuer:

  oc get secret my-app-tls-cert -n app-prod -o jsonpath='{.data.tls\.crt}' \
      | base64 -d \
      | openssl x509 -text -noout \
      | grep -E "(Issuer:|Subject:|Not Before|Not After|DNS:)"
  ### Test Zero-Downtime Auto-Rotation

  To verify that VSO rotates the certificate and reloads the deployment:


  1. Revoke the active lease in Vault or manually trigger renewal.
  2. Watch the rollout status of the Deployment:

  oc rollout status deployment/my-app -n app-prod -w
  3. Verify that the pod mounts the newly issued certificate without manual intervention.

  ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

  ## 9. Production Best Practices
  • Security Context Constraints (SCC): VSO runs seamlessly under OpenShift's standard restricted-v2 SCC with non-root ServiceAccounts.
  • Short-Lived Certificates: Use TTLs between 24h and 7 days. Short TTLs mitigate the blast radius of compromised keys and eliminate manual certificate lifecycle
  overhead.
  • Intermediate CAs: In production, do not generate Root CAs directly inside Vault. Mount an Intermediate CA signed by your Enterprise Offline Root CA into Vault
  PKI.
  • Custom Enterprise CA: If Vault uses an internal enterprise CA certificate, configure spec.caCertSecretRef inside VaultConnection referencing an OpenShift Confi
  gMap containing the CA bundle
