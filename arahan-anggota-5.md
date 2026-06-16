# Arahan Kerja & Setup untuk Anggota 5 (Prometheus Monitoring)

Dokumen ini berisi panduan langkah demi langkah bagi **Anggota 5** untuk memasang sistem monitoring Prometheus di cluster Kubernetes, menghubungkannya ke metrics Laravel, dan menyiapkan data source untuk **Anggota 6 (Grafana)**.

---

## Prasyarat Sebelum Mulai
Pastikan langkah-langkah sebelumnya telah diselesaikan:
1. Cluster Minikube sudah aktif (oleh Anggota 4).
2. Aplikasi Laravel dan database PostgreSQL sudah berjalan sehat di namespace `myapp` (oleh Anggota 3).
3. Namespace `monitoring` sudah dibuat.

---

## Langkah 1: Install Prometheus Stack Menggunakan Helm

Prometheus akan diinstal menggunakan **Kube-Prometheus-Stack**, sebuah bundel monitoring standar industri yang mencakup Prometheus, Alertmanager, Node Exporter, dan Grafana.

Jalankan perintah berikut di terminal Anda:

1. **Tambahkan Repositori Helm Prometheus:**
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   ```
2. **Update Repositori Helm:**
   ```bash
   helm repo update
   ```
3. **Instal Kube-Prometheus-Stack:**
   ```bash
   helm install monitoring prometheus-community/kube-prometheus-stack \
     --namespace monitoring \
     --create-namespace
   ```
4. **Verifikasi Pod Monitoring:**
   Tunggu hingga semua pod monitoring berstatus `Running`:
   ```bash
   kubectl get pods -n monitoring
   ```

---

## Langkah 2: Hubungkan Laravel ke Prometheus (ServiceMonitor)

Agar Prometheus mengetahui keberadaan pod Laravel dan mengambil data metrics dari endpoint `/metrics` setiap 15 detik, Anda perlu menerapkan **ServiceMonitor**.

1. **Gunakan File ServiceMonitor yang Sudah Dibuat Anggota 3:**
   File tersebut sudah tersedia di repositori `bridgeops-k8s-config` pada folder `monitoring/servicemonitor-laravel.yaml`.
2. **Terapkan ServiceMonitor:**
   ```bash
   kubectl apply -f monitoring/servicemonitor-laravel.yaml
   ```

> [!NOTE]
> **Mengapa ini berhasil?**
> ServiceMonitor ini mencari Service di namespace `myapp` yang memiliki label `app: laravel-app` (seperti yang telah dikonfigurasi di file `laravel-service.yaml` oleh Anggota 3).

---

## Langkah 3: Akses Dashboard UI Prometheus

1. **Jalankan Port Forward ke Service Prometheus:**
   ```bash
   kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
   ```
   *Biarkan terminal ini tetap terbuka.*
2. **Buka di Browser:**
   Akses **[http://localhost:9090](http://localhost:9090)**.

---

## Langkah 4: Verifikasi Target di Dashboard Prometheus

1. Di halaman utama Prometheus, klik menu **Status** -> **Targets**.
2. Cari section **`serviceMonitor/monitoring/laravel-servicemonitor/0`**.
3. Pastikan status target Laravel Anda berwarna hijau dengan tulisan **`UP`** (artinya Prometheus berhasil mendeteksi dan melakukan scrape metrics dari Laravel).

---

## Langkah 5: Uji Query PromQL

Jalankan query sederhana di halaman utama Prometheus (Tab **Graph**) untuk memverifikasi data masuk:

1. Ketik query: `up` lalu klik **Execute**.
   *Hasil harus menampilkan list seluruh target monitoring.*
2. Ketik query: `laravel_app_up` atau metrics default Laravel (jika package exporter Laravel aktif) untuk melihat status spesifik aplikasi Laravel Anda.

---

## Serah Terima ke Anggota 6 (Grafana)

Prometheus bertindak sebagai pengumpul data (Data Source), sedangkan **Anggota 6** bertugas membuat visualisasi (Dashboard) menggunakan Grafana.

Sebagai **Anggota 5**, Anda harus memberikan informasi berikut kepada **Anggota 6**:
1. **URL Data Source Prometheus:** 
   Di dalam cluster Kubernetes, Grafana dapat menghubungi Prometheus melalui DNS internal:
   `http://monitoring-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090`
2. **Credential Default Grafana:**
   Grafana sudah otomatis terpasang bersama Prometheus Stack. Untuk membukanya:
   * **Port Forward Grafana:**
     ```bash
     kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
     ```
   * **Akses Browser:** [http://localhost:3000](http://localhost:3000)
   * **Username:** `admin`
   * **Password:** Ambil password terenkripsi dari Secret Kubernetes dengan perintah:
     ```bash
     kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
     ```
