# 04 - kubectl

---

## 📝 Özet

`kubectl`, Kubernetes cluster'ını yönetmek için kullanılan **komut satırı istemcisidir
(client)**. Yazdığın her komutu, cluster'ın giriş kapısı olan **kube-apiserver**'a
gönderir; API server da isteği işleyip cevabı geri döndürür.

kubectl cluster'ın bir parçası değildir — sadece ona "konuşan" araçtır. Hangi
cluster'a, hangi kullanıcı/yetkiyle bağlanacağını **kubeconfig** dosyasından okur.

İki çalışma tarzı vardır:
- **Imperative :** "Şunu şimdi yap" (`kubectl create`, `kubectl run`).
- **Declarative :** "İstenen durum bu, sen uydur" (`kubectl apply -f dosya.yaml`).
  Gerçek projelerde tercih edilen budur.

---

## 🔑 Anahtar Kavramlar (dinlerken dikkat et)

- **kubeconfig:** Bağlantı bilgilerinin tutulduğu dosya (varsayılan: `~/.kube/config`).
- **context:** "Hangi cluster + hangi kullanıcı + hangi namespace" üçlüsü. Aralarında geçiş yapılır.
- **namespace:** Cluster içinde mantıksal bölme/klasör. Kaynakları gruplar.
- **resource (kaynak):** Yönettiğin nesneler — pod, deployment, service, configmap...
- **imperative vs declarative:** Tek seferlik komut mu, YAML ile istenen durum mu?
- **Komut yapısı:** `kubectl [komut] [tip] [ad] [flag]`

---

## 💡 Notlarım

### kubectl nasıl çalışır?
```
kubectl  →  kubeconfig'i okur  →  kube-apiserver'a HTTP isteği  →  cevap
```
- kubeconfig yoksa ya da yanlışsa bağlanamaz (klasik `connection refused` hatası).

### Komut yapısı
```
kubectl [KOMUT] [TİP] [AD] [FLAG'LER]
         get     pods   my-pod  -n dev
```
- **KOMUT:** ne yapılacak (get, describe, create, apply, delete, logs...).
- **TİP:** kaynak türü (pod, deployment, svc, node...).
- **AD:** belirli bir kaynağın adı (opsiyonel).
- **FLAG:** ek seçenekler (`-n namespace`, `-o yaml`, `--all-namespaces`...).

### kubeconfig ve context
- Varsayılan konum: `~/.kube/config` (ya da `$KUBECONFIG` değişkeni).
- Bir context = **cluster + user + namespace** kombinasyonu.
- Birden fazla cluster'la çalışıyorsan context'ler arasında geçiş yaparsın.

### namespace
- Kaynakları izole eden mantıksal bölmelerdir (örn. `dev`, `prod`, `kube-system`).
- Belirtmezsen `default` namespace kullanılır.
- Tüm namespace'lerde görmek için: `-A` / `--all-namespaces`.

### Imperative vs Declarative
| | Imperative | Declarative |
|--|-----------|-------------|
| Nasıl | `kubectl create/run...` | `kubectl apply -f file.yaml` |
| Mantık | "Şunu yap" | "İstenen durum bu" |
| Takip | Elle | YAML dosyaları (Git'te tutulabilir) |
| Kullanım | Hızlı deneme | Gerçek projeler ✅ |

### Kısa adlar (kısaltmalar)
`pods→po`, `services→svc`, `deployments→deploy`, `namespaces→ns`,
`nodes→no`, `configmaps→cm`, `replicasets→rs`.

---

## 💻 Komutlar / YAML

### Doğrulama & küme bilgisi
```bash
kubectl version                 # client (+ server) sürümü
kubectl cluster-info            # cluster erişim bilgisi
kubectl get nodes               # node'lar ve durumları (Ready?)
kubectl get nodes -o wide       # daha fazla detay (IP, OS, runtime)
```

### Namespace
```bash
kubectl get namespaces          # ns'leri listele
kubectl get pods -n kube-system # belirli ns'deki pod'lar
kubectl get pods -A             # tüm ns'lerdeki pod'lar
```

### Listeleme & inceleme
```bash
kubectl get pods                        # pod'ları listele
kubectl get pods -o wide                # + node/IP bilgisi
kubectl get deploy,svc                  # birden çok tipi birden
kubectl describe pod <ad>               # bir kaynağın tüm detayı + olaylar
kubectl get pod <ad> -o yaml            # kaynağın tam YAML tanımı
```

### Oluşturma
```bash
# Imperative (hızlı)
kubectl run nginx --image=nginx
kubectl create deployment web --image=nginx --replicas=3

# Declarative (önerilen)
kubectl apply -f deployment.yaml
kubectl apply -f ./klasor/               # klasördeki tüm YAML'lar
```

### Log & container'a girme
```bash
kubectl logs <pod>                      # pod loglarını gör
kubectl logs <pod> -f                   # canlı takip
kubectl logs <pod> -c <container>       # çok container'lı pod'da belirli container
kubectl exec -it <pod> -- /bin/bash     # container içinde shell aç
```

### Ölçekleme & güncelleme
```bash
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl rollout undo deployment/web     # önceki sürüme dön
```

### Silme
```bash
kubectl delete pod <ad>
kubectl delete -f deployment.yaml
kubectl delete deployment web
```

### Context yönetimi
```bash
kubectl config get-contexts             # tüm context'ler
kubectl config current-context          # aktif olan
kubectl config use-context <ad>         # context değiştir
kubectl config set-context --current --namespace=dev   # varsayılan ns ayarla
```

### Faydalı flag'ler
```bash
-o wide           # ek sütunlar
-o yaml / -o json # tam tanım
-A                # tüm namespace'ler
-w                # değişiklikleri canlı izle (watch)
--dry-run=client -o yaml   # komutu çalıştırmadan YAML üret (şablon için ideal)
```

> **İpucu:** YAML yazmayı hızlandırmak için imperative komuttan şablon üret:
> ```bash
> kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
> ```

### Alias (bashrc'ye eklenebilir)
```bash
echo 'alias k=kubectl' >> ~/.bashrc
source ~/.bashrc
# artık: k get pods
```

### Örnek YAML — basit Deployment + Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
```bash
kubectl apply -f web.yaml     # ikisini birden oluşturur
```

---

## ❓ Anlamadıklarım / Tekrar

- **`get` ile `describe` farkı?**
  → `get` = özet liste (tek satır). `describe` = tek kaynağın tüm detayı + olay geçmişi
  (event'ler). Sorun ararken `describe` kullanılır.

- **Imperative mi declarative mi kullanmalıyım?**
  → Hızlı deneme/öğrenme için imperative, gerçek projelerde declarative (`apply -f`).
  Declarative YAML'ları Git'te tutup versiyonlayabilirsin.

- **kubeconfig nerede, nasıl değişir?**
  → Varsayılan `~/.kube/config`. `$KUBECONFIG` ile başka dosya gösterebilirsin.

- **`-n` ne zaman gerekli?**
  → Kaynak `default` dışı bir namespace'deyse. Belirtmezsen sadece `default`'a bakar.
  Her yeri görmek için `-A`.
