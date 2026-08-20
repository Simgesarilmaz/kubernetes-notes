# 06 - Label & Selector

Bu not, Kubernetes'te **label** (etiket) ve **selector** (seçici) kavramlarını, kullanım kurallarını ve `kubectl` komutlarını kapsar.

---

## 1. Label (Etiket) Nedir?

Label, bir Kubernetes objesine (pod, node vb.) atadığımız **anahtar:değer** (`key:value`) çiftidir. Objeleri gruplamak, filtrelemek ve objeler arası ilişki kurmak için kullanılır.

```
tier : front-end
 │        │
key    value
(anahtar) (değer)
```

Örnekler:

```
tier:front-end
stage:test
name:app1
team:development
```

### Prefix (Önek) — Opsiyonel

Bir label anahtarının önünde opsiyonel bir **prefix** olabilir:

```
example.com/tier : front-end
    │          │       │
  Prefix    Anahtar   Değer
(opsiyonel)
```

- Prefix kısmı **zorunlu değildir**.
- `kubernetes.io/` ve `k8s.io/` önekleri **Kubernetes çekirdek bileşenlerine ayrılmıştır**; bu bileşenler tarafından kullanılır.

### Label Kuralları

- Anahtar ve değer alanları **en fazla 63 karakter** olmalıdır (değer boş olabilir).
- Alfanumerik bir karakterle (`[a-z 0-9 A-Z]`) **başlamalı ve bitmelidir**.
- Arada tire (`-`), alt çizgi (`_`), nokta (`.`) ve alfanumerik değerler içerebilir.
- Bir objeye (pod) **aynı anahtar iki defa** atanamaz.
- Label tanımı `metadata` kısmında yapılır.

---

## 2. Tek Dosyada Birden Fazla Kaynak (`---`)

Kubernetes'te kaynakları ayrı ayrı dosyalarda tanımlamak **zorunda değilsiniz**. Tek bir dosya içinde `---` ayıracı koyarak bir sonraki kaynağı ekleyebilirsiniz.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    app: firstapp
    tier: frontend
    mycluster.local/team: team1
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    app: firstapp
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
# ... pod3 – pod10 aynı yapıda, farklı app/tier/team kombinasyonlarıyla ...
---
apiVersion: v1
kind: Pod
metadata:
  name: pod11
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
  nodeSelector:
    hddtype: ssd
```

> Bu örnekte 11 pod tanımlanmıştır (pod1–pod11). pod1–pod10 farklı label kombinasyonlarına sahiptir; **pod11'de label yoktur ama `nodeSelector` vardır** (aşağıda anlatılıyor).

Podların label dağılımı (`kubectl get pods --show-labels` çıktısı):

```
pod1    app=firstapp,mycluster.local/team=team1,tier=frontend
pod2    app=firstapp,tier=frontend
pod3    app=firstapp,tier=backend
pod4    app=firstapp,tier=backend
pod5    app=secondapp,tier=frontend
pod6    app=secondapp,tier=frontend
pod7    app=secondapp,tier=backend
pod8    app=secondapp,tier=backend
pod9    team=team1
pod10   team=team2
pod11   <none>            # label yok, nodeSelector'ı var
```

---

## 3. Selector (Seçici) Kullanımı

Podları label'larına göre filtrelemek için `-l` (veya `--selector`) bayrağı kullanılır. `--show-labels` çıktıda label'ları da gösterir.

İki tür selector vardır: **equality-based** (eşitlik temelli) ve **set-based** (atama/küme temelli). İkisi benzer işleri yapar, sadece syntax farklıdır.

### 3.1. Equality-Based (Eşitlik Temelli) Selector

Virgül (`,`) burada **AND** anlamına gelir.

| Komut | Anlamı |
|-------|--------|
| `kubectl get pods -l "app" --show-labels` | `app` anahtarı atanmış tüm podlar (değer önemsiz) |
| `kubectl get pods -l "app=firstapp" --show-labels` | `app` değeri `firstapp` olanlar |
| `kubectl get pods -l "app=firstapp,tier=frontend" --show-labels` | `app=firstapp` **VE** `tier=frontend` (iki filtre AND) |
| `kubectl get pods -l "app=firstapp,tier!=frontend" --show-labels` | `app=firstapp` ama `tier` frontend **olmayanlar** |
| `kubectl get pods -l "app,tier=frontend" --show-labels` | `app` anahtarı var (değeri önemsiz) **VE** `tier=frontend` |
| `kubectl get pods -l "app=firstapp,app=secondapp" --show-labels` | Aynı anahtara iki eşitlik → çakışır, **boş döner** |

### 3.2. Set-Based (Atama/Küme Temelli) Selector

`in`, `notin` ve `!` operatörlerini kullanır. **Tek tırnak** kullanılır.

| Komut | Anlamı |
|-------|--------|
| `kubectl get pods -l 'app in (firstapp)' --show-labels` | `app` anahtarına `firstapp` atanmış podlar |
| `kubectl get pods -l 'app in (firstapp,secondapp)' --show-labels` | `app` = firstapp **VEYA** secondapp (`in` içindeki çoklu değer = OR) |
| `kubectl get pods -l 'app notin (firstapp)' --show-labels` | `app` anahtarı var ama değeri `firstapp` **değil** |
| `kubectl get pods -l 'app, app notin (firstapp)' --show-labels` | `app` anahtarı var **VE** değeri `firstapp` değil (virgül = AND) |
| `kubectl get pods -l '!app' --show-labels` | `app` anahtarı **hiç bulunmayan** podlar |
| `kubectl get pods -l "app in (firstapp),tier notin (frontend)" --show-labels` | `app=firstapp` ama `tier` frontend **atanmamış** olanlar |

### ⚠️ Önemli Ayrım: `!=` / `notin` vs `!app`

Bu ikisi **farklı** şeydir, karıştırma:

- **`app!=firstapp`** ve **`app notin (firstapp)`** → `app` anahtarı **olan** ama değeri `firstapp` **olmayan** podları getirir. `app` anahtarı hiç olmayanları **getirmez**.
- **`!app`** → `app` anahtarı **hiç olmayan** podları getirir.

Yani `!` (ünlem) bir nevi "bu anahtar **yok**" demektir; `!=` / `notin` ise "anahtar **var** ama değeri şu **değil**" demektir.

---

## 4. Label Yönetimi (Imperative)

Label'ları YAML ile (declarative) tanımlayabileceğimiz gibi, `kubectl` ile anlık (imperative) da yönetebiliriz.

### Label Ekleme

```bash
kubectl label pods pod9 app=thirdapp
```
> pod9'a `app=thirdapp` label'ını ekler.

### Label Silme

```bash
kubectl label pods pod9 app-
```
> Anahtar adının sonuna `-` koyarak o anahtarı (ve değerini) kaldırır.

### Label Güncelleme

Var olan bir label'ın değerini değiştirmek için `--overwrite` gerekir:

```bash
kubectl label --overwrite pods pod9 team=team3
```

### Tüm Podlara Label Ekleme

```bash
kubectl label pods --all foo=bar
```
> Namespace'teki tüm podlara `foo=bar` label'ını ekler.

---

## 5. Label ile Obje İlişkisi: `nodeSelector`

Label'lar sadece filtreleme için değil, **objeler arası ilişki kurmak** için de kullanılır. Bunun en yaygın örneği pod'u belirli bir node'a yerleştirmektir.

Normalde **kube-scheduler** kendi algoritmasına göre pod'u hangi node'da çalıştıracağına karar verir. Ancak `nodeSelector` ile bu seçime biz müdahale edebiliriz:

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:latest
  nodeSelector:
    hddtype: ssd
```

Bu tanım şu anlama gelir: *"kube-scheduler, bu pod'u schedule ederken `hddtype: ssd` label'ı atanmış bir node bul ve pod'u orada çalıştır."* Böylece pod ile node **eşleştirilmiş** olur.

### Node'a Label Atama

Node'lara da tıpkı podlar gibi label atanır:

```bash
# Node'ların mevcut label'larını gör
kubectl get nodes --show-labels

# Worker node'a hddtype=ssd label'ı ekle
kubectl label nodes k8s-worker hddtype=ssd
```

### 💡 Gözlem: pod11'in Schedule Edilmesi

`pod11` başta **Pending** durumundaydı, çünkü `nodeSelector: hddtype: ssd` koşulunu sağlayan hiçbir node yoktu. `k8s-worker` node'una `hddtype=ssd` label'ı eklenir eklenmez, scheduler eşleşmeyi buldu ve pod11 sırayla:

```
Pending → ContainerCreating → Running
```

durumuna geçti. (`kubectl get pods -w` ile canlı izlenebilir.)

### Pratik Senaryo

Diyelim 20 worker node'unuz var; bazılarında hızlı (SSD), bazılarında yavaş disk var. Disk I/O performansına ihtiyaç duyan podları, SSD'li makinelere label atayıp `nodeSelector` ile yönlendirirsek bu podlardan daha yüksek verim alırız.

---

## Özet

- **Label** = objelere atanan `key:value` çifti; `metadata` altında tanımlanır.
- Aynı objeye aynı anahtar iki kez atanamaz; max 63 karakter; alfanumerikle başlar/biter.
- **`---`** ile tek dosyada birden çok kaynak tanımlanabilir.
- **Equality-based** selector: `=`, `!=` (virgül = AND).
- **Set-based** selector: `in`, `notin`, `!` (in içi çoklu değer).
- `!=`/`notin` = "anahtar var, değeri şu değil"; `!key` = "anahtar hiç yok".
- Label yönetimi: ekle (`key=val`), sil (`key-`), güncelle (`--overwrite`), tümüne (`--all`).
- **`nodeSelector`** + node label'ı ile pod belirli node'lara yerleştirilir.
