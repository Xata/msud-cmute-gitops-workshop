# Observability — Grafana + Prometheus

Monitor your PaperMC server with Prometheus metrics and a Grafana dashboard.

## Prerequisites

Make sure you have completed the main workshop setup and your PaperMC server is running.

## Copy the metrics-enabled files

Copy the updated Dockerfile, deployment, and service that include the Prometheus exporter plugin:

```bash
cp workshop/metrics/Dockerfile Dockerfile
cp workshop/metrics/deployment.yaml k8s/deployment.yaml
cp workshop/metrics/service.yaml k8s/service.yaml
```

Commit and push. Your CI will rebuild the image with the Prometheus exporter plugin, and ArgoCD will deploy the updated manifests.

## Install kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.sidecar.dashboards.enabled=true \
  --set grafana.sidecar.dashboards.label=grafana_dashboard
```

Wait for it to come up:

```bash
kubectl get pods -n monitoring
```

## Apply the observability resources

Edit the following files and replace `<YOUR_NAME>` with your name:

- `workshop/observability/certificate.yaml`
- `workshop/observability/httproute.yaml`

Then apply everything:

```bash
kubectl apply -f workshop/observability/servicemonitor.yaml
kubectl apply -f workshop/observability/dashboard.yaml
kubectl apply -f workshop/observability/certificate.yaml
kubectl apply -f workshop/observability/httproute.yaml
```

## Add the Grafana listener to your Gateway

Add this listener to your `k8s/gateway.yaml` under `spec.listeners`:

```yaml
    - name: grafana-https
      protocol: HTTPS
      port: 443
      hostname: grafana.<YOUR_NAME>.mc.labs.cmute.cloud
      tls:
        mode: Terminate
        certificateRefs:
          - name: grafana-tls
      allowedRoutes:
        namespaces:
          from: All
```

Commit and push. ArgoCD will sync the updated gateway.

## Create the DNS record

Use the same Gateway IP as your Minecraft server:

```bash
kubectl get gateway paper-gateway -n paper
```

```bash
doctl compute domain records create labs.cmute.cloud \
  --record-type A \
  --record-name "grafana.<YOUR_NAME>.mc" \
  --record-data <SAME_GATEWAY_IP> \
  --record-ttl 300
```

## Access Grafana

Get your Grafana password:

```bash
kubectl get secret -n monitoring prometheus-stack-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d && echo
```

Navigate to `https://grafana.<YOUR_NAME>.mc.labs.cmute.cloud`

- **Username:** admin
- **Password:** output from above

The PaperMC Server dashboard will be available under Dashboards. It shows TPS, tick duration, JVM memory, player count, entities, and more.