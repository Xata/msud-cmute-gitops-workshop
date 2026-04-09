<picture>
  <source media="(prefers-color-scheme: dark)" srcset="pics/tcc_logo_darkmode.png">
  <source media="(prefers-color-scheme: light)" srcset="pics/tcc_logo.png">
  <img alt="The Cybersecurity Center" src="pics/tcc_logo.png">
</picture>

# MSU Denver GitOps Workshop

Deploy a PaperMC Minecraft server to DigitalOcean Kubernetes using GitOps principles.

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![ArgoCD](https://img.shields.io/badge/Argo%20CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![OpenTofu](https://img.shields.io/badge/OpenTofu-FFDA18?style=for-the-badge&logo=opentofu&logoColor=black)
![DigitalOcean](https://img.shields.io/badge/DigitalOcean-0080FF?style=for-the-badge&logo=digitalocean&logoColor=white)
![Envoy Gateway](https://img.shields.io/badge/Envoy%20Gateway-AC6199?style=for-the-badge&logo=envoyproxy&logoColor=white)
![PaperMC](https://img.shields.io/badge/PaperMC-EEE?style=for-the-badge&logo=minecraft&logoColor=black)

## Prerequisites

- [Git](https://git-scm.com/)
- [Docker](https://docs.docker.com/get-docker/)
- [doctl](https://docs.digitalocean.com/reference/doctl/how-to/install/)
- [OpenTofu](https://opentofu.org/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- A GitHub account
- A DigitalOcean API token

## Lab Environment Setup

### Fork the repository

1. Fork this repository to your GitHub account.

2. In your fork, go to `.github/workflows/build.yaml` and uncomment the entire file.

3. Commit and push to `main`. Go to the **Actions** tab in your fork and verify the workflow runs successfully. Once complete, your container image will be available at `ghcr.io/<your-github-username>/msud-cmute-gitops-workshop:latest`.

### Provision your cluster

1. Install and authenticate doctl:

```bash
doctl auth init --access-token <YOUR_DO_TOKEN>
```

2. Provision your DOKS cluster:

```bash
cd infra/tofu
cp terraform.tfvars.example terraform.tfvars
```

3. Edit `terraform.tfvars` — replace `<YOUR_NAME>` in `cluster_name` with your name (lowercase, no spaces).

4. Apply:

```bash
tofu init
tofu plan
tofu apply
```

5. Save your kubeconfig:

```bash
doctl kubernetes cluster kubeconfig save paper-<YOUR_NAME>
```

Example: 

```bash
doctl kubernetes cluster kubeconfig save paper-test
```

6. Verify access:

```bash
kubectl get nodes
```

You should see a single node in `Ready` status.

Example:

```bash
NAME            STATUS   ROLES    AGE     VERSION
default-1x8h9   Ready    <none>   2m59s   v1.35.1
```

### Bootstrap cluster infrastructure

1. Install Envoy and the Gateway API CRDs: 

```bash
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
```

You should see:

```bash
customresourcedefinition.apiextensions.k8s.io/backendtlspolicies.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io serverside-applied
```

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.1 \
  -n envoy-gateway-system \
  --create-namespace \
  --skip-crds
```

2. Install cert-manager:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.3/cert-manager.yaml
```

```bash
kubectl wait --for=condition=available deployment/cert-manager -n cert-manager --timeout=120s
```

3. Create the DigitalOcean DNS-01 secret and ClusterIssuer (use the same DO token from earlier):

NOTE: BE SURE TO REPLACE THE EMAIL LINE!!!

```bash
kubectl create secret generic digitalocean-dns \
  --namespace cert-manager \
  --from-literal=access-token=<YOUR_DO_TOKEN>
```

Edit `infra/cert-manager/clusterissuer.yaml` and replace `<YOUR_EMAIL>` with your email. Then run:

```bash
kubectl apply -f infra/cert-manager/clusterissuer.yaml
```

4. Install ArgoCD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

5. Verify everything is running:

```bash
kubectl get pods -n envoy-gateway-system
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s
```

### Prepare ArgoCD for the shared Gateway

The single `paper-gateway` (defined in `k8s/gateway.yaml`) has two listeners: a TCP listener for Minecraft and an HTTPS listener for the ArgoCD UI. Both share the same LoadBalancer IP.

1. Tell ArgoCD not to terminate TLS itself (the Gateway does it):

```bash
kubectl patch deployment argocd-server -n argocd --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'
kubectl rollout status deployment/argocd-server -n argocd
```

2. Edit `k8s/argocd-httproute.yaml` and replace `<YOUR_NAME>` with your name, then apply:

```bash
kubectl apply -f k8s/argocd-httproute.yaml
```

### Deploy with ArgoCD

1. Look at all files in `k8s/`:
   - `namespace.yaml`
   - `configmap.yaml`
   - `persistentvolumeclaim.yaml`
   - `deployment.yaml` IMPORTANT: replace `<YOUR_GITHUB_USERNAME>` with your GitHub username
   - `service.yaml`
   - `gateway.yaml` IMPORTANT: replace `<YOUR_NAME>` (the HTTPS listener hostname for ArgoCD)
   - `tcproute.yaml`
   - `certificate.yaml` IMPORTANT: replace `<YOUR_NAME>` in both certs (matches your DNS records below)

2. In `argocd/application.yaml`, look at the file and replace `<YOUR_GITHUB_USERNAME>` with your GitHub username.

3. Commit and push.

4. Apply the ArgoCD application:

```bash
kubectl apply -f argocd/application.yaml
```

5. Watch ArgoCD deploy everything:

```bash
kubectl get applications -n argocd
kubectl get pods -n paper
```

### Expose the server

1. Get your Gateway's external IP:

```bash
kubectl get gateway -n paper
```

2. Create your DNS record:

```bash
doctl compute domain records create labs.cmute.cloud \
  --record-type A \
  --record-name "<YOUR_NAME>.mc" \
  --record-data <GATEWAY_EXTERNAL_IP> \
  --record-ttl 300
```

Example:

```bash
doctl compute domain records create labs.cmute.cloud \
  --record-type A \
  --record-name "alice.mc" \
  --record-data 143.198.1.50 \
  --record-ttl 300
```

3. Create a second A record for ArgoCD pointing at the same Gateway IP:

```bash
doctl compute domain records create labs.cmute.cloud \
  --record-type A \
  --record-name "argocd.<YOUR_NAME>.mc" \
  --record-data <GATEWAY_EXTERNAL_IP> \
  --record-ttl 300
```

4. Get the initial ArgoCD admin password and log in at `https://argocd.<YOUR_NAME>.mc.labs.cmute.cloud`:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

3. Connect with your Minecraft client to `<YOUR_NAME>.mc.labs.cmute.cloud`.

### Test GitOps

Make a change and watch ArgoCD sync it automatically.

1. Edit `k8s/configmap.yaml`. Change `max-players` to `50`.

2. Commit and push.

3. Watch the sync:

```bash
kubectl get configmap paper-config -n paper -o yaml
```

ArgoCD will detect the change in your repo and apply it to the cluster automatically.

## Cleanup

Destroy your cluster when you're done:

```bash
cd infra/tofu
tofu destroy
```