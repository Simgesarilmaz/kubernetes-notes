# 03 - Kurulum (Setup)

> Bu not, kubeadm ile **iki makineli** (1 master + 1 worker) gerçek bir cluster
> kurulumunu birebir izler. Sürüm: **Kubernetes v1.36**.
> Resmî kaynak: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

---

## 📝 Özet

Cluster, bir **master (control plane)** ile bir veya daha fazla **worker node**'un birlikte oluşturduğu bütündür. Bu notta
iki makine hazırlanır, ikisine de aynı temel bileşenler kurulur, master'da cluster
başlatılır (`kubeadm init`), ağ eklentisi (Calico) yüklenir ve worker `kubeadm join`
ile cluster'a katılır.

### ⚠️ En kritik iki kural
1. **Sürüm tutarlılığı:** kubelet + kubeadm + kubectl + cluster **aynı minor sürümde**
   olmalı. Bu notta **v1.36** kullanılır. (Daha önce farklı sürüm kurduysan,
   aşağıdaki apt adımı `hold` ile 1.36'ya sabitler.)
2. **Adım hangi makinede çalışır?** Aşağıda her bölümün başında yazıyor:
   `[HER İKİ MAKİNE]`, `[SADECE MASTER]` veya `[SADECE WORKER]`.

---

## 🔑 Anahtar Kavramlar

- **Cluster** = master + worker node'lar (birlikte).
- **Container runtime (containerd):** container'ları fiilen çalıştıran katman.
- **CNI (Calico):** pod'ların birbiriyle haberleşmesini sağlayan ağ eklentisi.
- **kubeadm / kubelet / kubectl:** kurulumun üç temel paketi.
- **`kubeadm init`:** master'da control plane'i başlatır.
- **`kubeadm join`:** worker'ı cluster'a katar.
- **taint:** bir node'a "buraya normal iş yükü koyma" işareti.

---

## 💡 Notlarım

### Hangi adım nerede çalışır?
| Adım | Nerede |
|------|--------|
| 0. Makine isterleri + hostname | Her iki makine |
| 1. Kernel modülleri + swap | Her iki makine |
| 2. containerd | Her iki makine |
| 3. kubeadm/kubelet/kubectl | Her iki makine |
| 4. `kubeadm init` + kubeconfig + Calico | Sadece master |
| 5. `kubeadm join` | Sadece worker |

### `kubeadm init` parametreleri
- `--pod-network-cidr=192.168.0.0/16` → Calico'nun varsayılan aralığı. CNI olarak
  Calico kullandığımız için bu değer seçilir.
- `--apiserver-advertise-address=<ip>` → master'ın **IP'si** yazılır.
- `--control-plane-endpoint=<ip>` → API server'a erişim adresi (master IP'si).
- `<ip>` yazan yerleri kendi master IP'nle değiştir; olduğu gibi bırakma.

### Taint kaldırma ne işe yarar?
Normalde master node'una senin uygulama pod'ların yerleştirilmez; master bir
`taint` (leke) ile korunur. Şu komut o taint'i kaldırır (sonundaki `-` = "kaldır"):
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
Bu, master'ın da uygulama çalıştırmasına izin verir. **Tek makineli / lab** kurulumunda
gereklidir. Bizimki gibi ayrı worker'ı olan kurulumda **şart değildir** (ama zararı yok).

### containerd'yi Docker deposundan kurmak
Ubuntu'nun kendi paketi yerine Docker'ın deposundan `containerd.io` kurulur —
işlev aynı, sadece daha güncel paket.

---

## 💻 Komutlar / YAML

### 0) Makine isterleri + hostname  `[HER İKİ MAKİNE]`

İki sanal makine (biri master, biri worker) hazırla. Her biri için minimum:

- **CPU:** 2 çekirdek
- **RAM:** 2 GB
- **Disk:** 10 GB
- **OS:** Ubuntu
- **Ağ:** iki makine birbirinin IP'sine erişebilmeli (aynı ağ)
- **Benzersiz hostname:** biri `master`, diğeri `node1`
- **swap:** kapatılacak (Adım 1'de yapılıyor)

```bash
# master makinede:
sudo hostnamectl set-hostname master
# worker makinede:
sudo hostnamectl set-hostname node1
```

### 1) Kernel modülleri + swap kapatma  `[HER İKİ MAKİNE]`

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

sudo swapoff -a
free -m
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 2) containerd kurulumu  `[HER İKİ MAKİNE]`

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y containerd.io
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl start containerd

sudo mkdir -p /etc/containerd
sudo su -
containerd config default | tee /etc/containerd/config.toml
exit

# systemd cgroup sürücüsünü aç (kubelet uyumu için ŞART)
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### 3) kubeadm / kubelet / kubectl kurulumu  `[HER İKİ MAKİNE]`

```bash
# Firewall açıksa gerekli portlar (kapalıysa gerekmez, zararı yok)
sudo ufw allow 6443/tcp        # API server
sudo ufw allow 2379:2380/tcp   # etcd
sudo ufw allow 10250/tcp       # kubelet
sudo ufw allow 10259/tcp       # scheduler
sudo ufw allow 10257/tcp       # controller-manager

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p -m 755 /etc/apt/keyrings

# --- Sürüm: v1.36 (her iki satırda da) ---
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

> **Not:** Daha önce başka bir sürüm (ör. brew ile) kurmuştun; bu adım hepsini
> 1.36'ya hizalar ve `hold` ile sabitler. Karışıklık olmaması için tek sürümde kal.

### 4) Cluster'ı başlat  `[SADECE MASTER]`

```bash
sudo kubeadm config images pull

# <ip> yerine master'ın GERÇEK IP'sini yaz
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<ip> \
  --control-plane-endpoint=<ip>
```

`kubeadm init` başarılı olunca ekranın sonunda bir **`kubeadm join ...`** komutu
basar — **worker için lazım, kopyala/kaydet.**

kubectl'i normal kullanıcı için ayarla (master'da):
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Ağ eklentisi (Calico) kur (master'da):
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
```

> ⚠️ **Sürüm uyumu:** Calico v3.25.1 eski bir sürümdür ve k8s 1.36 ile uyumsuz
> olabilir. Node'lar `NotReady` kalırsa ya da Calico pod'ları hata verirse, güncel
> bir Calico sürümü kullan: https://docs.tigera.io/calico/latest/getting-started/kubernetes/

(İsteğe bağlı — tek makine gibi kullanmak istersen taint kaldır):
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 5) Worker'ı cluster'a kat  `[SADECE WORKER = node1]`

```bash
# master'daki 'kubeadm init' çıktısındaki komut; örneğin:
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

> Join komutunu kaybettiysen master'da yeniden üret:
> ```bash
> kubeadm token create --print-join-command
> ```

### 6) Doğrulama  `[SADECE MASTER]`

```bash
kubectl get nodes          # master + node1 görünmeli, ikisi de Ready
kubectl get pods -A        # sistem + Calico pod'ları Running
kubectl cluster-info
```

---

## ❓ Anlamadıklarım / Tekrar

- **Hangi adım nerede çalışıyor?**
  → 1-2-3 her iki makinede; `kubeadm init`/Calico sadece master; `kubeadm join`
  sadece worker.
  
- **Yeni cluster kurunca ağ neden ayrıca kuruluyor? (CNI eklentisi)**
  → Kubernetes pod'lar arası ağı kendisi kurmaz; bunu bilerek bir **eklentiye (CNI)**
  bırakır. `kubeadm init` control plane'i verir ama ağı vermez.
  → Bu yüzden bir CNI (Calico / Flannel / Cilium...) kurmak **şarttır**. CNI olmadan
  node'lar `NotReady` kalır.
  → CNI **sadece master'da bir kez** kurulur; DaemonSet olduğu için `join` edilen her
  worker'a otomatik yayılır (worker'a ayrıca kurmaya gerek yok).
  → `--pod-network-cidr` ile CNI'nin adres havuzu uyumlu olmalı (Calico → 192.168.0.0/16).

- **Hata ayıklama: apiserver başlamazsa nereye bakarım?**
  → Önce temizlik: `sudo kubeadm reset -f`
  → Log: `sudo journalctl -xeu kubelet --no-pager | tail -n 60`
  → Kontroller: `free -m` (Swap=0), `sudo grep SystemdCgroup /etc/containerd/config.toml` (true),
  `ip a | grep <ip>` (IP doğru mu).

- **`kubeadm join` komutunu nereden alacağım?**
  → `kubeadm init` çıktısının sonunda basılır. Kaçırdıysan:
  `kubeadm token create --print-join-command`.

- **Taint kaldırma şart mı?**
  → Ayrı worker'ı olan kurulumda hayır. Tek makineli/lab kurulumunda evet.

- **Node neden `NotReady`?**
  → Genelde CNI (Calico) henüz hazır değildir. Calico pod'ları Running olunca
  node `Ready` olur. Olmuyorsa Calico sürüm uyumunu kontrol et.

- **kubectl / kubeadm / kubelet farkı?**
  → `kubectl` = komut gönderen istemci. `kubeadm` = cluster'ı kuran araç.
  `kubelet` = her node'da pod'ları başlatan systemd servisi.
