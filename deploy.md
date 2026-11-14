# Panduan Migrasi, Deployment & Expose Online Boutique pada K3s Cluster (Implementasi GitOps ArgoCD dengan Helm)

Dokumen ini adalah panduan lengkap dan *step-by-step* untuk mendeploy dan menjalankan aplikasi **Google Online Boutique (`microservices-demo`)** pada klaster **K3s (3 VM: 1 Control Plane & 2 Worker Nodes)** dengan implementasi **ArgoCD via Helm Chart Resmi (`argo/argo-cd`)** menggunakan repositori:
**`https://github.com/Alvarras/microservices-on-k3s-cluster.git`**

---

## 1. Analisis Kapasitas Klaster (3 VM: 1 Master & 2 Workers)

### 1.1 Spesifikasi Infrastruktur VM / Playground

| Nama Mesin / Node | Peran (Role) | vCPU | RAM | Catatan |
|---|---|---|---|---|
| `cplane-01` (atau `k3s-master`) | **Control Plane** | 2 vCPU | 3 GB RAM | K3s API server, SQLite/Kine datastore, CoreDNS |
| `node-01` (atau `k3s-worker-1`) | **Worker Node 1** | 3 vCPU | 5 GB RAM | Pod Online Boutique & ArgoCD |
| `node-02` (atau `k3s-worker-2`) | **Worker Node 2** | 3 vCPU | 5 GB RAM | Pod Online Boutique & ArgoCD |
| `dev-machine` | **Client / Bastion** | - | - | Mesin eksekusi `kubectl`, `helm`, `git`, dan `labctl` |
| **TOTAL LAYER WORKER** | **2 Worker Nodes** | **6 vCPU** | **10 GB RAM** | **Kapasitas komputasi workload** |

---

### 1.2 Analisis Kelayakan Workload Online Boutique + ArgoCD (Helm)

```text
1. OS Linux + K3s Agent (2 Worker Nodes) : ~0.60 vCPU | ~1.40 GB RAM
2. Online Boutique Baseline Requests     : ~1.57 vCPU | ~1.31 GB RAM
3. ArgoCD Helm Stack Requests            : ~0.33 vCPU | ~0.56 GB RAM
-----------------------------------------------------------------------
TOTAL TERPAKAI (Allocated Requests)      : ~2.50 vCPU | ~3.27 GB RAM (~33% Utilisasi)
TOTAL KAPASITAS WORKER LAYER             :  6.00 vCPU | 10.00 GB RAM
=======================================================================
SISA KAPASITAS BEBAS (HEADROOM)          : ~3.50 vCPU | ~6.73 GB RAM (~67% SISA BEBAS!)
```

> [!IMPORTANT]
> **Kesimpulan:** Klaster Anda **SANGAT MEMADAI**. Penambahan ArgoCD via Helm yang sudah dioptimasi hanya memakan **~560 MB RAM**, menyisakan **lebih dari 6.7 GB RAM bebas** di layer worker.

---

## 2. File Konfigurasi di Repositori

| File | Fungsi |
|---|---|
| **[argocd/values-argocd.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/values-argocd.yaml)** | Konfigurasi Helm resmi `argo/argo-cd` teroptimasi (insecure mode, NodePort 30085 terintegrasi, resource tuning). |
| **[argocd/argocd-application.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/argocd-application.yaml)** | Manifest GitOps Application ArgoCD mengarah ke `https://github.com/Alvarras/microservices-on-k3s-cluster.git`. |
| **[values-k3s.yaml](file:///home/vrs/Projects/rancher/microservices-demo/values-k3s.yaml)** | Konfigurasi Helm Online Boutique teroptimasi untuk K3s. |
| **[frontend-nodeport.yaml](file:///home/vrs/Projects/rancher/microservices-demo/frontend-nodeport.yaml)** | Service bertipe `NodePort` (port `30080`) untuk mengekspos frontend toko. |
| **[redis-cart-pvc.yaml](file:///home/vrs/Projects/rancher/microservices-demo/redis-cart-pvc.yaml)** | Persistensi data keranjang belanja menggunakan storage `local-path` K3s. |
| **[k3s-ingress.yaml](file:///home/vrs/Projects/rancher/microservices-demo/k3s-ingress.yaml)** | Traefik Ingress untuk routing traffic web port 80 ke frontend. |
| **[ha-patch.yaml](file:///home/vrs/Projects/rancher/microservices-demo/ha-patch.yaml)** | Patch Pod Anti-Affinity agar replika frontend terbagi ke `node-01` & `node-02`. |

---

## 3. Instalasi ArgoCD Menggunakan Helm Chart Resmi

Berikut adalah langkah implementasi instalasi ArgoCD menggunakan Helm chart resmi `argo/argo-cd`:

### Langkah 3.1: Tambahkan Helm Repository Argo

Jalankan perintah ini di workstation / mesin `dev-machine`:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

---

### Langkah 3.2: Konfigurasi Values Teroptimasi (`argocd/values-argocd.yaml`)

File **[argocd/values-argocd.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/values-argocd.yaml)** telah disiapkan dengan konfigurasi:
1. `configs.params."server.insecure": true`: Menghindari issue SSL handshake / error 502 pada Ingress Traefik dan mempermudah akses via NodePort.
2. `server.service.type: NodePort` dengan `nodePortHttp: 30085`: Otomatis membuka port 30085 pada semua node worker.
3. Penonaktifan Dex dan Notifications untuk menghemat ~180 MB RAM.

Isi ringkas file:
```yaml
global:
  domain: argocd.local

configs:
  params:
    server.insecure: true

server:
  service:
    type: NodePort
    nodePortHttp: 30085
    nodePortHttps: 30443
  resources:
    requests: { cpu: 50m, memory: 100Mi }
    limits: { cpu: 150m, memory: 200Mi }

repoServer:
  resources:
    requests: { cpu: 100m, memory: 150Mi }
    limits: { cpu: 250m, memory: 350Mi }

controller:
  resources:
    requests: { cpu: 150m, memory: 250Mi }
    limits: { cpu: 300m, memory: 500Mi }

redis:
  resources:
    requests: { cpu: 25m, memory: 64Mi }
    limits: { cpu: 50m, memory: 128Mi }

dex:
  enabled: false

notifications:
  enabled: false
```

---

### Langkah 3.3: Eksekusi Instalasi Helm ArgoCD

Jalankan perintah berikut:

```bash
# Jika dijalankan dari dev-machine:
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f argocd/values-argocd.yaml
```

Verifikasi seluruh pod ArgoCD telah berjalan:
```bash
kubectl get pods -n argocd -o wide
```

---

### Langkah 3.4: Ambil Password Awal Login Admin

Password awal dibuat otomatis oleh ArgoCD ke dalam Secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
* **Username**: `admin`
* **Password**: *(Output dari perintah di atas)*

---

## 4. Panduan Akses Web UI ArgoCD & Expose

Karena Service `argocd-server` sudah otomatis dikonfigurasi sebagai **NodePort 30085** oleh Helm values di atas, Anda dapat mengeksposnya ke publik secara langsung:

### Cara 1: Expose via NodePort (Port 30085) - Direkomendasikan
```bash
# Generate link HTTPS publik melalui worker node (node-01)
labctl expose port $PLAY_ID 30085 --machine node-01 --public
```
Salin link yang muncul di browser, lalu login dengan username `admin` dan password di Langkah 3.4.

---

### Cara 2: Expose via Port-Forward (Alternatif)
```bash
# Jalankan port-forward di background pada dev-machine
labctl ssh -m dev-machine $PLAY_ID -- \
    "kubectl port-forward --address 0.0.0.0 -n argocd svc/argocd-server 8085:80 > /tmp/pf-argocd.log 2>&1 &"

# Expose port 8085 ke publik
labctl expose port $PLAY_ID 8085 --machine dev-machine --public
```

---

## 5. Deployment Online Boutique via GitOps ArgoCD

Setelah ArgoCD aktif, deploy Online Boutique dengan mendaftarkan manifest **[argocd/argocd-application.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/argocd-application.yaml)**:

```bash
kubectl apply -f argocd/argocd-application.yaml
```

ArgoCD akan otomatis:
1. Melakukan clone dari repositori `https://github.com/Alvarras/microservices-on-k3s-cluster.git`.
2. Men-deploy Helm chart `./helm-chart` dengan nilai dari `values-k3s.yaml`.
3. Menjalankan seluruh 11 microservices di namespace `default` pada kedua worker node.

---

## 6. Panduan Expose Frontend Toko Online Boutique

### Expose via NodePort (Port 30080)
```bash
# 1. Terapkan service NodePort frontend
kubectl apply -f frontend-nodeport.yaml

# 2. Expose port 30080 ke publik
labctl expose port $PLAY_ID 30080 --machine node-01 --public
```
Buka URL publik yang dihasilkan untuk mengakses toko online.

---

## 7. Ringkasan Perintah Cepat (Cheat Sheet)

```bash
# 1. Tambah Helm repo Argo & update
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 2. Install ArgoCD via Helm
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace -f argocd/values-argocd.yaml

# 3. Dapatkan initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# 4. Expose ArgoCD UI (NodePort 30085)
labctl expose port $PLAY_ID 30085 --machine node-01 --public

# 5. Deploy Online Boutique via GitOps
kubectl apply -f argocd/argocd-application.yaml

# 6. Expose Online Boutique Frontend (NodePort 30080)
kubectl apply -f frontend-nodeport.yaml
labctl expose port $PLAY_ID 30080 --machine node-01 --public
```
