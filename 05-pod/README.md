# 05 - Pod

---

## 📝 Özet

**Pod**, Kubernetes'te oluşturup çalıştırabileceğin **en küçük birimdir.** Kubernetes
container'ları doğrudan çalıştırmaz; onları bir pod'un içine sarar ve pod'u yönetir.

Bir pod, **bir veya daha fazla container** barındırır. Aynı pod içindeki container'lar
aynı **ağ alanını** (tek IP, birbirlerine `localhost` üzerinden erişir) ve gerektiğinde
aynı **depolamayı** (volume) paylaşır.

Pod'lar **geçicidir (ephemeral):** silinip yeniden oluştuğunda yeni bir IP alır, eski
pod geri gelmez. Bu yüzden gerçek uygulamalarda pod'ları **tek başına** değil, onları
yöneten üst nesnelerle (Deployment, ReplicaSet) oluştururuz.

---

## 🔑 Anahtar Kavramlar (dinlerken dikkat et)

- Pod nedir? (en küçük dağıtım birimi)
- Pod = bir veya daha fazla container (network + storage paylaşır)
- Pod yaşam döngüsü (Pending → Running → Succeeded/Failed)
- Pod'lar geçicidir (ephemeral), IP değişir
- Multi-container pod, sidecar pattern
- `kubectl exec`, `logs`, `describe`

---

## 💡 Notlarım

### Pod neden var?
- Kubernetes'in yönetim birimi container değil **pod**'dur. Scheduler pod'ları node'lara
  yerleştirir, kubelet pod'ları çalıştırır.
- Pod bir "sarmalayıcı"dır: içindeki container'lara ortak kimlik, ağ ve yaşam döngüsü verir.

### Pod içinde ne paylaşılır?
- **Ağ (network namespace):** Pod'un tek bir IP'si vardır. İçindeki container'lar
  birbirine `localhost:<port>` ile erişir. (Aynı pod'daki iki container aynı portu açamaz.)
- **Depolama (volume):** Pod'a tanımlı volume'ler, container'lar arasında paylaşılabilir.

### Pod yaşam döngüsü (phase)
| Phase | Anlamı |
|-------|--------|
| **Pending** | Pod kabul edildi ama container'lar henüz çalışmıyor (image çekiliyor, node bekleniyor). |
| **Running** | Pod bir node'a atandı, en az bir container çalışıyor. |
| **Succeeded** | Tüm container'lar başarıyla bitti (0 çıkış kodu), tekrar başlamayacak. |
| **Failed** | En az bir container hatayla sonlandı. |
| **Unknown** | Pod'un durumu alınamıyor (genelde node ile iletişim koptu). |

> Container düzeyinde ayrıca durumlar vardır: **Waiting**, **Running**, **Terminated**.
> `kubectl describe pod` bunları ve olayları (events) gösterir — sorun ararken ilk bakılacak yer.

### restartPolicy
- Pod'un container'ı bittiğinde ne olacağını belirler:
  - **Always** (varsayılan) — container biterse hep yeniden başlatılır (uzun süreli servisler).
  - **OnFailure** — sadece hatayla biterse yeniden başlatılır.
  - **Never** — hiç yeniden başlatılmaz (tek seferlik işler).

### Pod'lar ephemeral (geçici)
- Pod silinince kaybolur; yerine gelen yeni pod **yeni IP** alır ve içindeki geçici veriler gider.
- Bu yüzden:
  - Kalıcı veri için **volume** (kalıcı depolama) kullanılır.
  - Sabit bir erişim adresi için pod'un IP'sine değil, **Service**'e bağlanılır.
  - Uygulamalar tek pod yerine **Deployment** ile çalıştırılır (pod ölürse yenisini üretir).

### Multi-container pod desenleri
- **Sidecar:** Ana container'ın yanında yardımcı bir container (log toplama, proxy vb.).
  Aynı pod'da olduğu için ağ ve volume'u paylaşırlar.
- **Init container:** Ana container'lardan **önce** çalışıp biten hazırlık container'ı
  (ör. bir bağımlılığın hazır olmasını bekleme, dosya hazırlama). Bitmeden ana container başlamaz.

---

## 💻 Komutlar / YAML

### Pod oluşturma & listeleme
```bash
kubectl run nginx --image=nginx           # imperative: hızlı tek pod
kubectl get pods                          # pod'ları listele
kubectl get pods -o wide                  # + node ve IP bilgisi
kubectl get pods -w                       # değişimleri canlı izle (watch)
kubectl get pod nginx -o yaml             # pod'un tam tanımı
```

### İnceleme & hata ayıklama
```bash
kubectl describe pod nginx                # detay + events (Pending sebebi burada)
kubectl logs nginx                        # container loglarını gör
kubectl logs nginx -f                     # canlı takip
kubectl logs nginx -c <container>         # çok container'lı pod'da belirli container
kubectl exec -it nginx -- /bin/bash       # container içinde shell aç
kubectl exec nginx -- ls /app             # tek komut çalıştır
```

### Erişim & silme
```bash
kubectl port-forward pod/nginx 8080:80    # yerel 8080 → pod 80 (test için)
kubectl delete pod nginx
kubectl delete -f pod.yaml
```

### Şablon üret (YAML yazmayı hızlandır)
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

### Örnek YAML — tek container'lı Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
  restartPolicy: Always
```
```bash
kubectl apply -f pod.yaml
```

### Örnek YAML — multi-container (ana + sidecar)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-sidecar
spec:
  containers:
    - name: app
      image: nginx:1.27
    - name: log-agent          # sidecar
      image: busybox
      command: ["sh", "-c", "while true; do echo tick; sleep 5; done"]
```

### Örnek YAML — init container
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
    - name: wait-db
      image: busybox
      command: ["sh", "-c", "echo hazirlik; sleep 3"]
  containers:
    - name: app
      image: nginx:1.27
```

---

## ❓ Anlamadıklarım / Tekrar

- **Neden doğrudan Pod değil de Deployment?**
  → Tek pod ölürse kendiliğinden geri gelmez. Deployment "hep N tane çalışsın" der;
  pod ölünce yenisini üretir, güncelleme/ölçekleme sağlar. Prod'da pod tek başına kullanılmaz.

- **Pod neden `Pending` takılır?**
  → Genelde: uygun node yok (kaynak yetersiz), image çekilemiyor, ya da node `NotReady`.
  Sebebi `kubectl describe pod <ad>` çıktısındaki **Events** kısmında yazar.

- **Sidecar ile init container farkı?**
  → Init container ana container'lardan **önce** çalışıp **biter**. Sidecar ana container'la
  **birlikte** çalışmaya devam eder (yardımcı görev).

- **`restartPolicy` ne zaman önemli?**
  → Uzun süreli servis → Always. Tek seferlik/iş (job) → OnFailure veya Never.
