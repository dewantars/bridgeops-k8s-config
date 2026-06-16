# Arahan Integrasi & Setup untuk Anggota 4 (Ansible)

Dokumen ini berisi panduan langkah demi langkah bagi **Anggota 4 (Abdul)** untuk menjalankan playbook Ansible di laptopnya, mengaktifkan cluster Minikube di VM, dan mengintegrasikannya dengan manifest Kubernetes milik **Anggota 3 (Dewanta)** yang sudah ada di GitHub.

---

## Prasyarat di Laptop Anggota 4
Sebelum memulai, pastikan laptop Anggota 4 sudah memiliki:
1. **VirtualBox** terinstal.
2. **Ansible** terinstal (di laptop host, WSL, atau MacOS/Linux shell).
3. SSH Key-pair (jika belum ada, buat dengan perintah `ssh-keygen`).

---

## Langkah 1: Setup VM di VirtualBox
Karena playbook Ansible menggunakan package manager **`dnf`**, VM target harus menggunakan sistem operasi Linux keluarga Red Hat (sangat disarankan: **Rocky Linux 9** atau **AlmaLinux 9**).

1. Buat VM baru di VirtualBox dengan OS Rocky Linux 9 / AlmaLinux 9.
2. **Konfigurasi Network Adapter:**
   * Buka *Settings VM* -> *Network* -> *Adapter 1*.
   * Ubah *Attached to* menjadi **Host-only Adapter** (atau **Bridged Adapter**).
   * Jalankan VM, ketik perintah `ip a` untuk melihat IP address VM (misal: `192.168.217.133`).
3. **Konfigurasi Akses SSH Tanpa Password:**
   * Di laptop host, kirim public key SSH ke VM agar Ansible bisa login otomatis:
     ```bash
     ssh-copy-id -i ~/.ssh/id_rsa.pub sisadul@192.168.217.133
     ```
     *(Ganti `sisadul` dan IP `192.168.217.133` sesuai dengan username dan IP VM target).*

---

## Langkah 2: Persiapan Kode Ansible (di Repo `bridgeops-app`)
Semua kode Ansible sudah tersedia di repository `bridgeops-app` pada branch **`abdul/ansible`**.
1. Lakukan checkout ke branch tersebut di laptop Anggota 4:
   ```bash
   git checkout abdul/ansible
   ```
2. Buka folder `ansible/` dan sesuaikan file `inventory.ini` dengan IP VM Anda:
   ```ini
   [managed_hosts]
   192.168.217.133 ansible_user=sisadul ansible_ssh_private_key_file=~/.ssh/id_rsa
   ```

---

## Langkah 3: Jalankan Playbook Ansible
Jalankan perintah ini dari folder `ansible/` di laptop host Anggota 4:

1. **Tes Koneksi:**
   ```bash
   ansible -i inventory.ini managed_hosts -m ping
   ```
   *Pastikan statusnya menampilkan `"ping": "pong"` (berwarna hijau).*

2. **Jalankan Playbook Setup:**
   ```bash
   ansible-playbook -i inventory.ini playbook-setup.yml
   ```
   *Ansible akan otomatis menginstal Docker, Kubectl, Minikube, dan Helm di dalam VM Rocky Linux.*

---

## Langkah 4: Jalankan Minikube di VM
Setelah playbook selesai tanpa error:
1. Masuk ke VM target menggunakan SSH:
   ```bash
   ssh sisadul@192.168.217.133
   ```
2. Jalankan Minikube dengan driver Docker:
   ```bash
   minikube start --driver=docker
   ```
3. Pastikan cluster aktif:
   ```bash
   kubectl get nodes
   ```

---

## Langkah 5: Integrasi dengan Manifest Anggota 3 (GitOps)
Setelah Minikube aktif, deploy aplikasi menggunakan manifest milik Anggota 3 yang sudah di-push ke GitHub:

1. **Buat Namespace:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/dewantars/bridgeops-k8s-config/main/namespaces/myapp-namespace.yaml
   kubectl apply -f https://raw.githubusercontent.com/dewantars/bridgeops-k8s-config/main/namespaces/argocd-namespace.yaml
   kubectl apply -f https://raw.githubusercontent.com/dewantars/bridgeops-k8s-config/main/namespaces/monitoring-namespace.yaml
   ```

2. **Install ArgoCD (Menggunakan Helm yang telah dipasang Ansible):**
   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm install argocd argo/argo-cd -n argocd --create-namespace
   ```

3. **Deploy ArgoCD Application GitOps:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/dewantars/bridgeops-k8s-config/main/argocd/laravel-application.yaml
   ```

4. **Verifikasi GitOps:**
   * Tunjukkan ke dosen/penguji bahwa ArgoCD secara otomatis menarik seluruh manifest Laravel & PostgreSQL dari repository GitHub Anda dan men-deploy-nya ke dalam cluster Minikube di VM Rocky Linux.
