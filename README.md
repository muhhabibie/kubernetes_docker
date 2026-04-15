# High Availability Kubernetes Cluster (Capstone Project)

Proyek ini adalah implementasi infrastruktur High Availability (HA) menggunakan Kubernetes v1.31 di lingkungan lokal (VirtualBox). Sistem ini dirancang untuk menjamin ketersediaan layanan melalui distribusi beban kerja di 3 Worker Node dengan fitur pemulihan otomatis (Self-Healing) dan manajemen penyimpanan permanen (Persistent Volume).

---

## Anggota Kelompok & Pembagian Tugas

Catatan Pelaksanaan: Mengingat tingginya kebutuhan sumber daya komputasi (menjalankan 4 Virtual Machine secara bersamaan) serta besarnya ukuran file instalasi yang diperlukan, eksekusi teknis (instalasi dan running VM) difokuskan pada satu perangkat utama (perangkat Habibi). Oleh karena itu, kolaborasi dan penyusunan repository ini dilakukan secara terpusat tanpa riwayat commit multi-user di Git demi efisiensi proses pengerjaan.

* **Muhammad Habibi (235150201111063)**
    * Eksekutor Teknis: Melakukan inisialisasi High Availability Cluster (1 Master & 3 Workers) pada mesin virtual.
    * Integrasi Sistem: Melakukan instalasi Container Runtime `cri-dockerd` dan konfigurasi jaringan Calico CNI.
    * Manajemen Storage & Deployment: Mengonfigurasi Persistent Storage MySQL (PV/PVC) serta melakukan implementasi strategi deployment 3 replika aplikasi ke dalam klaster.

* **Nadhif Rif'at Rasendriya (235150201111074)**
    * Konseptor Topologi & Arsitektur: Merancang struktur topologi jaringan lokal, alokasi IP statis untuk Control Plane dan Data Plane, serta menyusun strategi deployment.
    * Technical Writer & Documentation: Menyusun blueprint panduan instalasi (README.md) dan merumuskan dokumentasi analitis terhadap hasil implementasi sistem.
    * Quality Assurance & Troubleshooting: Membantu analisis pemecahan masalah (troubleshooting) secara konseptual selama proses integrasi node dan memvalidasi konfigurasi beban kerja (load balancing) pada Pods.

---

## Arsitektur Jaringan & Node

Cluster ini berjalan menggunakan jaringan lokal (FILKOM) dengan konfigurasi IP statis menggunakan Bridge Adapter pada VirtualBox:

| Node Name | Role | IP Address | Operating System |
| :--- | :--- | :--- | :--- |
| Master-Node | Control Plane | 192.168.1.13 | Ubuntu 22.04/24.04 |
| Worker-1 | Data Plane | 192.168.1.10 | Ubuntu 22.04/24.04 |
| Worker-2 | Data Plane | 192.168.1.11 | Ubuntu 22.04/24.04 |
| Worker-3 | Data Plane | 192.168.1.12 | Ubuntu 22.04/24.04 |

Persyaratan Utama:
* VirtualBox: Konfigurasi Network wajib menggunakan Bridge Adapter.
* Networking: Terkoneksi ke jaringan lokal (satu segmen yang sama).
* PENTING: Lakukan Disable Swap pada seluruh node sebelum instalasi menggunakan perintah: `sudo swapoff -a`

---

## Panduan Instalasi Cluster (Technical Setup)

### 1. Persiapan Docker Engine & Repository
Jalankan perintah ini di seluruh node (Master & Workers):

```bash
# Download GPG key & Add Repository
wget -O - [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) > ./docker.key
gpg --no-default-keyring --keyring ./docker.gpg --import ./docker.key
gpg --no-default-keyring --keyring ./docker.gpg --export > ./docker-archive-keyring.gpg
sudo mv ./docker-archive-keyring.gpg /etc/apt/trusted.gpg.d/

sudo add-apt-repository "deb [arch=amd64] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(lsb_release -cs) stable" -y
sudo apt update -y
sudo apt install git wget curl socat docker-ce -y
````

### 2\. Instalasi cri-dockerd (Docker CRI Support)

Jalankan di seluruh node untuk menyediakan dukungan Container Runtime Interface (CRI):

```bash
VER=$(curl -s [https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest](https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest)|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
wget [https://github.com/Mirantis/cri-dockerd/releases/download/v$](https://github.com/Mirantis/cri-dockerd/releases/download/v$){VER}/cri-dockerd-${VER}.amd64.tgz
tar xzvf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

# Setup Systemd Service
wget [https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service](https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service)
wget [https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket](https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket)
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
```

### 3\. Instalasi Kubernetes v1.31 & Optimasi System

Jalankan di seluruh node:

```bash
# Add Kubernetes Repo
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.31/deb/](https://pkgs.k8s.io/core:/stable:/v1.31/deb/) /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold docker-ce kubelet kubeadm kubectl

# Enable iptables bridge & IP Forwarding
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Disable Swap (Mandatory)
sudo swapoff -a
```

### 4\. Cluster Initialization & Calico Networking

Jalankan hanya di Master Node (192.168.1.13):

```bash
sudo kubeadm init --apiserver-advertise-address=192.168.1.13 --cri-socket unix:///var/run/cri-dockerd.sock --pod-network-cidr=10.244.0.0/16

# Konfigurasi Akses Kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico Network
kubectl create -f [https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml)
curl [https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml) -O
kubectl create -f custom-resources.yaml
```

### 5\. Monitoring & Dashboard Setup

Jalankan hanya di Master Node:

```bash
# Install Metrics Server
kubectl apply -f [https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml](https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml)

# Install Helm & Dashboard
curl -fsSL -o get_helm.sh [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3)
chmod +x get_helm.sh && ./get_helm.sh
helm repo add kubernetes-dashboard [https://kubernetes.github.io/dashboard/](https://kubernetes.github.io/dashboard/)
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

# Ekspos Dashboard (NodePort)
kubectl expose deployment kubernetes-dashboard-kong --name k8s-dash-svc --type NodePort --port 443 --target-port 8443 -n kubernetes-dashboard

# Token Akses (Admin User)
kubectl apply -f k8s-dash.yaml
kubectl create token widhi -n kube-system
```

-----

## Deployment Aplikasi (k8s-login-app)

Aplikasi dideploy dengan 3 Replicas yang disebar secara otomatis oleh Kube-Scheduler ke ketiga Worker Node untuk menjamin High Availability.

Langkah Deployment:

```bash
cd k8s-login-app/k8s
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-pv.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f web-service.yaml
kubectl apply -f web-deployment.yaml
```

-----

## Bukti Implementasi dan Analisis

Bagian ini memuat dokumentasi visual dari implementasi klaster Kubernetes yang telah dikonfigurasi, beserta penjelasan fungsional dari setiap komponen.

### 1. Status Node Klaster

![Get Nodes](screenshots/get%20nodes%20running.png)

**Penjelasan:** Tangkapan layar dari eksekusi perintah `kubectl get nodes` membuktikan bahwa klaster telah berhasil diinisialisasi. Terdapat satu `control-plane` (Master) dan tiga `worker` node yang semuanya berstatus `Ready`. Hal ini menunjukkan bahwa instalasi kubelet, kubeadm, dan CNI (Calico) telah terintegrasi dengan baik pada keempat mesin virtual.

### 2. Distribusi Beban Kerja (High Availability)

![Get Pods](screenshots/get%20pods.png)

**Penjelasan:** Eksekusi perintah `kubectl get pods -o wide` menunjukkan implementasi *High Availability* yang berhasil. Terdapat tiga replika pod dari `login-app` yang berjalan (`Running`). Kube-scheduler telah berhasil mendistribusikan ketiga pod tersebut secara merata ke `worker-1`, `worker-2`, dan `worker-3`. Jika salah satu worker mati, aplikasi akan tetap berjalan di worker lainnya.

### 3. Ketersediaan Docker Images

![Docker Images](screenshots/Docker%20Images.png)

**Penjelasan:** Perintah `sudo docker images` menampilkan daftar *image* kontainer yang ditarik dan dikelola di dalam node. Terlihat *image* esensial Kubernetes seperti `kube-apiserver`, `kube-controller-manager`, `etcd`, dan `coredns`, serta *image* dari Calico (`calico/cni`, `calico/node`) yang berfungsi untuk mengatur jaringan antar pod.

### 4. Ekspos Layanan (Services dan NodePort)

![Nodes Port Status](screenshots/Nodes%20port%20status.png)

**Penjelasan:** Melalui `kubectl get svc`, dapat diverifikasi bahwa layanan telah terekspos dengan benar. Database `mysql` dikonfigurasi menggunakan tipe `ClusterIP` agar hanya dapat diakses secara internal dalam klaster untuk keamanan. Sementara itu, `login-app` diekspos menggunakan tipe `NodePort` pada port `32121`, sehingga aplikasi web ini dapat diakses dari luar klaster melalui IP jaringan lokal.

### 5. Aksesibilitas Aplikasi via Worker Nodes

Ketiga tangkapan layar di bawah ini adalah bukti pengujian akses aplikasi melalui *browser* menggunakan IP dari masing-masing Worker Node dengan port `32121`.

![Worker 1](screenshots/Worker%201.png)

**Penjelasan Worker 1:** Aplikasi berhasil diakses melalui IP Worker-1 (`192.168.1.10:32121`). Halaman "Image Upload Dashboard" termuat dengan sempurna.

![Worker 2](screenshots/Worker%202.png)

**Penjelasan Worker 2:** Aplikasi juga berhasil diakses melalui IP Worker-2 (`192.168.1.11:32121`). Ini membuktikan bahwa mekanisme NodePort merutekan trafik dengan benar meskipun diakses dari node yang berbeda.

![Worker 3](screenshots/Worker%203.png)

**Penjelasan Worker 3:** Serupa dengan node lainnya, akses melalui IP Worker-3 (`192.168.1.12:32121`) berhasil menampilkan antarmuka aplikasi. Ketersediaan akses di seluruh worker membuktikan bahwa konfigurasi *routing* klaster dan *load balancing* telah berjalan sesuai dengan prinsip desain *High Availability*.