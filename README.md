# GitOps Deployment of Online Boutique on K3s Cluster

This project implements a production-grade **GitOps continuous delivery workflow** using **ArgoCD** to deploy the **Online Boutique** microservices application onto a dedicated **K3s Kubernetes cluster**.

> **Application Source:**
> The underlying microservices demo application is sourced directly from [Google Cloud Platform Microservices Demo](https://github.com/GoogleCloudPlatform/microservices-demo).

---

## 1. Cluster Architecture & Sizing Specifications

The target environment is built on a 3-Virtual Machine (VM) topology running lightweight **K3s Kubernetes**:

| Node Name | Node Role | vCPU | Memory (RAM) | Responsibilities |
|---|---|---|---|---|
| `cplane-01` (or `k3s-master`) | **Control Plane** | 2 vCPU | 3 GB RAM | K3s Server, SQLite/Kine datastore, CoreDNS, Traefik |
| `node-01` (or `k3s-worker-1`) | **Worker Node 1** | 3 vCPU | 5 GB RAM | Online Boutique microservices & ArgoCD workloads |
| `node-02` (or `k3s-worker-2`) | **Worker Node 2** | 3 vCPU | 5 GB RAM | Online Boutique microservices & ArgoCD workloads |
| `dev-machine` | **Bastion / Client** | - | - | Execution host for `kubectl`, `helm`, `git`, and `labctl` |

### Capacity Breakdown

* **Total Worker Layer Capacity:** 6 vCPU and 10 GB RAM across 2 worker nodes.
* **Control Plane Isolation:** The control plane is isolated with a taint (`CriticalAddonsOnly=true:NoSchedule`) so that application workloads only run on the worker layer.
* **Workload Footprint:**
  * Online Boutique (11 microservices + Redis + Load Generator): ~1.57 vCPU requests / ~1.31 GB RAM requests.
  * ArgoCD Stack (Server, Repo Server, Controller, Redis): ~0.33 vCPU requests / ~0.56 GB RAM requests.
  * Operating System + K3s Agent overhead: ~0.60 vCPU / ~1.40 GB RAM.
* **Total Resource Utilization:** ~2.50 vCPU (~42%) and ~3.27 GB RAM (~33%).
* **Remaining Free Headroom:** ~3.50 vCPU and ~6.73 GB RAM (~67% free) available for traffic scaling and high availability.

---

## 2. Prerequisites

Ensure you have the following requirements satisfied:

* Running K3s cluster matching the specifications above.
* Command-line utilities installed on your management machine: `kubectl` (v1.28+), `helm` (v3.12+), and `git`.
* Firewall ports opened between nodes:
  * `6443/TCP` (Kubernetes API server)
  * `10250/TCP` (Kubelet metrics and logs)
  * `8472/UDP` (Flannel VXLAN overlay network)
  * `80/TCP` and `443/TCP` (HTTP/HTTPS ingress and NodePort traffic)

---

## 3. Step-by-Step Deployment Guide

Follow these exact steps to deploy ArgoCD via Helm and roll out the Online Boutique application.

### Step 3.1: Add the Official Argo Helm Repository

Add the Argo repository:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

Update your local Helm chart index:

```bash
helm repo update
```

---

### Step 3.2: Install and Configure ArgoCD via Helm

Install ArgoCD into the dedicated `argocd` namespace using the optimized values file `argocd/values-argocd.yaml`:

```bash
helm upgrade --install argocd argo/argo-cd --namespace argocd --create-namespace -f argocd/values-argocd.yaml
```

Verify that all ArgoCD pods are running:

```bash
kubectl get pods -n argocd -o wide
```

Verify the ArgoCD server service is exposed via NodePort:

```bash
kubectl get svc argocd-server -n argocd
```

---

### Step 3.3: Retrieve the Initial ArgoCD Admin Password

Extract the automatically generated initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

> **Default Username:** `admin`

---

### Step 3.4: Register the Git Repository in ArgoCD

Apply the declarative repository secret to register your Git repository into ArgoCD:

```bash
kubectl apply -f argocd/repository.yaml
```

Verify the repository connection status using the ArgoCD CLI:

```bash
argocd repo list
```

---

### Step 3.5: Deploy the Online Boutique Application via GitOps

Apply the ArgoCD `Application` custom resource to trigger the automated deployment:

```bash
kubectl apply -f argocd/argocd-application.yaml
```

ArgoCD will automatically clone the repository, render the Helm chart from `./helm-chart` with `values-k3s.yaml`, and deploy all 11 microservices into the `default` namespace.

---

### Step 3.6: Configure 1-Minute Automated Git Reconciliation

Configure ArgoCD to check your Git repository for new commits every 60 seconds (1 minute):

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"timeout.reconciliation":"60s","timeout.reconciliation.jitter":"0s"}}'
```

---

### Step 3.7: Verify Microservices Deployment Health

Verify that all application pods are distributed across the worker nodes:

```bash
kubectl get pods -n default -o wide
```

Verify that all internal services are healthy:

```bash
kubectl get svc -n default
```

Check real-time resource consumption across all nodes:

```bash
kubectl top nodes
```

Check resource utilization per pod:

```bash
kubectl top pods -n default --sort-by=memory
```

---

## 4. Environment & Public Exposure Guide

This cluster is hosted on **iximiuz Labs** and managed under **Rancher**.

To access both the **Online Boutique web store** and the **ArgoCD web dashboard** from your local browser, follow the exposure instructions below.

---

### Method A: Public Access via NodePort (Persistent & Recommended)

The NodePort service routes traffic directly through the cluster worker nodes (`node-01` or `node-02`). Use `labctl` to generate a secure public URL.

#### Expose the Online Boutique Storefront (Port 30080)

Apply the frontend NodePort service:

```bash
kubectl apply -f frontend-nodeport.yaml
```

Verify the NodePort is bound on port 30080:

```bash
kubectl get svc frontend-nodeport -n default
```

Generate the public URL by exposing port 30080 on worker node `node-01`:

```bash
labctl expose port $PLAY_ID 30080 --machine node-01 --public
```

Open the generated HTTPS URL in your web browser to browse the Online Boutique storefront.

---

#### Expose the ArgoCD Web Dashboard (Port 30085)

Verify that the `argocd-server` service is running on NodePort 30085:

```bash
kubectl get svc argocd-server -n argocd
```

Generate the public URL by exposing port 30085 on worker node `node-01`:

```bash
labctl expose port $PLAY_ID 30085 --machine node-01 --public
```

Open the generated HTTPS URL in your browser and log in with username `admin` and the password retrieved in Step 3.3.

---

### Method B: Public Access via Port-Forwarding (On-Demand / Debugging)

Port-forwarding tunnels traffic from Kubernetes services to the `dev-machine` VM.

> **Crucial Rule:**
> You must always supply the `--address 0.0.0.0` flag with `kubectl port-forward`. If omitted, the port will only listen on `127.0.0.1` inside the VM, preventing `labctl` from creating a public route.

#### Port-Forward the Online Boutique Storefront (Port 8080)

Forward port 80 of the frontend service to port 8080 on the `dev-machine` VM:

```bash
labctl ssh -m dev-machine $PLAY_ID -- kubectl port-forward --address 0.0.0.0 svc/frontend 8080:80
```

In a separate terminal, expose the forwarded port 8080 to generate a public URL:

```bash
labctl expose port $PLAY_ID 8080 --machine dev-machine --public
```

---

#### Port-Forward the ArgoCD Dashboard (Port 8085)

Forward port 80 of the ArgoCD server to port 8085 on the `dev-machine` VM:

```bash
labctl ssh -m dev-machine $PLAY_ID -- kubectl port-forward --address 0.0.0.0 -n argocd svc/argocd-server 8085:80
```

In a separate terminal, expose the forwarded port 8085 to generate a public URL:

```bash
labctl expose port $PLAY_ID 8085 --machine dev-machine --public
```

Open the resulting URL in your browser to access the ArgoCD console.
