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
| **[argocd/values-argocd.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/values-argocd.yaml)** | Konfigurasi Helm resmi `argo/argo-cd` teroptimasi (insecure mode, NodePort 30085, registrasi otomatis repo Git). |
| **[argocd/repository.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/repository.yaml)** | Manifest Secret deklaratif Kubernetes untuk mendaftarkan repo GitHub ke ArgoCD. |
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

### Langkah 3.2: Eksekusi Instalasi Helm ArgoCD

Jalankan perintah berikut:

```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f argocd/values-argocd.yaml
```

Verifikasi seluruh pod ArgoCD telah berjalan:
```bash
kubectl get pods,svc -n argocd -o wide
```

---

### Langkah 3.3: Ambil Password Awal Login Admin

Password awal dibuat otomatis oleh ArgoCD ke dalam Secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
* **Username**: `admin`
* **Password**: *(Output dari perintah di atas)*

---

## 4. Cara Mendaftarkan Repositori Git di ArgoCD (Connecting Repositories)

Agar ArgoCD dapat membaca chart dan manifest dari `https://github.com/Alvarras/microservices-on-k3s-cluster.git`, repositori harus terdaftar. Anda dapat memilih salah satu metode di bawah ini:

### Metode 1: Deklaratif via Kubernetes Secret (Direkomendasikan - GitOps Standard)

Terapkan file **[argocd/repository.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/repository.yaml)**:

```bash
kubectl apply -f argocd/repository.yaml
```

Isi dari file `repository.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-microservices
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/Alvarras/microservices-on-k3s-cluster.git
  name: microservices-on-k3s-cluster
  # Jika repository bersifat Private (Privat di GitHub):
  # username: Alvarras
  # password: <GITHUB_PERSONAL_ACCESS_TOKEN_ghp_xxxx>
```

---

### Metode 2: Melalui Web UI ArgoCD (GUI)

1. Buka Web UI ArgoCD di browser Anda: [https://6af8199cfd7048c0a5ce1f69-710995.node-ap-c7ae.iximiuz.com/](https://6af8199cfd7048c0a5ce1f69-710995.node-ap-c7ae.iximiuz.com/)
2. Login dengan `admin` dan password admin Anda.
3. Klik menu **Settings** (ikon gerigi pada bilah navigasi kiri).
4. Pilih **Repositories** -> Klik tombol **+ CONNECT REPO**.
5. Isi formulir pendaftaran:
   * **Choose connection method**: `VIA HTTPS`
   * **Type**: `git`
   * **Repository URL**: `https://github.com/Alvarras/microservices-on-k3s-cluster.git`
   * *(Opsional)* Jika repo privat: isi **Username** (`Alvarras`) dan **Password** (GitHub Personal Access Token `ghp_...`).
6. Klik tombol **CONNECT**.
7. Status koneksi akan bertanda centang hijau: **`Successful`**.

---

### Metode 3: Melalui ArgoCD CLI

```bash
# Login terlebih dahulu
argocd login 172.16.0.3:30085 --plaintext --username admin --password <PASSWORD>

# Daftarkan repositori
argocd repo add https://github.com/Alvarras/microservices-on-k3s-cluster.git

# Verifikasi status repositori
argocd repo list
```

Output yang diharapkan:
```text
TYPE  NAME                          REPO                                                          STATUS      MESSAGE
git   microservices-on-k3s-cluster  https://github.com/Alvarras/microservices-on-k3s-cluster.git  Successful
```

---

## 5. Deployment Online Boutique via GitOps ArgoCD

Setelah repository terhubung, deploy Online Boutique dengan mendaftarkan manifest **[argocd/argocd-application.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/argocd-application.yaml)**:

```bash
kubectl apply -f argocd/argocd-application.yaml
```

ArgoCD akan otomatis:
1. Melakukan sync dari repositori `https://github.com/Alvarras/microservices-on-k3s-cluster.git`.
2. Men-deploy Helm chart `./helm-chart` dengan nilai dari `values-k3s.yaml`.
3. Menjalankan seluruh 11 microservices di namespace `default` pada kedua worker node.

---

## 6. Panduan Akses Web UI ArgoCD & Expose

### Akses Web UI ArgoCD (NodePort 30085)
```bash
labctl expose port $PLAY_ID 30085 --machine node-01 --public
```

* **URL Web UI (Live)**: [https://6af8199cfd7048c0a5ce1f69-710995.node-ap-c7ae.iximiuz.com/](https://6af8199cfd7048c0a5ce1f69-710995.node-ap-c7ae.iximiuz.com/)
* **Username**: `admin`
* **Password**: `G4FnzerxQe3Gbvh1`

---

### Expose Frontend Toko Online Boutique (NodePort 30080)
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

# 4. Daftarkan Repository Git
kubectl apply -f argocd/repository.yaml

# 5. Expose ArgoCD UI (NodePort 30085)
labctl expose port $PLAY_ID 30085 --machine node-01 --public

# 6. Deploy Online Boutique via GitOps
kubectl apply -f argocd/argocd-application.yaml

# 7. Expose Online Boutique Frontend (NodePort 30080)
kubectl apply -f frontend-nodeport.yaml
labctl expose port $PLAY_ID 30080 --machine node-01 --public
```
