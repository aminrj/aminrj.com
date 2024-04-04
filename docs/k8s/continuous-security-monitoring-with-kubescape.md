# Continuous Kubernetes Security Scans with Kubescape

## Prerequisites

* A running Minikube cluster.
* Terraform installed on your system.
* Basic understanding of Kubernetes concepts and the use of `kubectl`.

If you are new to the topic, consider reading this article bellow that would help you with the prerequisites:

:construction: **TODO:** Add article link

## Steps

### 1. Set up Terraform

* Create a new directory for your project.
* Create a file named `main.tf` with the following Terraform configuration:

``` terraform

# Initilize terraform providers
provider "kubernetes" {
config_path = "~/.kube/config"
}

provider "helm" {
kubernetes {
    config_path = "~/.kube/config"
}
}
```

* Initialize Terraform:

    ``` bash
    terraform init
    ```

### 2. Deploy Grafana and Prometheus

```terraform
# Create a namespace for observability
resource "kubernetes_namespace" "observability-namespace" {
    metadata {
        name = "observability"
    }
}

# Prometheus setup
resource "helm_release" "prometheus" {
    name       = "prometheus"
    repository = "https://prometheus-community.github.io/helm-charts"
    chart      = "prometheus"
    version    = "25.8.2"
    namespace  = "observability"

    depends_on = [kubernetes_namespace.observability-namespace]
}

# Grafana setup
resource "helm_release" "grafana" {
    name       = "grafana"
    repository = "https://grafana.github.io/helm-charts"
    chart      = "grafana"
    version    = "7.1.0"
    namespace  = "observability"

    values     = [file("${path.module}/values/grafana.yaml")]
    depends_on = [kubernetes_namespace.observability-namespace]
} 
```

Before hitting `terraform apply`, you need to add the Grafana configuration in the `values` folder:

``` yaml title="values/grafana.yaml"
persistence.enabled: true
persistence.size: 10Gi
persistence.existingClaim: grafana-pvc
persistence.accessModes[0]: ReadWriteOnce
persistence.storageClassName: standard
  
adminUser: admin
adminPassword: grafana

datasources: 
 datasources.yaml:
   apiVersion: 1
   datasources:
    # configure Prometheus datasource
    - name: Prometheus
      type: prometheus
      access: proxy
      orgId: 1
      url: http://prometheus-server.observability.svc.cluster.local
      basicAuth: false
      isDefault: false
      version: 1
      editable: true
```

Here we specify default admin credentials (only for demonstration purpose, do not try this on a real production cluster). We also add the data source for Prometheus so we don't have to add it in the Grafa UI each time we deploy our infrastructure.

* Deploy Prometheus and Grafana:

     ```bash
     terraform apply
     ```

     (Type 'yes' when prompted)

### 3. Deploy Loki and Promtail

Since, we will be using Loki to send security logs to Grafana along with Kubernetes audit logs, we configure Loki and Promtail as well:

```terraform
# Helm chart for Loki
resource "helm_release" "loki" {
    name       = "loki"
    repository = "https://grafana.github.io/helm-charts"
    chart      = "loki"
    version    = "5.41.5"
    namespace  = "observability"

    values     = [file("${path.module}/values/loki.yaml")]
    depends_on = [kubernetes_namespace.observability-namespace]
} 

# Helm chart for promtail
resource "helm_release" "promtail" {
    name       = "promtail"
    repository = "https://grafana.github.io/helm-charts"
    chart      = "promtail"
    version    = "6.15.3"
    namespace  = "observability"

    values     = [file("${path.module}/values/promtail.yaml")]
    depends_on = [kubernetes_namespace.observability-namespace]
}
```

Same thing here with the configuration of the tools, create the following yaml files in the `values` folder:

```yaml title="values/loki.yaml"
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  storage:
    type: 'filesystem'
singleBinary:
  replicas: 1
```

```yaml title="values/prometheus.yaml"
# Mount folder /var/log from node
extraVolumes:
  - name: node-logs
    hostPath:
      path: /var/log

extraVolumeMounts:
  - name: node-logs
    mountPath: /var/log/host
    readOnly: true

# Add Loki as a client to Promtail
config:
  clients:
    - url: http://loki-gateway.observability.svc.cluster.local/loki/api/v1/push

# Scraping kubernetes audit logs located in /var/log/kubernetes/audit/
  snippets:
    scrapeConfigs: |
      - job_name: audit-logs
        static_configs:
          - targets:
              - localhost
            labels:
              job: audit-logs
              __path__: /var/log/host/kubernetes/**/*.log
```

* Deploy Loki and Promtail:

     ```bash
     terraform apply
     ```

     (Type 'yes' when prompted)

## 4. Install Kubescape

Following the same GitOps approach, we will deploy Kubescape the same way we deploy other observability tools, using Terraform to deploy Kubescape operator Helm chart.

```terraform
# Install the Kubescape Helm chart
resource "helm_release" "kubescape" {
  name       = "kubescape"
  repository = "https://kubescape.github.io/helm-charts"
  chart      = "kubescape-operator"
  version    = "1.18.3"
  namespace  = "kubescape"
  create_namespace = true

  depends_on = [kubernetes_namespace.observability-namespace]

  set {
    name  = "clusterName"
    value = "minikube" 
  }
}
```

Check that the pods are in a running state:

```bash
kubectl get po -n kubescape
NAME                                  READY   STATUS             RESTARTS        AGE
kubescape-5dc5cb78c6-m8ss7            1/1     Running            0               20s
kubevuln-65565b7497-tkzjw             1/1     Running            0               20s
operator-86c99bbf74-8g9xk             1/1     Running            0               13s
storage-5bdb8b54ff-lzs58              1/1     Running            0               20s
```

This Helm chart deploys the Kubescape-operator that contains both Kubevuln and kubescape.

At this point, we can view the results directly using kubectl as follow:

View configuration scan summaries:

```bash
kubectl get workloadconfigurationscansummaries -A
```

Detailed reports are also available:

```bash
kubectl get workloadconfigurationscans -A
```

View image vulnerabilities scan summaries:

```bash
kubectl get vulnerabilitymanifestsummaries -A
```

Detailed reports are available with:

```bash
kubectl get vulnerabilitymanifests -A
```

However, we want to use our monitoring setup to visualize our findings.

## 4. View Kubescape data to Grafana

:construction: **TODO:** Add article link

- Check the configurations
- Take screenshots of the scans
- Think about fixing sending scan data to Prometheus
















## 4. Explore scan results using the CLI tool

```bash
$ kubescape scan framework nsa
 ✅  Initialized scanner
 ✅  Loaded policies
 ✅  Loaded exceptions
 ✅  Loaded account configurations
 ✅  Accessed Kubernetes objects
Control: C-0017 100% |███████████████████████████████████████████████| (24/24, 71 it/s)
 ✅  Done scanning. Cluster: minikube
 ✅  Done aggregating results

──────────────────────────────────────────────────
Framework scanned: NSA
┌─────────────────┬────┐
│        Controls │ 24 │
│          Passed │ 8  │
│          Failed │ 14 │
│ Action Required │ 2  │
└─────────────────┴────┘
Failed resources by severity:
┌──────────┬────┐
│ Critical │ 0  │
│     High │ 16 │
│   Medium │ 51 │
│      Low │ 8  │
└──────────┴────┘
Run with '--verbose'/'-v' to see control failures for each resource.
┌──────────┬─────────────────────────────────────────────┬──────────────────┬───────────────┬───────────────────┐
│ Severity │ Control name                                │ Failed resources │ All Resources │ Compliance score  │
├──────────┼─────────────────────────────────────────────┼──────────────────┼───────────────┼───────────────────┤
│ Critical │ Disable anonymous access to Kubelet service │        0         │       0       │ Action Required * │
│ Critical │ Enforce Kubelet client TLS authentication   │        0         │       0       │ Action Required * │
│   High   │ Resource limits                             │        12        │      25       │        52%        │
│   High   │ Host PID/IPC privileges                     │        2         │      25       │        92%        │
│   High   │ HostNetwork access                          │        1         │      25       │        96%        │
│   High   │ Privileged container                        │        1         │      25       │        96%        │
│  Medium  │ Non-root containers                         │        9         │      25       │        64%        │
│  Medium  │ Allow privilege escalation                  │        5         │      25       │        80%        │
│  Medium  │ Ingress and Egress blocked                  │        14        │      25       │        44%        │
│  Medium  │ Automatic mapping of service account        │        12        │      83       │        86%        │
│  Medium  │ Cluster internal networking                 │        2         │       6       │        67%        │
│  Medium  │ Linux hardening                             │        7         │      25       │        72%        │
│  Medium  │ Secret/etcd encryption enabled              │        1         │       1       │        0%         │
│  Medium  │ Audit logs enabled                          │        1         │       1       │        0%         │
│   Low    │ Immutable container filesystem              │        7         │      25       │        72%        │
│   Low    │ PSP enabled                                 │        1         │       1       │        0%         │
├──────────┼─────────────────────────────────────────────┼──────────────────┼───────────────┼───────────────────┤
│          │              Resource Summary               │        33        │      208      │      67.51%       │
└──────────┴─────────────────────────────────────────────┴──────────────────┴───────────────┴───────────────────┘
```

## 5. Configure Prometheus for Kubescape Metrics

* **Create `prometheus-config.yaml`:**

     ```yaml
     global:
       scrape_interval: 15s 
     scrape_configs:
       - job_name: 'kubescape'
         file_sd_configs:
           - files:
             - results.txt 
     ```

* **Create Kubernetes ConfigMap:**

     ```bash
     kubectl create configmap prometheus-config --from-file=prometheus-config.yaml -n monitoring
     ```

* **Update Prometheus Deployment (in `main.tf`)**

     ```terraform
     # Inside your 'helm_release' for Prometheus:
     set {
         name  = "extraVolumes"
         value =  <<EOF
         - name: config-volume
           configMap:
             name: prometheus-config 
         EOF
       }

       set {
         name  = "extraVolumeMounts"
         value =  <<EOF
         - name: config-volume
           mountPath: /etc/prometheus/prometheus.yml
           subPath: prometheus.yml 
         EOF
       }
     ```  

* **Re-apply Terraform:**

    ```bash
    terraform apply
    ```  

### 6. Set up Grafana Dashboard

* Port-forward Grafana:

     ```bash
     kubectl port-forward -n monitoring service/grafana 3000:80 
     ```

* Access Grafana: http://localhost:3000 (Default: "admin"/"admin")
* Import a Kubescape dashboard ([https://grafana.com/grafana/dashboards/](https://grafana.com/grafana/dashboards/)) or create your own.

### 7. Automate Scans (Optional)

   Set up a Kubernetes CronJob to execute Kubescape scans regularly.
