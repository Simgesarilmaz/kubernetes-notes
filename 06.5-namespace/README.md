# 06.5 - Namespace

## Problem: Ortak Dosya Sunucusu Neden Yetmez?

Bir firmada 10 ayrı ekibin dosya paylaşacağı bir servis kurmak istediğimizi düşünelim. Herkese `/share` klasörüne erişim veririz ve başlangıçta her şey yolunda görünür. Fakat zamanla şu problemler ortaya çıkar:

- **Yönetilemezlik:** Tüm kullanıcılar dosyaları aynı yere yazdığı için, belli bir süre sonra klasör içeriği yönetilemez hale gelir.
- **İsim çakışması:** Farklı kullanıcılar aynı dosya isimlerini kullanabilir ve birbirlerinin dosyalarını ezebilir.
- **Erişim/izolasyon eksikliği:** Örneğin İK (HR) çalışanlarının görmesi gereken bir dosya yüklerim, ama diğer çalışanların bunu görmemesi gerekir. Ortak klasörde bunu sağlamak zordur.
- **Kota yönetimi:** Herkese ayrı ayrı kota (kaynak sınırı) ayarlamak gerekir, fakat tek bir ortak klasörde bunu yapamayız. Başlangıçta sorun olmasa da zamanla ciddi bir soruna dönüşür.

### Çözüm Fikri

Bu sorunları çözmek için ekiplere ayrı klasörler oluşturabiliriz (`/share/ekip1`, `/share/ekip2`, … `/share/ekip10`). Bunu daha da genişletip **proje bazlı** klasörler de yaratabiliriz.

> **Namespace'leri işte bu klasörler gibi düşünebiliriz.** Kaynakları mantıksal olarak bölerek erişim, izolasyon ve kota yönetimini mümkün kılarlar.

---

## Namespace Nedir?

Namespace'ler, adlar için bir **kapsam (scope)** sağlar. Temel kuralları:

- Kaynak adlarının **bir namespace içinde benzersiz** olması gerekir (farklı namespace'lerde aynı isim kullanılabilir).
- Namespace'ler **birbirinin içine yerleştirilemez** (iç içe olamaz).
- Her Kubernetes kaynağı **yalnızca bir namespace içinde** olabilir.

Kısacası namespace'ler, cluster kaynaklarını **birden çok kullanıcı/ekip arasında bölmenin** bir yoludur.

### Varsayılan Namespace'ler

Her Kubernetes kurulumunda varsayılan olarak 4 namespace oluşturulur:

| Namespace | Açıklama |
|-----------|----------|
| `default` | Aksi belirtilmedikçe objelerin toplandığı varsayılan namespace. |
| `kube-system` | Kubernetes sisteminin kendi objelerinin (control plane bileşenleri vb.) tutulduğu yer. |
| `kube-public` | Tüm kullanıcılar tarafından erişilmesi gereken objelerin oluşturulacağı yer. |
| `kube-node-lease` | Node heartbeat (node'ların canlılık bilgisi / lease) işlemleri için özel namespace. |

> **Ne zaman gerekli?** Cluster tek bir ekip tarafından yürütülüyorsa `default` namespace ile devam edebiliriz. Fakat ortam büyüdükçe namespace'ler bize **erişim kontrolü ve kota** için yardımcı olur.

---

## Namespace Komutları

### Listeleme

Belirli bir namespace'teki pod'ları listelemek:

```bash
kubectl get pods --namespace kube-system
# kısa hali:
kubectl get pods -n kube-system
```

Tüm namespace'lerdeki pod'ları listelemek:

```bash
kubectl get pods --all-namespaces
# kısa hali:
kubectl get pods -A
```

Mevcut namespace'leri listelemek:

```bash
kubectl get namespaces
```

### Yeni Namespace Oluşturma

Namespace bir Kubernetes objesi olduğu için iki yöntemle oluşturulabilir.

**1. Imperative (komut ile):**

```bash
kubectl create namespace app1
```

```bash
kubectl get namespaces
NAME               STATUS   AGE
app1               Active   10s
calico-apiserver   Active   9d
calico-system      Active   9d
default            Active   9d
kube-node-lease    Active   9d
kube-public        Active   9d
kube-system        Active   9d
tigera-operator    Active   9d
```

**2. Declarative (YAML ile):**

Aşağıdaki YAML hem bir namespace hem de o namespace içinde bir pod oluşturur. `---` iki objeyi birbirinden ayırır. Pod'un `metadata.namespace` alanı, onu hangi namespace'e bağlayacağımızı belirtir.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
---
apiVersion: v1
kind: Pod
metadata:
  namespace: development
  name: namespacepod
spec:
  containers:
  - name: namespacecontainer
    image: nginx:latest
    ports:
    - containerPort: 80
```

Uygulamak:

```bash
kubectl apply -f podnamespace.yaml
namespace/development created
pod/namespacepod created
```

---

## Namespace İçindeki Objelerle Çalışmak

Pod `development` namespace'inde oluştuğu için, `default` namespace'te görünmez:

```bash
kubectl get pods
No resources found in default namespace.

kubectl get pods -n development
NAME           READY   STATUS    RESTARTS   AGE
namespacepod   1/1     Running   0          40s
```

Bir obje ile iletişim kurmak istediğimizde **namespace'ini belirtmek** gerekir. Aksi halde obje bulunamaz:

```bash
kubectl exec -it namespacepod -- /bin/sh
Error from server (NotFound): pods "namespacepod" not found
```

Namespace belirtildiğinde çalışır:

```bash
kubectl exec -it namespacepod -n development -- /bin/sh
#
```

### Varsayılan Namespace'i Değiştirmek

Her komutta `-n development` yazmamak için, mevcut context'in varsayılan namespace'ini değiştirebiliriz:

```bash
kubectl config set-context --current --namespace=development
Context "kubernetes-admin@kubernetes" modified.
```

Böylece varsayılan namespace'imiz artık `development` olur ve komutlarda ayrıca belirtmemize gerek kalmaz.

---

## Namespace Silme

> ⚠️ **Dikkat:** Bir şey silinirken Kubernetes "emin misin?" diye sormaz. Namespace silindiğinde içindeki **tüm objeler de silinir**.

```bash
kubectl delete namespaces development
namespace "development" deleted
```

---

## Özet

Namespace'ler, tıpkı dosya sunucusundaki ekip/proje klasörleri gibi, cluster'ı mantıksal parçalara böler. Sağladıkları başlıca faydalar: isim çakışmalarını önleme (isimler namespace başına benzersizdir), ekipler/projeler arası izolasyon, erişim kontrolü (RBAC) ve kaynak kotası (ResourceQuota) uygulayabilme imkânı.
