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

## 🔑 Anahtar Kavramlar

- Pod nedir? (en küçük dağıtım birimi)
- Pod = bir veya daha fazla container (network + storage paylaşır)
- Pod yaşam döngüsü (Pending → Running → Succeeded/Failed)
- Pod'lar geçicidir (ephemeral), IP değişir
- Multi-container pod: **sidecar** ve **init container**
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

### Sık görülen container STATUS'ları

> `kubectl get pods` çıktısındaki **STATUS** sütununda phase yerine çoğu zaman
> aşağıdaki değerleri görürsün. Bunlar **pod phase değildir**; container'ın
> `Waiting` / `Terminated` durumundaki **reason** (sebep) değerleridir.

| STATUS | Anlamı | Sık sebep | İlk bakılacak |
|--------|--------|-----------|---------------|
| **ContainerCreating** | Pod node'a atandı; image indiriliyor, volume/ağ hazırlanıyor. Normal geçici durum. | — (uzun sürerse sorun) | `kubectl describe pod` |
| **ImagePullBackOff** / ErrImagePull | Container image'ı çekilemiyor. "BackOff" = artan bekleme ile tekrar tekrar deniyor. | Yanlış image adı/tag, private registry yetkisi yok, Docker Hub rate limit | `kubectl describe pod` (Events) |
| **CrashLoopBackOff** | Container başlıyor → hemen çöküyor → restart → yine çöküyor. Restart araları giderek uzuyor. | Eksik env/config, başlangıç komutu hatalı, uygulama exception atıyor | `kubectl logs <ad>` (önceki için `--previous`) |

**Kural:**
- `ImagePullBackOff` → sorun container **başlamadan önce** (image katmanında) → `describe`.
- `CrashLoopBackOff` → sorun container **başladıktan sonra** (uygulama katmanında) → `logs`.

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

---

### Sidecar deseni (WordPress + fluentd)

Bir pod içinde birden fazla container çalıştırılabilir, ancak bunun amacı iki bağımsız
uygulamayı bir araya getirmek değildir. WordPress ile MySQL gibi birbirinden bağımsız
iki uygulamayı aynı pod'a koymak yanlış olur; çünkü bu durumda ikisini **ayrı ayrı
ölçekleyemeyiz**. Trafik arttığında genellikle beş WordPress kopyasına ihtiyaç duyulurken
tek bir MySQL yeterlidir. İki uygulama tek pod'da olsaydı, WordPress'i çoğaltmak için
MySQL'i de çoğaltmak gerekirdi. Bu yüzden bağımsız uygulamalar **ayrı pod'lara** konur.

Multi-container pod'un asıl amacı, bir **ana uygulamanın** yanına ona sıkı sıkıya bağlı
bir **yardımcı container** eklemektir. Bu yardımcıya **sidecar** denir. Tipik örnek
WordPress (ana) ve fluentd (sidecar) ikilisidir: WordPress log üretir, fluentd bu logları
toplayıp dışarı (ör. Elasticsearch'e) gönderir. fluentd tek başına bir anlam taşımaz;
bağımsız bir uygulama değil, WordPress'in bir eklentisidir. Bu nedenle "her container tek
uygulama çalıştırsın" kuralını da çiğnemez — sidecar ikinci bir uygulama değil, birincinin
tamamlayıcısıdır.

Sidecar ana uygulamayla **aynı hızda ölçeklenir** (her WordPress kopyası kendi log
toplayıcısıyla gelir) ve **aynı yaşam döngüsünü** paylaşır; pod öldüğünde ikisi birlikte
ölür, birlikte doğar.

Sidecar'ı ayrı bir pod'a koymak yerine aynı pod'da tutmamızın somut bir nedeni vardır.
Aynı pod içindeki container'lar **aynı ağı (`localhost`)** ve **aynı volume'u** paylaşır.
fluentd, WordPress'in log dosyalarına ancak paylaşılan bir volume üzerinden erişebilir.
Ayrı pod'larda olsalardı ne aynı dosya sistemini ne de `localhost`'u paylaşırlardı ve
fluentd logları hiç göremezdi.

Özetle kural şudur: bir pod'da **tek bir ana uygulama** bulunur; ek bir container ancak
bu ana uygulamaya bağlı bir **yardımcıysa** (sidecar ya da init container) pod'a eklenir.

#### Örnek YAML — WordPress + fluentd sidecar (paylaşılan volume)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wordpress-with-logging
spec:
  volumes:
    - name: log-store          # iki container'ın paylaştığı ortak volume
      emptyDir: {}
  containers:
    - name: wordpress          # ANA container
      image: wordpress:6
      volumeMounts:
        - name: log-store
          mountPath: /var/log/app   # WordPress logları buraya yazar
    - name: fluentd            # SIDECAR container
      image: fluentd:v1.16
      volumeMounts:
        - name: log-store
          mountPath: /var/log/app   # fluentd aynı dizini OKUR
```

Her iki container da `log-store` volume'unu aynı `mountPath` ile bağladığı için aynı dizini
görür: biri yazar, diğeri okur. Sidecar mantığının kalbi bu paylaşımdır. Buradaki
`emptyDir: {}`, pod'un ömrü boyunca yaşayan geçici bir volume'dur; pod silindiğinde o da
kaybolur — logları asıl kalıcı hâle getiren zaten fluentd'nin kendisidir.

#### Demo: aynı pod'da iki container ne paylaşır?

YAML ile aynı pod içinde iki container tanımlandı: **webcontainer** (nginx) ve
**sidecarcontainer** (pod adı: `multicontainer`). Bu demo, aynı pod'daki container'ların
neyi paylaştığını gösterir.

**1) Ortak ağ (aynı IP)**
Her iki container'da `ifconfig` çalıştırılınca `eth0` IP'si **aynı** çıktı.
Yani aynı pod içindeki container'lar tek bir ağ alanını (network namespace) paylaşır; bu
yüzden birbirlerine `localhost` üzerinden erişirler.
```bash
kubectl exec -it multicontainer -c sidecarcontainer -- /bin/sh

```

**2) Ortak volume (tek volume, farklı path)**
Volume'ler iki container'da farklı yollara mount edildi:
- webcontainer → `/usr/share/nginx/html`
- sidecarcontainer → `/var/log`

webcontainer'da oluşturulan `a.txt`, sidecar'ın `/var/log` yolunda da göründü. Sebebi:
ikisi de **aynı volume'u** mount ediyor, sadece kendi içlerinde farklı klasöre bağlıyor.
`mountPath` volume'un o container'da nereden görüneceğini belirler; volume'un kendisi tektir.
Yani `a.txt` fiziksel olarak ortak volume'da durur; biri ona `/usr/share/nginx/html/a.txt`,
diğeri `/var/log/a.txt` diye bakar.

**3) Görev ayrımı (sidecar mantığının özü)**
sidecarcontainer arka planda **15 saniyede bir GitHub'tan** güncel içeriği indirip ortak
volume'a yazıyordu. nginx (webcontainer) ise sadece o volume'daki içeriği sunuyordu.
Böylece:
- webcontainer → içeriği **sunar** (tek işi bu, GitHub'ı bilmez)
- sidecarcontainer → içeriği **güncel tutar** (çeker, ortak volume'a koyar)

nginx image'ına GitHub çekme kodu koymaya gerek kalmaz; o iş sidecar'a devredilir. Sidecar
desenini mümkün kılan da yukarıdaki iki paylaşımdır: **aynı ağ + aynı volume.**

---

### Init container

Init container, esas uygulama container'ı başlamadan önce çalışan bir **hazırlık
container'ıdır.** Görevini tamamlar ve **kapanır**; ancak o kapandıktan sonra app container
başlar. İşini bitirmezse app container **hiç başlatılmaz** — yani init container bir
ön koşul / kapı görevi görür.

Sidecar'dan en önemli farkı **yaşam döngüsüdür:** sidecar ana uygulamayla birlikte, pod
yaşadıkça **sürekli** çalışır; init container ise ana uygulamadan **önce, bir kez** çalışır
ve **biter**. (Yani init container pod'un yaşam döngüsü boyunca çalışmaz.)

YAML'da da yerleri farklıdır: normal container'lar `containers:` altında, init container'lar
ise **ayrı bir `initContainers:` alanında** tanımlanır. Birden fazla init container varsa
**sırayla** (biri bitince diğeri) çalışırlar.

**Neden gerek duyulur?** Esas uygulamanın çalışabilmesi için önceden yapılması gereken bir
hazırlık ya da beklenmesi gereken bir bağımlılık olduğunda. Klasik örnek: bir
Service / veritabanı hazır olana kadar uygulamayı başlatmamak.

#### Örnek YAML — myservice hazır olana kadar bekleyen init container
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: initcontainerpod
spec:
  containers:
    - name: appcontainer
      image: busybox
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
    - name: initcontainer
      image: busybox
      command: ['sh', '-c', "until nslookup myservice; do echo waiting for myservice; sleep 2; done"]
```
Init container'ın komutu: "`myservice` adlı Service DNS'te çözülene kadar her 2 saniyede bir
kontrol et; çözülmüyorsa 'waiting for myservice' yaz ve beklemeye devam et."

#### Demo: Init:0/1 → Running
myservice **yokken** pod'un durumu:
```bash
kubectl get pods
# initcontainerpod   0/1   Init:0/1   0   8s
```
- `Init:0/1` → 1 init container'dan 0'ı bitti; init **hâlâ çalışıyor**, myservice'i bekliyor.
- `0/1` (READY) → app container henüz **başlamadı**.
- `kubectl describe pod initcontainerpod` → `Status: Pending`.
- `kubectl logs -f initcontainerpod -c initcontainer` → tekrarlayan
  `waiting for myservice` + `NXDOMAIN` (servis DNS'te yok).

Sonra myservice oluşturulunca:
```bash
kubectl apply -f service1.yaml     # service/myservice created
```
nslookup başarılı olur → `until` döngüsü biter → init container kapanır → app container
başlar → pod `1/1 Running` olur.

> **STATUS okuma ipucu:** `Init:0/1` init container'ın çalıştığını/beklediğini gösterir;
> pod `Running`'e geçtiyse init container görevini bitirip kapanmış demektir.

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
kubectl port-forward pod/nginx 8080:80    # makinen ile pod arasında GEÇİCİ tünel/proxy (localhost:8080 -> pod:80)
                                          # sadece komut açıkken + sadece senin makinenden çalışır → test/debug içindir
                                          # kalıcı/dışarı açık erişim için Service (NodePort, LoadBalancer)
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

> Multi-container (sidecar) ve init container YAML örnekleri için yukarıdaki
> **Sidecar deseni** ve **Init container** bölümlerine bak.

---

## ❓ Anlamadıklarım / Tekrar

- **Neden doğrudan Pod değil de Deployment?**
  → Tek pod ölürse kendiliğinden geri gelmez. Deployment "hep N tane çalışsın" der;
  pod ölünce yenisini üretir, güncelleme/ölçekleme sağlar. Prod'da pod tek başına kullanılmaz.

- **Pod neden `Pending` takılır?**
  → Genelde: uygun node yok (kaynak yetersiz), image çekilemiyor, ya da node `NotReady`.
  Sebebi `kubectl describe pod <ad>` çıktısındaki **Events** kısmında yazar.

- **Sidecar ile init container farkı?**
  → Init container ana container'lardan **önce** çalışıp **biter** (bir kez). Sidecar ana
  container'la **birlikte** çalışmaya devam eder (sürekli, yardımcı görev).

- **`Init:0/1` ne demek?**
  → Init container hâlâ çalışıyor / bir ön koşulu bekliyor; app container henüz başlamadı.
  Ön koşul sağlanınca pod `Running`'e geçer.

- **`restartPolicy` ne zaman önemli?**
  → Uzun süreli servis → Always. Tek seferlik/iş (job) → OnFailure veya Never.
