# Monitoring Stack

This Helm chart deploys a lab-oriented observability stack for Kubernetes:
- Grafana for dashboards and visualization
- Mimir for metrics storage
- Loki for logs
- Tempo for distributed traces
- Alloy as a node-level telemetry collector
- MinIO as S3-compatible storage for Mimir, Loki, and Tempo
- CloudNativePG for Grafana's PostgreSQL database

The chart uses Nginx Ingress and Longhorn storage. It creates a CloudNativePG
`Cluster` resource, but it does not install the CloudNativePG operator.

## Prerequisites
The target cluster must already provide:

- Helm and Kubernetes access
- An Nginx Ingress Controller
- A `longhorn` StorageClass, or matching storage class overrides
- The CloudNativePG operator and CRDs

For local access, resolve the configured hosts to the Ingress address:
```text
grafana.pcs.ui
minio.pcs.ui
```

The default credentials in `values.yaml` are for local or lab use only. Override
all passwords and access keys before using this chart in a shared environment.

## Architecture

```mermaid
flowchart LR
  Client[Applications] --> Alloy[Alloy DaemonSet]
  Alloy --> Mimir[Mimir metrics]
  Alloy --> Loki[Loki logs]
  Alloy --> Tempo[Tempo traces]
  Mimir --> MinIO[MinIO S3]
  Loki --> MinIO
  Tempo --> MinIO
  Browser[Browser] --> Nginx[Nginx Ingress]
  Nginx --> Grafana[Grafana]
  Grafana --> Mimir
  Grafana --> Loki
  Grafana --> Tempo
  Grafana --> CNPG[CloudNativePG PostgreSQL]
```

## Components and defaults

| Component | Kubernetes resource | Default replicas | Internal ports |
| --- | --- | ---: | --- |
| Grafana | Deployment | 1 | `3000` |
| Mimir | StatefulSet | 2 | `9009` |
| Loki | StatefulSet | 3 | `3100`, `9095` |
| Tempo | StatefulSet | 2 | `3200`, `4317`, `4318` |
| Alloy | DaemonSet | One per node | `12345`, `4317`, `4318` |
| MinIO | StatefulSet | 1 | `9000`, `9001` |
| PostgreSQL | CloudNativePG Cluster | 3 | `5432` |

All component service names are release-scoped:

```text
<release>-grafana
<release>-mimir
<release>-loki
<release>-tempo
<release>-alloy
<release>-minio
```

Grafana provisions these datasources automatically:

- Prometheus-compatible Mimir at `<release>-mimir:<port>/prometheus`
- Loki at `<release>-loki:<port>`
- Tempo at `<release>-tempo:<port>`

Alloy collects node metrics and annotated pod metrics, tails Kubernetes pod
logs, and accepts OTLP over gRPC (`4317`) and HTTP (`4318`). It forwards those
signals to Mimir, Loki, and Tempo respectively.

## Ingress

Grafana is available at `grafana.pcs.ui`. MinIO's console is available at
`minio.pcs.ui` when the MinIO ingress is enabled.

The Loki, Mimir, Tempo, and Alloy services are ClusterIP services and are not
exposed through Ingress by default.

## Storage

The default `longhorn` storage class provisions `5Gi` for MinIO, Loki, Mimir,
Tempo, and each PostgreSQL instance. Adjust the component-specific values in
`values.yaml` for a different cluster or workload.

MinIO is configured as a single replica and is intended for development or lab
use, not highly available production object storage.

## Install and validate

Render and lint the chart before installation:

```bash
helm dependency update .
helm lint . --strict
helm template monitoring-stack . --namespace monitoring > rendered.yaml
```

Install or upgrade it with:

```bash
helm upgrade --install monitoring-stack . \
  --namespace monitoring \
  --create-namespace
```

Useful checks after installation:

```bash
kubectl get pods,svc,ingress -n monitoring
kubectl get statefulsets,daemonsets,deployments -n monitoring
kubectl get pvc,jobs -n monitoring
kubectl get cluster.postgresql.cnpg.io -n monitoring
```

The MinIO post-install hook creates the Loki and Mimir buckets. Ensure the Job
completes before troubleshooting object-storage-backed components.

## Configuration

The main configuration sections in `values.yaml` are:

- `minio`: image, credentials, persistence, resources, and ingress
- `loki`: replicas, S3 bucket, persistence, and resources
- `mimir`: replicas, S3 bucket, replication, persistence, and resources
- `tempo`: replicas, retention, S3 bucket, persistence, and resources
- `alloy`: image and resources
- `database`: CloudNativePG cluster, credentials, storage, and resources
- `grafana`: image, credentials, database, ingress, and resources

Use a separate override file for credentials and environment-specific values:

```bash
helm upgrade --install monitoring-stack . \
  --namespace monitoring \
  --create-namespace \
  --values values.local.yaml
```

## Known limitations

- Default credentials are stored in the example values file and must be changed.
- MinIO runs as one replica.
- Alloy's log collection mounts `/var/log/pods` and `/var/lib/docker/containers`; verify these paths for the node runtime.
- The bucket initialization Job currently creates Loki and Mimir buckets. Create the configured Tempo bucket separately before relying on Tempo object storage.
- The chart expects the CloudNativePG operator and Nginx Ingress Controller to be installed separately.
# Monitoring Stack

This Helm chart deploys a lightweight monitoring stack for Kubernetes with:

- Grafana for dashboards and visualization
- Loki for logs
- CloudNativePG for the Grafana database
- S3-compatible object storage for Loki index and chunks
- Traefik ingress for Grafana access

The stack is designed for a local or lab Kubernetes environment and is configured for simple internal routing and local service discovery.

## Architecture

```mermaid
flowchart LR
    Browser[Browser / User] --> Ingress[Nginx Ingress\nHost: grafana.pcs.ui]
    Ingress --> GrafanaSvc[Service: monitoring-stack-grafana\nClusterIP: 3000]
    GrafanaSvc --> GrafanaPod[Deployment: monitoring-stack-grafana\n2 replicas]

    GrafanaPod --> PGSvc[Service: grafana-db-cluster-rw\n5432]
    PGSvc --> PGCluster[CloudNativePG Cluster\n3 instances]

    LokiPod[Loki StatefulSet\n3 replicas] --> LokiSvc[Service: monitoring-stack-loki\nClusterIP: 3100]
    LokiPod --> Headless[Service: loki-headless\nclusterIP: None]
    LokiPod --> S3[(MinIO\nmonitoring-stack-minio:9000)]

    GrafanaPod --> Datasource[ConfigMap datasource\nLoki endpoint]
    LokiPod --> Config[ConfigMap: loki-configmap]
``` 

## Components

### 1. Grafana

The Grafana deployment is created from the chart and exposes a web UI through Nginx.

- Deployment: `monitoring-stack-grafana`
- Replicas: 2
- Service: `monitoring-stack-grafana`
- Port: `3000`
- Ingress host: `grafana.pcs.ui`
- Default credentials: user `admin`, password `admin123`

Grafana is configured to connect to the PostgreSQL database via the CloudNativePG RW service and the Loki datasource for logs.

### 2. PostgreSQL / CloudNativePG

A PostgreSQL cluster is created with CloudNativePG for Grafana persistence.

- Cluster name: `grafana-db-cluster`
- Database: `grafana`
- User: `grafana`
- Secret: `monitoring-stack-grafana-db-credentials`
- Instances: 3
- Storage: `5Gi` using `longhorn`

The database is exposed internally through the generated CNPG services:

- `grafana-db-cluster-rw` for writes
- `grafana-db-cluster-r`
- `grafana-db-cluster-ro`

### 3. Loki

Loki is deployed as a StatefulSet with replication for log storage and clustering.

- StatefulSet: `loki`
- Replicas: 3
- StatefulSet service name: `loki-headless`
- Service: `monitoring-stack-loki` (ClusterIP)
- Port: `3100` for HTTP, `9095` for gRPC
- Memberlist port: `7946`

The Loki config uses a `ConfigMap` and reads S3 credentials from a Kubernetes Secret. Loki writes data to an S3-compatible object store using the configured endpoint and bucket name.

### 4. MinIO object storage

Loki is configured to use the in-cluster MinIO service as its S3-compatible backend for logs.

- Endpoint: `monitoring-stack-minio:9000`
- Bucket: `loki-data`
- Credentials are stored in the `monitoring-stack-loki-secret` Secret

This provides persistent log storage outside the pod filesystem.

### 5. Ingress and routing

Grafana is exposed by Nginx through an Ingress resource.

#### Route

```text
Browser --> grafana.pcs.ui --> Nginx Ingress --> Service: monitoring-stack-grafana --> Grafana Pod
```

This is configured in the chart with:

- Ingress class: `nginx`
- Host: `grafana.pcs.ui`
- Path: `/`
- Backend: `monitoring-stack-grafana:3000`

Loki is exposed separately through its internal ClusterIP service at `monitoring-stack-loki:3100`.

## Internal service flow

### Grafana request flow

```text
HTTP request
  -> Traefik ingress
  -> monitoring-stack-grafana Service
  -> Grafana Deployment pod
  -> PostgreSQL Cluster (through grafana-db-cluster-rw)
  -> Grafana database
```

### Loki log flow

```text
Application / logs
  -> Loki pod (collector/ingester)
  -> Loki config and storage settings
  -> S3-compatible object store
```

### Grafana datasource flow

Grafana uses a ConfigMap to configure a Loki datasource.

```text
Grafana UI
  -> datasource config
  -> Loki service
  -> Loki HTTP endpoint: http://loki.<namespace>.svc.cluster.local:3100
```

## Helm values

The chart is parameterized through the root `values.yaml` file.

Key values include:

- `loki.replicas`
- `loki.serviceName`
- `loki.servicePort`
- `loki.s3.bucketName`
- `database.clusterName`
- `database.instances`
- `database.storageSize`
- `grafana.ingress.host`
- `grafana.replicas`

## Deployment

Install or upgrade the stack with:

```bash
helm upgrade --install monitoring-stack . -n monitoring --create-namespace
```

Verify the resources:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get ingress -n monitoring
kubectl get statefulset -n monitoring
```

## Notes

- The Grafana app is served at `grafana.pcs.ui` through the Nginx ingress.
- Loki is available internally at `monitoring-stack-loki:3100` and does not use the Grafana ingress path.
- The database is internal to the cluster and is not exposed directly outside Kubernetes.
- S3 credentials and database credentials are managed through Kubernetes Secrets.

## Summary

This stack provides a practical local observability environment:

- Grafana for UI and dashboards
- Loki for log aggregation
- CNPG for Grafana persistence
- S3-compatible storage for long-term log retention
- Ingress-driven access for the Grafana frontend

This gives a simple path from local development or lab deployment to a working monitoring environment with minimal operational overhead.
