# Observability

We will setup a monitoring cluster within kubernetes to centralize and process logs.
These logs can originate from different sources, from kubernetes it self or other applications deployed outside kubernetes.

The stack we will be using consists of Grafana, Loki and Promtail.
First, we will use promtail to push logs from different sources to loki
Loki will aggregate these logs and store them for further analysis or audit needs
Last Grafana will visualize these logs and we can create alerts on some events if needed.

## Spin-up a testing minikube cluster

To simplify the demo, we will start from a fresh kubernetes cluster run localy using minikube and we are using Terraform as our Infrastructure As code tool.

``` bash
minikube start --profile observability-cluster
```

## Use Terraform for Infrastructure as Code

This ensures that we can bootstrap the cluster quickly in a consistent and declarative way.
Also, this same script can be used to deploy this cluster and all resources that we are about to configure on different cloud providers.

We create a `main.tf` file that will hold all our Terraform script.
We define the two Terraform providers needed for this setup, `Helm` and `Kubectl`.
In line number 11, we use our `Kubectl` provider to create a namespace for our helm charts.

``` terraform linenums="1"
# Initialize terraform providers
provider "kubernetes" {
  config_path = "~/.kube/config"
}
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}
# Create a namespace for observability
resource "kubernetes_namespace" "observability-namespace" {
  metadata {
    name = "observability"
  }
}
```

## Bootstrap Grafana

Next, we deploy our first Helm chart to install Grafana in the cluster:

``` terraform linenums="1"
# Helm chart for Grafana
resource "helm_release" "grafana" {
  name             = "grafana"
  repository       = "https://grafana.github.io/helm-charts"
  chart            = "grafana"
  version          = "7.1.0"
  namespace        = "observability"

  values = [file("${path.module}/values/grafana.yaml")]
  depends_on = [ kubernetes_namespace.observability-namespace ]
}
```

We use a value file to configure our Grafana by defining the password for the admin user and, later on, the datasource for Loki.
This will help use avoid configuring it each time we redeploy our cluster.

Let's go ahead and test our setup so far by typing terrafrom command in the terminal:

``` bash
terraform init
terraform apply --auto-apply
```

This will initialize terraform providers and state and deploy the Grafana helm chart.

``` bash title="terraform output
TODO: put the output here
```

We check what was created on the `observability` namespace and that all the pods are running.

``` bash title="kubectl get all --namespace observability"
TODO: put the output here
```

To access the Grafana UI, we need to get access to the cluster.
Since we are using a local minikube cluster, the easiest way to access the Grafana service is to create a `port-forward` using the command.
This command forward traffic from our cluster on port 80 to our local machine on port 3000.

``` bash
kubectl port-forward -n observability svc/grafana 3000:80
```

We can log into Grafana UI using the `admin` login and the password defined in the `values/grafana.yaml`.

TODO: Grafana screenshot.

## Bootstrap Loki

We do the same with Loki using Terraform too:

``` terraform linenums="1"
# Helm chart for Loki
resource "helm_release" "loki" {
  name       = "loki"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "loki"
  version    = "5.41.5"
  namespace  = "observability"

  values = [file("${path.module}/values/loki.yaml")]
  depends_on = [ kubernetes_namespace.observability-namespace ]
}  
```

To configure our Loki deployemnt, we create a configuration file under `values/loki.yaml` and we define our storage settings with the kind of architecture we want to use.
Since we are deploying localy for testing only, we use the single binary architecture and the local storage option.

``` yaml linenums="1"
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  storage:
    type: 'filesystem'
singleBinary:
  replicas: 1

promtail:
  enabled: true
```

## Deploy and configure Promtail

Finaly, we deploy Promtail to ship logs to Loki
Loki mode of getting logs is different from Prometheus.
Prometheus pulls metrics from different targets that are providing metrics endpoints.
For logs, Loki relay on push of logs from log shiping tools such as Promtail.
We follow the same approach to deploy Promtail on our cluster:

``` terraform linenums="1"
# Helm chart for promtail
resource "helm_release" "promtail" {
  name       = "promtail"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "promtail"
  version    = "6.15.3"
  namespace  = "observability"

  values = [file("${path.module}/values/promtail.yaml")]
  depends_on = [ kubernetes_namespace.observability-namespace ]
}
```

To configure Promtail, we create a config file `values/promtail.yaml` with the following content:

``` yaml linenums="1"
extraVolumes:
  - name: node-logs
    hostPath:
      path: /var/log

extraVolumeMounts:
  - name: node-logs
    mountPath: /var/log/host
    readOnly: true

# Add Loki as a client to Promtail
clients:
  - url: http://loki-gateway.observability.svc.cluster.local/loki/api/v1/push
```

This configure Promtail to fetch logs from the current host located in `/var/log/host` and ship them to Loki endpoint.

And that's it, we have put all the pieces necessary for collecting logs from Kubernetes and shiping them to Grafana for further analysis and audits.

$TODO: Put the necessary screenshot$

## Video tutorial

If you prefer to follow along, this tutorial is available here:
