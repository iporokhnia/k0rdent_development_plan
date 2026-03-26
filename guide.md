# k0rdent Enterprise 1.2.2 Installation Guide

**Scope:** Management cluster + Regional cluster (OpenStack) + Child cluster (OpenStack) + Grafana with metrics.

---

## Architecture Overview

```
Management Cluster (single-node k0s)
├── k0rdent Enterprise 1.2.2 (namespace: kcm-system)
├── Istio control plane (namespace: istio-system)
│   ├── k0rdent-istio-base 0.1.0
│   └── k0rdent-istio 0.1.0 (istiod + east-west gateway + operator)
├── KOF 1.5.0 — Observability & FinOps (namespace: kof)
│   ├── kof-operators (grafana-operator, otel-operator, vm-operator)
│   ├── kof-mothership (promxy, VMCluster, alertmanager)
│   └── Grafana (grafana-operator CR)
│
├── Regional Cluster (OpenStack Standalone, 1 CP + 3 Workers)
│   ├── CAPI providers (deployed by Region object)
│   ├── Istio member (east-west gateway)
│   ├── KOF Storage (VMCluster, VictoriaLogs)
│   └── KOF Collectors (daemon + cluster-stats)
│       │
│       └── Child Cluster (OpenStack Standalone, 1 CP + 1 Worker)
│           ├── Istio member
│           └── KOF Collectors (metrics/logs → Regional)
```

**Data flow:** Child collectors → Regional storage → Management promxy → Grafana.

---

## Prerequisites

### Infrastructure

- **Management node:** A Linux VM with a single-node Kubernetes cluster (k0s recommended). Minimum 8 vCPU, 16 GB RAM, 160 GB disk.
- **OpenStack project** with API access, sufficient quota for:
  - Regional: 1 CP (4 vCPU / 8 GB) + 3 Workers (4 vCPU / 8 GB each)
  - Child: 1 CP (2 vCPU / 4 GB) + 1 Worker (4 vCPU / 8 GB)
  - Floating IPs, networks, routers will be created automatically by CAPO
- **SSH keypair** created in OpenStack (referenced in ClusterDeployment configs)
- **External network** (floating IP pool) available in OpenStack
- **Ubuntu 22.04 image** uploaded to OpenStack (name: `ubuntu-22.04-x86_64`)

### Credentials

- **Mirantis registry** credentials for `registry.mirantis.com` (username + password)
- **OpenStack** credentials: `clouds.yaml` with username/password or application credentials

### Tools on Management Node

- `kubectl` v1.31+ configured to management cluster
- `helm` v3.16.3+
- `base64`, `jq`

### DNS (Important for Isolated Environments)

If OpenStack environment uses internal DNS, you **must** use the internal DNS server in `dnsNameservers` fields of ClusterDeployments. Using public DNS (e.g., `8.8.8.8`) will cause resolution failures for internal endpoints, breaking the OpenStack Cloud Controller Manager (OCCM) on managed clusters.

---

## Placeholder Reference

The following placeholders are used throughout this guide. Replace them with your actual values.

| Placeholder | Description | Example |
|---|---|---|
| `<REGISTRY_USER>` | Mirantis registry username | `myuser` |
| `<REGISTRY_PASSWORD>` | Mirantis registry password | `mypassword` |
| `<KEYSTONE_URL>` | OpenStack Keystone endpoint (with `/v3`) | `https://keystone.example.com/v3` |
| `<OS_USERNAME>` | OpenStack username | `admin` |
| `<OS_PASSWORD>` | OpenStack password | `secret` |
| `<OS_PROJECT_ID>` | OpenStack project ID | `a91e8c57...` |
| `<OS_PROJECT_NAME>` | OpenStack project name | `my-project` |
| `<OS_USER_DOMAIN>` | OpenStack user domain name | `Default` |
| `<SSH_KEY_NAME>` | OpenStack SSH keypair name | `my-keypair` |
| `<DNS_SERVER>` | DNS server reachable from CAPO-provisioned nodes (could be ommited in public clouds) | `172.18.176.4` |
| `<EXTERNAL_NETWORK>` | OpenStack external network name (FIP pool) | `public` |
| `<MANAGEMENT_IP>` | Management node IP (or floating IP) | `172.19.116.157` |
| `<REGIONAL_NAME>` | Name for regional ClusterDeployment (≤15 chars) | `my-regional` |
| `<CHILD_NAME>` | Name for child ClusterDeployment | `my-child` |

---

## Step 1 — Initial Setup

### 1.1 Helm Registry Login

```bash
helm registry login registry.mirantis.com \
  --username <REGISTRY_USER> \
  --password <REGISTRY_PASSWORD>
```

### 1.2 Create Namespaces and Registry Credentials

```bash
kubectl create namespace kcm-system
kubectl create namespace kof
kubectl create namespace istio-system
```

Create the registry pull secret in all three namespaces:

```bash
for NS in kcm-system kof istio-system; do
  kubectl create secret docker-registry registry-creds \
    --docker-server=registry.mirantis.com \
    --docker-username=<REGISTRY_USER> \
    --docker-password=<REGISTRY_PASSWORD> \
    -n $NS
done
```

---

## Step 2 — Install k0rdent Enterprise

> **Important:** k0rdent Enterprise requires a two-step installation. The first install (bootstrap) runs without the admission webhook to allow initial CRD bootstrapping. The second step (upgrade) enables the full configuration including the webhook and cluster-api-operator.

### 2.1 Bootstrap Install

Create file `kcm-values-bootstrap.yaml`:

```yaml
admissionWebhook:
  enabled: false
controller:
  createAccessManagement: false
  createManagement: true
  createRelease: true
  registryCredsSecret: registry-creds
global:
  registry: registry.mirantis.com/k0rdent-enterprise
imagePullSecrets:
- name: registry-creds
k0rdent-ui:
  service:
    nodePort: 30300
    type: NodePort
regional:
  telemetry: {}
  velero:
    enabled: true
```

```bash
helm install kcm \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/k0rdent-enterprise \
  --version 1.2.2 \
  -n kcm-system \
  -f kcm-values-bootstrap.yaml \
  --timeout 10m
```

> **Critical:** `controller.createManagement: true` and `controller.createRelease: true` are required. Without them, the Management object and ClusterTemplates will not be created.

Wait for the controller to become ready:

```bash
kubectl get pods -n kcm-system
kubectl get management
```

Expect: `Management` object exists, controller pod is Running.

### 2.2 Upgrade to Full Configuration

Create file `kcm-values.yaml`:

```yaml
admissionWebhook:
  enabled: true
controller:
  createAccessManagement: false
  createManagement: true
  createRelease: true
  registryCredsSecret: registry-creds
global:
  registry: registry.mirantis.com/k0rdent-enterprise
imagePullSecrets:
- name: registry-creds
k0rdent-ui:
  service:
    nodePort: 30300
    type: NodePort
regional:
  cluster-api-operator:
    enabled: true
  telemetry: {}
  velero:
    enabled: true
```

```bash
helm upgrade kcm \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/k0rdent-enterprise \
  --version 1.2.2 \
  -n kcm-system \
  -f kcm-values.yaml \
  --timeout 10m
```

Verify:

```bash
kubectl get management
kubectl get clustertemplates
kubectl get pods -n kcm-system
```

Expected: Management object READY, ClusterTemplates available (including `openstack-standalone-cp-1-0-21`), all pods Running.

---

## Step 3 — Install Istio

> **Order is critical:** `k0rdent-istio-base` MUST be installed BEFORE `k0rdent-istio`.

### 3.0 Create Shared Registry Values File

Create file `global-values.yaml`. This file redirects all image pulls and chart fetches to the Mirantis enterprise registry. It is reused across Istio and KOF installations.

```yaml
global:
  registry: registry.mirantis.com/k0rdent-enterprise
  imageRegistry: registry.mirantis.com/k0rdent-enterprise
  image:
    registry: registry.mirantis.com/k0rdent-enterprise
  hub: registry.mirantis.com/k0rdent-enterprise/istio
  helmChartsRepo: oci://registry.mirantis.com/k0rdent-enterprise/charts
cert-manager-istio-csr:
  image:
    repository: registry.mirantis.com/k0rdent-enterprise/jetstack/cert-manager-istio-csr
cert-manager-service-template:
  skipVerifyJob: true
  repo:
    type: oci
    url: oci://registry.mirantis.com/k0rdent-enterprise/charts
cluster-api-visualizer:
  image:
    repository: registry.mirantis.com/k0rdent-enterprise
    tag: 1.4.1
external-dns:
  image:
    repository: registry.mirantis.com/k0rdent-enterprise/external-dns/external-dns
grafana-operator:
  image:
    repository: registry.mirantis.com/k0rdent-enterprise/grafana/grafana-operator
ingress-nginx-service-template:
  skipVerifyJob: true
  repo:
    type: oci
    url: oci://registry.mirantis.com/k0rdent-enterprise/charts
jaeger-operator:
  image:
    repository: registry.mirantis.com/k0rdent-enterprise/jaegertracing/jaeger-operator
kcm:
  kof:
    repo:
      spec:
        url: oci://registry.mirantis.com/k0rdent-enterprise/charts
k0rdent-istio:
  repo:
    spec:
      url: oci://registry.mirantis.com/k0rdent-enterprise/charts
opencost:
  opencost:
    exporter:
      image:
        registry: registry.mirantis.com/k0rdent-enterprise
    ui:
      image:
        registry: registry.mirantis.com/k0rdent-enterprise
opentelemetry-operator:
  manager:
    image:
      repository: registry.mirantis.com/k0rdent-enterprise/opentelemetry-operator/opentelemetry-operator
    collectorImage:
      repository: registry.mirantis.com/k0rdent-enterprise/otel/opentelemetry-collector-contrib
  kubeRBACProxy:
    image:
      repository: registry.mirantis.com/k0rdent-enterprise/brancz/kube-rbac-proxy
opentelemetry-kube-stack:
  collectors:
    daemon:
      image:
        repository: registry.mirantis.com/k0rdent-enterprise/kof/kof-opentelemetry-collector-contrib
```

### 3.1 Install k0rdent-istio-base

Create file `istio-base-values.yaml`:

```yaml
injectionNamespaces:
- kof
cert-manager-service-template:
  enabled: false
```

```bash
helm install k0rdent-istio-base \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/k0rdent-istio-base \
  --version 0.1.0 \
  -n istio-system \
  -f global-values.yaml \
  -f istio-base-values.yaml \
  --timeout 10m
```

### 3.2 Install k0rdent-istio

Create file `istio-values.yaml`:

```yaml
cert-manager:
  enabled: false
cert-manager-service-template:
  enabled: false
istiod:
  meshConfig:
    extensionProviders:
    - name: otel-tracing
      opentelemetry:
        port: 4317
        service: kof-collectors-daemon-collector.kof.svc.cluster.local
```

```bash
helm install k0rdent-istio \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/k0rdent-istio \
  --version 0.1.0 \
  -n istio-system \
  -f global-values.yaml \
  -f istio-values.yaml \
  --timeout 10m
```

Verify:

```bash
kubectl get pods -n istio-system
kubectl get servicetemplates -n kcm-system
```

Expected: istiod Running, ServiceTemplates for istio created in kcm-system.

---

## Step 4 — Install KOF

### 4.1 Install kof-operators

Create file `kof-operators-values.yaml`:

```yaml
grafana-operator:
  enabled: true
```

> **Note:** `grafana-operator.enabled` is `false` by default in kof-operators 1.5.0. You must explicitly enable it.

```bash
helm install kof-operators \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/kof-operators \
  --version 1.5.0 \
  -n kof \
  -f global-values.yaml \
  -f kof-operators-values.yaml \
  --timeout 10m
```

### 4.2 Install kof-mothership

Create file `kof-mothership-values.yaml`:

```yaml
grafana:
  enabled: true
kcm:
  installTemplates: true
victoriametrics:
  vmcluster:
    spec:
      retentionPeriod: "7"
```

> **Note:** `kcm.installTemplates: true` is critical — it creates ServiceTemplates that KOF uses to deploy collectors and storage on managed clusters.

```bash
helm install kof-mothership \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/kof-mothership \
  --version 1.5.0 \
  -n kof \
  -f global-values.yaml \
  -f kof-mothership-values.yaml \
  --timeout 10m
```

Verify:

```bash
kubectl get pods -n kof
kubectl get servicetemplates -n kcm-system
```

Expected: ~11 pods in kof namespace (vm-operator, promxy, vmstorage, vminsert, vmselect, alertmanager, grafana-operator, etc.). ServiceTemplates for kof-regional, kof-child, kof-storage, kof-collectors created.

### 4.3 Install kof-regional and kof-child ServiceTemplates

> **Important:** These charts use `--set-file globalValues=` (NOT `-f`). This is a different mechanism — the file content is passed as a Helm value, not as a values file.

```bash
helm install kof-regional \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/kof-regional \
  --version 1.5.0 \
  -n kof \
  --set-file globalValues=global-values.yaml
```

```bash
helm install kof-child \
  oci://registry.mirantis.com/k0rdent-enterprise/charts/kof-child \
  --version 1.5.0 \
  -n kof \
  --set-file globalValues=global-values.yaml
```

---

## Step 5 — OpenStack Credentials

### 5.1 Create the clouds.yaml Secret

> **Critical:** The secret **must** be named `openstack-cloud-config`. This name is hardcoded in the k0rdent codebase for Sveltos credential propagation. Using any other name will prevent OCCM from starting on managed clusters.

Create file `openstack-cloud-config.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openstack-cloud-config
  namespace: kcm-system
type: Opaque
stringData:
  clouds.yaml: |
    clouds:
      openstack:
        auth:
          auth_url: <KEYSTONE_URL>
          username: <OS_USERNAME>
          password: <OS_PASSWORD>
          project_id: <OS_PROJECT_ID>
          project_name: <OS_PROJECT_NAME>
          user_domain_name: <OS_USER_DOMAIN>
          project_domain_id: default
        region_name: RegionOne
        interface: public
        identity_api_version: 3
```

```bash
kubectl apply -f openstack-cloud-config.yaml
```

### 5.2 Create Credential (for management → regional)

Create file `openstack-credential.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1beta1
kind: Credential
metadata:
  name: os-credential
  namespace: kcm-system
spec:
  description: "OpenStack credential"
  identityRef:
    apiVersion: v1
    kind: Secret
    name: openstack-cloud-config
    namespace: kcm-system
```

```bash
kubectl apply -f openstack-credential.yaml
```

Verify:

```bash
kubectl get credential -n kcm-system
```

Expected: `os-credential` with READY=true.

### 5.3 Create CCM Resource Template ConfigMap

This ConfigMap is a Sveltos template that renders the `cloud.conf` Secret on managed clusters. It is required for the OpenStack Cloud Controller Manager (OCCM) to initialize nodes. Without it, worker nodes remain tainted and no workloads can be scheduled.

> **Note:** This ConfigMap is applied on the **management** cluster. Sveltos reads it from management and renders the result on target clusters.

The ConfigMap name **must** follow the pattern `{secret-name}-resource-template` — in our case `openstack-cloud-config-resource-template`.

Create file `openstack-cloud-config-resource-template.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: openstack-cloud-config-resource-template
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: kcm
  annotations:
    projectsveltos.io/template: "true"
data:
  configmap.yaml: |
    {{- $cluster := .InfrastructureProvider -}}
    {{- $identity := (getResource "InfrastructureProviderIdentity") -}}
    {{- $clouds := fromYaml (index $identity "data" "clouds.yaml" | b64dec) -}}
    {{- if not $clouds }}
      {{ fail "failed to decode clouds.yaml" }}
    {{ end -}}
    {{- $openstack := index $clouds "clouds" "openstack" -}}
    {{- if not (hasKey $openstack "auth") }}
      {{ fail "auth key not found in openstack config" }}
    {{- end }}
    {{- $auth := index $openstack "auth" -}}
    {{- $auth_url := index $auth "auth_url" -}}
    {{- $app_cred_id := index $auth "application_credential_id" -}}
    {{- $app_cred_name := index $auth "application_credential_name" -}}
    {{- $app_cred_secret := index $auth "application_credential_secret" -}}
    {{- $network_id := $cluster.status.externalNetwork.id -}}
    {{- $network_name := $cluster.status.externalNetwork.name -}}
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: openstack-cloud-config
      namespace: kube-system
    type: Opaque
    stringData:
      cloud.conf: |
        [Global]
        auth-url="{{ $auth_url }}"
        {{- if $app_cred_id }}
        application-credential-id="{{ $app_cred_id }}"
        {{- end }}
        {{- if $app_cred_name }}
        application-credential-name="{{ $app_cred_name }}"
        {{- end }}
        {{- if $app_cred_secret }}
        application-credential-secret="{{ $app_cred_secret }}"
        {{- end }}
        {{- if and (not $app_cred_id) (not $app_cred_secret) }}
        username="{{ index $auth "username" }}"
        password="{{ index $auth "password" }}"
        tenant-name="{{ index $auth "project_name" }}"
        user-domain-name="{{ index $auth "user_domain_name" }}"
        {{- if index $auth "project_id" }}
        tenant-id="{{ index $auth "project_id" }}"
        {{- end }}
        {{- end }}
        region="{{ index $openstack "region_name" }}"

        [LoadBalancer]
        {{- if $network_id }}
        floating-network-id="{{ $network_id }}"
        {{- end }}

        [Networking]
        {{- if $network_name }}
        public-network-name="{{ $network_name }}"
        {{- end }}
```

```bash
kubectl apply -f openstack-cloud-config-resource-template.yaml
```

---

## Step 6 — Deploy Regional Cluster

Create file `regional-cd.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterDeployment
metadata:
  name: <REGIONAL_NAME>
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/istio-gateway: "true"
    k0rdent.mirantis.com/istio-mesh: region-openstack
    k0rdent.mirantis.com/istio-role: member
    k0rdent.mirantis.com/kcm-region-cluster: "true"
    k0rdent.mirantis.com/kof-cluster-role: regional
spec:
  template: openstack-standalone-cp-1-0-21
  credential: os-credential
  config:
    controlPlaneNumber: 1
    workersNumber: 3
    controlPlane:
      flavor: m1.large
      image:
        filter:
          name: ubuntu-22.04-x86_64
      sshKeyName: <SSH_KEY_NAME>
    worker:
      flavor: m1.large
      image:
        filter:
          name: ubuntu-22.04-x86_64
      sshKeyName: <SSH_KEY_NAME>
    identityRef:
      cloudName: openstack
      region: RegionOne
    externalNetwork:
      filter:
        name: <EXTERNAL_NETWORK>
    managedSubnets:
    - cidr: 10.9.10.0/24
      dnsNameservers:
      - <DNS_SERVER>
```

```bash
kubectl apply -f regional-cd.yaml
```

Wait for the cluster to become Ready (typically 10-15 minutes):

```bash
kubectl get clusterdeployment <REGIONAL_NAME> -n kcm-system -w
```

Once Ready, extract the kubeconfig:

```bash
kubectl get secret <REGIONAL_NAME>-kubeconfig -n kcm-system \
  -o jsonpath='{.data.value}' | base64 -d > /tmp/regional-kubeconfig.yaml
```

Verify access to regional cluster:

```bash
kubectl --kubeconfig /tmp/regional-kubeconfig.yaml get nodes
```

---

## Step 7 — Create Region

The Region object tells k0rdent to install CAPI providers and kcm-regional on the regional cluster.

Create file `region.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1beta1
kind: Region
metadata:
  name: region-openstack
spec:
  kubeConfig:
    name: <REGIONAL_NAME>-kubeconfig
    key: value
  providers:
  - name: cluster-api-provider-k0sproject-k0smotron
  - name: cluster-api-provider-openstack
  - name: projectsveltos
  core:
    kcm:
      config:
        cert-manager:
          enabled: false
```

> **Note:** `cert-manager.enabled: false` prevents conflicts with the cert-manager deployed by Istio MCS on the regional cluster.

```bash
kubectl apply -f region.yaml
```

Wait for Region to become Ready. This takes time as it deploys CAPI providers and kcm-regional:

```bash
kubectl get region region-openstack -w
```

---

## Step 8 — Create Regional Credential and Deploy Child Cluster

### 8.1 Create Regional Credential

The regional Credential has `spec.region` pointing to the Region object. This tells k0rdent to deploy the ClusterDeployment through the regional cluster instead of management.

> **Note:** `spec.region` is immutable after creation. If you need to change it, delete and recreate the Credential.

Create file `os-regional-credential.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1beta1
kind: Credential
metadata:
  name: os-regional-credential
  namespace: kcm-system
spec:
  description: "OpenStack credential (regional)"
  identityRef:
    apiVersion: v1
    kind: Secret
    name: openstack-cloud-config
    namespace: kcm-system
  region: region-openstack
```

```bash
kubectl apply -f os-regional-credential.yaml
```

### 8.2 Deploy Child Cluster

Create file `child-cd.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterDeployment
metadata:
  name: <CHILD_NAME>
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/istio-mesh: region-openstack
    k0rdent.mirantis.com/istio-role: member
    k0rdent.mirantis.com/kof-cluster-role: child
    k0rdent.mirantis.com/kof-regional-cluster-name: <REGIONAL_NAME>
spec:
  template: openstack-standalone-cp-1-0-21
  credential: os-regional-credential
  config:
    controlPlaneNumber: 1
    workersNumber: 1
    controlPlane:
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
      sshKeyName: <SSH_KEY_NAME>
    worker:
      flavor: m1.large
      image:
        filter:
          name: ubuntu-22.04-x86_64
      sshKeyName: <SSH_KEY_NAME>
    identityRef:
      cloudName: openstack
      region: RegionOne
    externalNetwork:
      filter:
        name: <EXTERNAL_NETWORK>
    managedSubnets:
    - cidr: 10.9.11.0/24
      dnsNameservers:
      - <DNS_SERVER>
```

> **Critical:** The label `k0rdent.mirantis.com/kof-regional-cluster-name: <REGIONAL_NAME>` is **required** in KOF 1.5.0. Without it, the KOF operator cannot discover the regional cluster and collectors will not be configured.

```bash
kubectl apply -f child-cd.yaml
```

Wait for Ready:

```bash
kubectl get clusterdeployment <CHILD_NAME> -n kcm-system -w
```

---

## Step 9 — Grafana

### 9.1 Deploy Grafana CR

Create file `grafana.yaml`:

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: Grafana
metadata:
  name: grafana-vm
  namespace: kof
  labels:
    dashboards: grafana
spec:
  version: 11.3.0
  persistentVolumeClaim:
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 200Mi
      storageClassName: local-path
  service:
    spec:
      type: ClusterIP
      ports:
      - name: grafana
        port: 3000
        targetPort: 3000
        protocol: TCP
```

```bash
kubectl apply -f grafana.yaml
```

### 9.2 Access Grafana

The simplest access method is `kubectl port-forward`:

```bash
kubectl port-forward -n kof svc/grafana-vm-service 3000:3000
```

Then open `http://localhost:3000` in your browser.

Get admin credentials (auto-generated by grafana-operator):

```bash
kubectl get secret grafana-vm-admin-credentials -n kof \
  -o jsonpath='{.data.GF_SECURITY_ADMIN_PASSWORD}' | base64 -d && echo
```

Username: `admin`

> **Other access methods:** NodePort or Ingress can also be configured, but require additional tuning of the Grafana CR service spec and potentially Istio/ingress configuration, which is outside the scope of this guide.

### 9.3 Verify Metrics

Check that promxy can see metrics from managed clusters:

```bash
kubectl exec -n kof deploy/kof-mothership-promxy -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result[].metric.cluster'
```

Expected output should include both `<REGIONAL_NAME>` and `<CHILD_NAME>`.

In Grafana, the pre-configured dashboards should show metrics from all clusters through the `promxy` datasource.

---

## Verification Checklist

Run these commands to verify the complete setup:

```bash
# Management components
kubectl get management
kubectl get pods -n kcm-system
kubectl get pods -n istio-system
kubectl get pods -n kof

# Cluster deployments
kubectl get clusterdeployment -n kcm-system

# Region
kubectl get region

# Regional cluster
kubectl --kubeconfig /tmp/regional-kubeconfig.yaml get nodes
kubectl --kubeconfig /tmp/regional-kubeconfig.yaml get pods -n kof

# Metrics
kubectl exec -n kof deploy/kof-mothership-promxy -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result | length'
```

---

## Known Issues

| Issue | Description | Workaround |
|---|---|---|
| **Istio HelmRepository type** | k0rdent-istio-base creates HelmRepository `istio` without `type: oci`, causing Flux to reject OCI URLs | `kubectl patch helmrepo -n kcm-system istio --type=merge -p '{"spec":{"type":"oci"}}'` |
| **kof-mothership partial failure** | 2 ServiceTemplates may fail with missing `spec.resources.path`. Other resources deploy correctly. | Non-blocking. Core functionality works. |
| **daemon-collector OOMKilled** | Default memory limit (500Mi) insufficient for high pod count | Add to `global-values.yaml`: `opentelemetry-kube-stack.collectors.daemon.resources.limits.memory: 1Gi` |
| **Grafana dashboards empty via promxy** | Grafana 11.3.0 may show empty data through promxy datasource despite API returning data | Use VictoriaMetrics datasource directly, or verify with CLI: `kubectl exec deploy/kof-mothership-promxy -- wget -qO- 'http://localhost:9090/api/v1/query?query=up'` |
| **Controller delete-loop** | If k0rdent controller pod survives from a previous failed installation, it may immediately delete the Management object after helm install | Full teardown: `helm uninstall` + `kubectl delete namespace kcm-system` (kills the controller pod) → then fresh install |

---

## File Checklist

Files to create during installation:

| File | Step | Description |
|---|---|---|
| `kcm-values-bootstrap.yaml` | 2.1 | k0rdent Enterprise bootstrap values (no webhook) |
| `kcm-values.yaml` | 2.2 | k0rdent Enterprise full values |
| `global-values.yaml` | 3.0 | Shared registry redirect values |
| `istio-base-values.yaml` | 3.1 | k0rdent-istio-base values |
| `istio-values.yaml` | 3.2 | k0rdent-istio values |
| `kof-operators-values.yaml` | 4.1 | kof-operators values |
| `kof-mothership-values.yaml` | 4.2 | kof-mothership values |
| `openstack-cloud-config.yaml` | 5.1 | OpenStack clouds.yaml Secret |
| `openstack-cloud-config-resource-template.yaml` | 5.3 | Sveltos template for CCM cloud.conf |
| `openstack-credential.yaml` | 5.2 | Credential (management → regional) |
| `regional-cd.yaml` | 6 | Regional ClusterDeployment |
| `region.yaml` | 7 | Region object |
| `os-regional-credential.yaml` | 8.1 | Credential (regional, for child clusters) |
| `child-cd.yaml` | 8.2 | Child ClusterDeployment |
| `grafana.yaml` | 9.1 | Grafana CR |
