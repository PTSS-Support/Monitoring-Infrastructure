# Monitoring-Infrastructure

This repository contains the Kubernetes manifests for setting up Prometheus and Grafana monitoring infrastructure in OpenShift. The setup includes proper RBAC configurations, routes, and necessary configurations for monitoring applications in the `hotel-dev` namespace.

## Components

- **Prometheus**: For metrics collection and storage
- **Grafana**: For metrics visualization and dashboarding
- **Routes**: OpenShift routes for accessing both Prometheus and Grafana
- **RBAC**: Required permissions for Prometheus to scrape metrics
- **ConfigMaps**: Configuration for Prometheus scraping

## Detailed Component Explanations

### Prometheus Configuration (`prometheus.yaml`, `prometheus-configmap.yaml`)
The Prometheus deployment consists of two key parts:
1. **Deployment Configuration** (`prometheus.yaml`):
    - Defines the Prometheus container deployment
    - Sets resource limits and requests
    - Mounts necessary configuration volumes
    - Uses `prom/prometheus:v2.45.0` image

2. **Scraping Configuration** (`prometheus-configmap.yaml`):
    - Contains the core Prometheus configuration
    - Defines service discovery settings
    - Key features:
        - Auto-discovers pods in the `hotel-dev` namespace
        - Uses Kubernetes service discovery (`kubernetes_sd_configs`)
        - Implements relabeling for proper metric collection
        - Configures a 15-second scrape interval

### RBAC Setup (`prometheus-rbac.yaml`)
The RBAC configuration is crucial for Prometheus to function properly:
1. **ServiceAccount**:
    - Creates a dedicated service account for Prometheus
    - Ensures proper identity for the Prometheus pod

2. **Role**:
    - Grants specific permissions in the `hotel-dev` namespace
    - Allows Prometheus to:
        - List and watch services, endpoints, and pods
        - Access ConfigMaps
    - These permissions are necessary for service discovery

3. **RoleBinding**:
    - Links the Role to the ServiceAccount
    - Ensures Prometheus has the right permissions to scrape metrics

### Service Discovery
Prometheus uses Kubernetes service discovery to automatically find and monitor pods:
1. **How it works**:
   ```yaml
   kubernetes_sd_configs:
     - role: pod
       namespaces:
         names:
           - hotel-dev
   ```
    - Continuously watches for pods in the `hotel-dev` namespace
    - Uses relabeling to process pod annotations

2. **Pod Discovery**:
    - Pods must have these annotations to be discovered:
      ```yaml
      prometheus.io/scrape: "true"
      prometheus.io/path: "/metrics"
      prometheus.io/port: "8080"
      ```
    - The relabeling rules in `prometheus-configmap.yaml` process these annotations

### Grafana Setup (`grafana.yaml`, `grafana-datasources.yaml`)
1. **Grafana Deployment** (`grafana.yaml`):
    - Deploys the Grafana instance
    - Configures the root URL
    - Sets resource limits
    - Uses `emptyDir` for data storage

2. **Data Source Configuration** (`grafana-datasources.yaml`):
    - Automatically configures Prometheus as a data source
    - Uses service discovery through labels
    - Key configuration:
      ```yaml
      datasource:
        type: prometheus
        url: 'http://prometheus:9090'
        isDefault: true
      ```

### Networking Setup
1. **Prometheus Service** (`prometheus-server.yaml`):
    - Creates a Kubernetes service for Prometheus
    - Exposes port 9090
    - Enables internal cluster communication

2. **Routes** (`prometheus-route.yaml`, `grafana-route.yaml`):
    - Configure OpenShift routes for external access
    - Enable TLS termination
    - Force HTTPS redirect through `insecureEdgeTerminationPolicy: Redirect`


## Prerequisites

- OpenShift cluster access
- `oc` CLI tool installed
- Appropriate permissions in the `hotel-dev` namespace

## Installation Order

Follow these steps to deploy the monitoring infrastructure:

1. First, create the namespace (if not exists):
   ```bash
   oc new-project hotel-dev
   ```

2. Set up Prometheus RBAC:
   ```bash
   oc apply -f prometheus-rbac.yaml
   ```

3. Create Prometheus ConfigMap:
   ```bash
   oc apply -f prometheus-configmap.yaml
   ```

4. Deploy Prometheus:
   ```bash
   oc apply -f prometheus.yaml
   oc apply -f prometheus-server.yaml
   oc apply -f prometheus-route.yaml
   ```

5. Deploy Grafana:
   ```bash
   oc apply -f grafana.yaml
   oc apply -f grafana-route.yaml
   oc apply -f grafana-datasources.yaml
   ```

## Accessing the Services

After deployment, the services will be available at:

- Grafana: https://hotel-grafana-hotel-dev.apps.inholland.hcs-lab.nl
- Prometheus: https://prometheus-hotel-dev.apps.inholland.hcs-lab.nl

## Configuration

### Prometheus

Prometheus is configured to automatically discover and scrape metrics from pods in the `hotel-dev` namespace that have the following annotations:

```yaml
prometheus.io/scrape: "true"
prometheus.io/path: "/metrics"
prometheus.io/port: "8080"
```

### Grafana

Grafana is configured with:
- Prometheus as the default data source
- Resource limits of 200m CPU and 256Mi memory
- Persistent storage using emptyDir volume

## Resource Requirements

The deployment includes resource limits and requests:

- Prometheus:
    - Requests: 100m CPU, 128Mi memory
    - Limits: 200m CPU, 256Mi memory
- Grafana:
    - Requests: 100m CPU, 128Mi memory
    - Limits: 200m CPU, 256Mi memory

