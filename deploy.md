# Panduan Migrasi, Deployment & Expose Online Boutique pada K3s Cluster (Mendukung GitOps dengan ArgoCD)

Dokumen ini adalah panduan lengkap dan *step-by-step* untuk mendeploy dan menjalankan aplikasi **Google Online Boutique (`microservices-demo`)** pada klaster **K3s (3 VM: 1 Control Plane & 2 Worker Nodes)** menggunakan repositori:
**`https://github.com/Alvarras/microservices-on-k3s-cluster.git`**

Panduan ini mencakup:
1. Analisis kapasitas klaster & kelayakan penambahan ArgoCD.
2. Langkah eksekusi deployment (Helm langsung atau GitOps via ArgoCD).
3. **Panduan Lengkap Expose Aplikasi** (Menggunakan **NodePort** dan **Port-Forward** terintegrasi dengan `labctl` / iximiuz playground, serta Traefik Ingress).

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

### 1.2 Analisis Kelayakan Workload Online Boutique + ArgoCD

```text
1. OS Linux + K3s Agent (2 Worker Nodes) : ~0.60 vCPU | ~1.40 GB RAM
2. Online Boutique Baseline Requests     : ~1.57 vCPU | ~1.31 GB RAM
3. ArgoCD Stack Requests                 : ~0.38 vCPU | ~0.66 GB RAM
-----------------------------------------------------------------------
TOTAL TERPAKAI (Allocated Requests)      : ~2.55 vCPU | ~3.37 GB RAM (~34% Utilisasi)
TOTAL KAPASITAS WORKER LAYER             :  6.00 vCPU | 10.00 GB RAM
=======================================================================
SISA KAPASITAS BEBAS (HEADROOM)          : ~3.45 vCPU | ~6.63 GB RAM (~66% SISA BEBAS!)
```

> [!IMPORTANT]
> **Kesimpulan:** Klaster Anda **SANGAT MEMADAI**. Penambahan ArgoCD hanya mengonsumsi sebagian kecil resource, menyisakan lebih dari **6.5 GB RAM bebas** di layer worker.

---

## 2. File Konfigurasi Siap Pakai di Repositori

Repositori ini telah disiapkan dengan file-file pendukung:

| File | Fungsi |
|---|---|
| **[values-k3s.yaml](file:///home/vrs/Projects/rancher/microservices-demo/values-k3s.yaml)** | Konfigurasi Helm teroptimasi (platform `onprem`, non-aktifkan GCP Operations, resource tuning). |
| **[k3s-ingress.yaml](file:///home/vrs/Projects/rancher/microservices-demo/k3s-ingress.yaml)** | Traefik Ingress untuk routing traffic web port 80 ke frontend. |
| **[frontend-nodeport.yaml](file:///home/vrs/Projects/rancher/microservices-demo/frontend-nodeport.yaml)** | Service bertipe `NodePort` (port `30080`) untuk mengekspos frontend toko. |
| **[redis-cart-pvc.yaml](file:///home/vrs/Projects/rancher/microservices-demo/redis-cart-pvc.yaml)** | Persistensi data keranjang belanja menggunakan storage `local-path` K3s. |
| **[ha-patch.yaml](file:///home/vrs/Projects/rancher/microservices-demo/ha-patch.yaml)** | Patch Pod Anti-Affinity agar replika frontend terbagi ke `node-01` & `node-02`. |
| **[argocd/argocd-application.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/argocd-application.yaml)** | Manifest GitOps Application ArgoCD mengarah ke `https://github.com/Alvarras/microservices-on-k3s-cluster.git`. |
| **[argocd/argocd-ingress.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/argocd-ingress.yaml)** | Ingress Traefik untuk dashboard visual ArgoCD. |
| **[argocd/argocd-nodeport.yaml](file:///home/vrs/Projects/rancher/microservices-demo/argocd/argocd-nodeport.yaml)** | Service bertipe `NodePort` (port `30085` / `30443`) untuk akses langsung dashboard ArgoCD. |

---

## 3. Langkah demi Langkah Menjalankan Aplikasi

Jika Anda menggunakan lingkungan **iximiuz playground / labctl**, Anda memiliki playground K3s aktif (`dev-machine`, `cplane-01`, `node-01`, `node-02`).

### Langkah 3.1: Dapatkan ID Playground Aktif

Jika memulai sesi baru:
```bash
PLAY_ID=$(labctl playground start k3s)
echo "Playground ID: $PLAY_ID"
```

Jika playground sudah berjalan (cek dengan `labctl playground list`):
```bash
export PLAY_ID="6af8199cfd7048c0a5ce1f69"  # ganti dengan PLAY_ID Anda
```

---

### Langkah 3.2: Push Perubahan Kode ke Repositori GitHub Anda

Pastikan seluruh file baru telah tersimpan di repositori GitHub Anda:
```bash
cd /home/vrs/Projects/rancher/microservices-demo
git add .
git commit -m "feat: optimize for k3s cluster with argocd and nodeport manifests"
git branch -M main
git push -u origin main
```

---

### Langkah 3.3: Pilih Metode Deployment

Pilih salah satu metode deployment di bawah: **Metode A (GitOps via ArgoCD - Direkomendasikan)** atau **Metode B (Helm Langsung)**.

#### METODE A: Deployment Berbasis GitOps Menggunakan ArgoCD (Direkomendasikan)

1. **Instal ArgoCD di klaster K3s:**
   ```bash
   labctl ssh $PLAY_ID -- kubectl create namespace argocd
   labctl ssh $PLAY_ID -- kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Optimasi Mode Insecure (agar kompatibel dengan Ingress Traefik & NodePort):**
   ```bash
   labctl ssh $PLAY_ID -- kubectl patch configmap argocd-cmd-params-cm -n argocd --type merge -p '{"data":{"server.insecure":"true"}}'
   labctl ssh $PLAY_ID -- kubectl rollout restart deployment argocd-server -n argocd
   ```

3. **Deploy Online Boutique via ArgoCD:**
   Salin dan apply manifest `argocd/argocd-application.yaml`:
   ```bash
   labctl ssh $PLAY_ID -- kubectl apply -f https://raw.githubusercontent.com/Alvarras/microservices-on-k3s-cluster/main/argocd/argocd-application.yaml
   ```

4. **Ambil Password Login Awal ArgoCD:**
   ```bash
   labctl ssh $PLAY_ID -- kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
   ```
   *Username default:* `admin`

---

#### METODE B: Deployment Langsung Menggunakan Helm

Jika ingin mendeploy langsung tanpa melalui ArgoCD:

```bash
# 1. Masuk ke dev-machine atau clone repo di dev-machine
labctl ssh -m dev-machine $PLAY_ID -- "git clone https://github.com/Alvarras/microservices-on-k3s-cluster.git /tmp/app && cd /tmp/app && helm upgrade --install onlineboutique ./helm-chart -f values-k3s.yaml --namespace default"

# 2. Terapkan PVC Redis untuk persistensi keranjang belanja
labctl ssh $PLAY_ID -- kubectl apply -f https://raw.githubusercontent.com/Alvarras/microservices-on-k3s-cluster/main/redis-cart-pvc.yaml
```

---

## 4. Panduan Lengkap Mengekspos Aplikasi (Exposing Guide)

Untuk mengakses web aplikasi Online Boutique maupun Web UI ArgoCD dari browser lokal Anda, gunakan metode berikut:

---

### METODE 1: Expose Menggunakan NodePort (Direkomendasikan & Stabil)

NodePort membuka port spesifik (rentang 30000–32767) pada interface IP semua node worker (`node-01` atau `node-02`). Kemudian `labctl expose port` akan membuatkan URL publik HTTPS yang dapat diakses langsung dari browser Anda.

#### 1. Expose Frontend Toko Online Boutique (Port 30080)

1. Terapkan service NodePort:
   ```bash
   labctl ssh $PLAY_ID -- kubectl apply -f https://raw.githubusercontent.com/Alvarras/microservices-on-k3s-cluster/main/frontend-nodeport.yaml
   ```

2. Verifikasi service NodePort telah aktif:
   ```bash
   labctl ssh $PLAY_ID -- kubectl get svc frontend-nodeport -n default
   ```
   *Output contoh:*
   ```text
   NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
   frontend-nodeport   NodePort   10.43.120.45    <none>        80:30080/TCP   15s
   ```

3. Expose port `30080` pada salah satu worker node (`node-01`) menjadi URL publik:
   ```bash
   labctl expose port $PLAY_ID 30080 --machine node-01 --public
   ```
   *Output contoh:*
   ```text
   HTTP port node-01:30080 exposed as https://69a725671fce5dfccd59a92f-026f01.node-eu-d241.iximiuz.com
   https://69a725671fce5dfccd59a92f-026f01.node-eu-d241.iximiuz.com
   ```
   Buka URL tersebut di browser untuk mengakses **Online Boutique**.

---

#### 2. Expose Web UI ArgoCD (Port 30085)

1. Terapkan service NodePort untuk ArgoCD:
   ```bash
   labctl ssh $PLAY_ID -- kubectl apply -f https://raw.githubusercontent.com/Alvarras/microservices-on-k3s-cluster/main/argocd/argocd-nodeport.yaml
   ```

2. Verifikasi service ArgoCD NodePort:
   ```bash
   labctl ssh $PLAY_ID -- kubectl get svc argocd-server-nodeport -n argocd
   ```

3. Expose port `30085` pada node worker (`node-01`):
   ```bash
   labctl expose port $PLAY_ID 30085 --machine node-01 --public
   ```
   *Output:* Dihasilkan link HTTPS publik menuju dashboard ArgoCD. Anda dapat login dengan user `admin` dan initial password yang diambil di Langkah 3.3.

---

### METODE 2: Expose Menggunakan `kubectl port-forward`

Metode ini sangat fleksibel dan cepat untuk kebutuhan debugging atau pengujian sementara tanpa membuat service `NodePort`.

> [!IMPORTANT]
> **Kunci Sukses Port-Forward:**
> Perintah `kubectl port-forward` **WAJIB** menyertakan flag `--address 0.0.0.0` agar mendengarkan di semua interface jaringan VM `dev-machine`. Jika tidak, port hanya bisa diakses secara lokal (`127.0.0.1`) di dalam VM dan tidak bisa diekspos keluar oleh `labctl`.

#### 1. Port-Forward Frontend Online Boutique ke Port 8080 di `dev-machine`

1. Jalankan port-forward di background pada mesin `dev-machine`:
   ```bash
   labctl ssh -m dev-machine $PLAY_ID -- "kubectl port-forward --address 0.0.0.0 svc/frontend 8080:80 > /tmp/pf-frontend.log 2>&1 &"
   ```

2. Verifikasi port telah listen di `dev-machine`:
   ```bash
   labctl ssh -m dev-machine $PLAY_ID -- "curl -sI http://127.0.0.1:8080 | head -n 3"
   ```

3. Expose port `8080` dari `dev-machine` ke publik:
   ```bash
   labctl expose port $PLAY_ID 8080 --machine dev-machine --public
   ```
   *Output:* Salin URL publik yang dihasilkan dan buka di browser Anda.

---

#### 2. Port-Forward ArgoCD Server ke Port 8085 di `dev-machine`

1. Jalankan port-forward ArgoCD di background pada `dev-machine`:
   ```bash
   labctl ssh -m dev-machine $PLAY_ID -- "kubectl port-forward --address 0.0.0.0 -n argocd svc/argocd-server 8085:80 > /tmp/pf-argocd.log 2>&1 &"
   ```

2. Expose port `8085` dari `dev-machine` ke publik:
   ```bash
   labctl expose port $PLAY_ID 8085 --machine dev-machine --public
   ```
   *Output:* Buka URL publik yang dihasilkan untuk login ke dashboard ArgoCD.

---

### METODE 3: Expose Menggunakan Traefik Ingress Bawaan K3s

Karena K3s secara default menjalankan Traefik Ingress Controller yang mendengarkan di port `80` pada seluruh node:

1. Terapkan ingress Online Boutique:
   ```bash
   labctl ssh $PLAY_ID -- kubectl apply -f https://raw.githubusercontent.com/Alvarras/microservices-on-k3s-cluster/main/k3s-ingress.yaml
   ```

2. Expose langsung port `80` pada `node-01`:
   ```bash
   labctl expose port $PLAY_ID 80 --machine node-01 --public
   ```
   *Output:* Buka URL publik yang dihasilkan untuk mengakses toko online.

---

## 5. Ringkasan Perintah Penting (Cheat Sheet)

```bash
# Set Playground ID
export PLAY_ID="<PLAY_ID_ANDA>"

# Cek status node & pod
labctl ssh $PLAY_ID -- kubectl get nodes -o wide
labctl ssh $PLAY_ID -- kubectl get pods -A -o wide

# Monitor resource (CPU/Memory)
labctl ssh $PLAY_ID -- kubectl top nodes
labctl ssh $PLAY_ID -- kubectl top pods --sort-by=memory

# Expose Frontend Online Boutique via NodePort
labctl ssh $PLAY_ID -- kubectl apply -f https://raw.githubusercontent.com/Alvarras/microservices-on-k3s-cluster/main/frontend-nodeport.yaml
labctl expose port $PLAY_ID 30080 --machine node-01 --public

# Expose ArgoCD UI via NodePort
labctl ssh $PLAY_ID -- kubectl apply -f https://raw.githubusercontent.com/Alvarras/microservices-on-k3s-cluster/main/argocd/argocd-nodeport.yaml
labctl expose port $PLAY_ID 30085 --machine node-01 --public

# Ambil Password Admin ArgoCD
labctl ssh $PLAY_ID -- kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
