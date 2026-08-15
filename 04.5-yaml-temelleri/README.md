# 04.5 - YAML Temelleri

## 📝 Özet

**YAML** (YAML Ain't Markup Language), insan tarafından kolay okunup yazılabilen bir
**veri/yapılandırma formatıdır.** Sadece Kubernetes'e özel değildir; Docker Compose,
CI/CD (GitHub Actions, GitLab CI), Ansible gibi birçok araç da YAML kullanır.

Kubernetes'te **her nesneyi** (Pod, Deployment, Service...) bir YAML dosyasıyla
tanımlarız ve `kubectl apply -f dosya.yaml` ile cluster'a uygularız. Bu yüzden YAML
söz dizimini oturtmak, sonraki tüm konuların temelidir.

**En kritik kural:** YAML **girintiye (boşluğa) duyarlıdır** ve **TAB kabul etmez** —
sadece boşluk (space) kullanılır. Bir boşluk kayması dosyayı geçersiz yapar.

---

## 🔑 Anahtar Kavramlar (dinlerken dikkat et)

- YAML nedir, nerelerde kullanılır?
- Girinti (indentation) = boşluk, **TAB yok**
- `key: value` (map/sözlük) yapısı
- `- ` ile başlayan liste (array) yapısı
- İç içe (nested) yapı
- Veri tipleri: string, number, boolean, null
- `---` ile tek dosyada birden çok belge
- `#` ile yorum satırı
- Kubernetes nesne iskeleti: `apiVersion / kind / metadata / spec`

---

## 💡 Notlarım

### Temel yapı: key–value (map)
```yaml
ad: ahmet
yas: 30
aktif: true
```
- `anahtar: değer` şeklinde. İki nokta üstünden **sonra bir boşluk** olmalı (`ad:ahmet` ❌).

### Girinti (indentation)
- İç içe yapı **boşlukla** kurulur (genelde 2 boşluk).
- Aynı seviyedeki alanlar aynı hizada olmalı.
```yaml
kisi:
  ad: ahmet         # 2 boşluk içeride → kisi'nin altında
  adres:
    sehir: ankara   # 4 boşluk içeride → adres'in altında
```

### Listeler (array)
- Her eleman `- ` (tire + boşluk) ile başlar.
```yaml
meyveler:
  - elma
  - armut
  - kiraz
```
- Liste elemanları map de olabilir (Kubernetes'te `containers` böyledir):
```yaml
containers:
  - name: nginx
    image: nginx:1.27
  - name: redis
    image: redis:7
```

### Veri tipleri
```yaml
metin: merhaba          # string (tırnak opsiyonel)
metin2: "123"           # tırnaklı → string olarak kalır
sayi: 123               # number
ondalik: 3.14           # float
dogruMu: true           # boolean (true/false)
bosDeger: null          # null (~ da olur)
```
> Dikkat: `evet/hayir`, `on/off`, `yes/no` bazı yorumlayıcılarda boolean'a dönüşebilir.
> Kesin string istiyorsan tırnak kullan: `deger: "yes"`.

### Yorum satırı
```yaml
# bu bir yorumdur
port: 80   # satır sonunda da yorum olur
```

### Tek dosyada birden çok belge
- `---` ayıracı ile aynı dosyaya birden çok nesne konur (K8s'te çok kullanılır):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
---
apiVersion: v1
kind: Service
metadata:
  name: svc-1
```

### Kubernetes nesne iskeleti (her nesnede aynı 4 üst alan)
```yaml
apiVersion: v1     # hangi API sürümü (Pod→v1, Deployment→apps/v1)
kind: Pod          # nesne türü
metadata:          # kimlik: ad, etiketler, namespace
  name: nginx
  labels:
    app: web
spec:              # "istenen durum" — nesnenin asıl tanımı
  containers:
    - name: nginx
      image: nginx:1.27
```
- **apiVersion + kind + metadata + spec** dörtlüsü Kubernetes YAML'larının değişmez iskeletidir.
- `spec`'in içeriği nesne türüne göre değişir (Pod'un spec'i başka, Service'in başka).

---

## 💻 Komutlar / YAML

### Doğrulama & keşif
```bash
# YAML'ı uygulamadan önce doğrula (cluster'a bir şey yazmaz)
kubectl apply -f pod.yaml --dry-run=client

# Bir alanın ne olduğunu / hangi alt alanları olduğunu öğren
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers

# Var olan bir nesnenin YAML halini gör (örnek/şablon çıkarmak için)
kubectl get pod nginx -o yaml
```

### Şablon üretme (elle yazmadan)
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
```

### Uygulama
```bash
kubectl apply -f pod.yaml        # tek dosya
kubectl apply -f ./klasor/       # klasördeki tüm YAML'lar
kubectl delete -f pod.yaml
```

---

## ❓ Anlamadıklarım / Tekrar

- **Neden TAB kullanamıyorum?**
  → YAML spesifikasyonu girinti için sadece boşluğa izin verir. Editörünü "TAB → 2 space"
  olacak şekilde ayarla; görünmez TAB'lar en sık hata sebebidir.


- **Liste mi map mi karıştırıyorum?**
  → `- ` ile başlıyorsa liste elemanı; `anahtar: değer` ise map alanı. K8s'te
  `containers` bir listedir (her eleman `- name:` ile başlar).

- **`apiVersion` ne zaman `v1`, ne zaman `apps/v1`?**
  → Temel nesneler (Pod, Service, ConfigMap) `v1`. Deployment/ReplicaSet/StatefulSet
  gibi workload'lar `apps/v1`. Emin değilsen: `kubectl explain <kind>` ilk satırda yazar.

- **YAML'ımı uygulamadan nasıl kontrol ederim?**
  → `kubectl apply -f dosya.yaml --dry-run=client` — söz dizimi/şema hatalarını
  cluster'a yazmadan gösterir.
