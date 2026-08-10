# ☸️ Kubernetes Temelleri — Notlarım

> **Ayti.tech "Kubernetes Temelleri"** eğitimi boyunca tuttuğum notlar, komutlar ve pratik YAML örnekleri.
> Klasörler eğitimdeki konu sırasına göre düzenlenmiştir; her dersi izledikçe ilgili dosyayı dolduruyorum.

![Kubernetes](https://img.shields.io/badge/Kubernetes-Learning-326CE5?logo=kubernetes&logoColor=white)
![Status](https://img.shields.io/badge/Durum-Devam%20Ediyor-yellow)

🎓 Eğitim reposu : [aytitech/k8sfundamentals](https://github.com/aytitech/k8sfundamentals)

---

## 📚 İçindekiler

| # | Konu | Klasör | ✔ |
|---|------|--------|:-:|
| 01 | Giriş | [`01-giris`](./01-giris) | ✅ |
| 02 | Kubernetes Mimarisi (Teori) | [`02-mimari`](./02-mimari) | ✅ |
| 03 | Kurulum (Setup) | [`03-setup`](./03-setup) | ✅ |
| 04 | kubectl | [`04-kubectl`](./04-kubectl) | ✅ |
| 05 | Pod | [`05-pod`](./05-pod) | ⬜ |
| 06 | Label & Selector | [`06-labelselector`](./06-labelselector) | ⬜ |
| 07 | Service | [`07-service`](./07-service) | ⬜ |
| 08 | Deployment | [`08-deployment`](./08-deployment) | ⬜ |
| 09 | Secret & ConfigMap | [`09-secretconfigmap`](./09-secretconfigmap) | ⬜ |
| 10 | Resource Request & Limit | [`10-requestlimit`](./10-requestlimit) | ⬜ |
| 11 | Image Pull Secret | [`11-imagesecret`](./11-imagesecret) | ⬜ |
| 12 | Volume | [`12-volume`](./12-volume) | ⬜ |
| 13 | PersistentVolume & PVC | [`13-pvpvc`](./13-pvpvc) | ⬜ |
| 14 | StorageClass | [`14-storageclass`](./14-storageclass) | ⬜ |
| 15 | StatefulSet | [`15-statefulset`](./15-statefulset) | ⬜ |
| 16 | DaemonSet | [`16-daemonset`](./16-daemonset) | ⬜ |
| 17 | Job & CronJob | [`17-jobcronjob`](./17-jobcronjob) | ⬜ |
| 18 | Liveness & Readiness Probe | [`18-liveready`](./18-liveready) | ⬜ |
| 19 | Affinity | [`19-affinity`](./19-affinity) | ⬜ |
| 20 | Taint & Toleration | [`20-tainttoleration`](./20-tainttoleration) | ⬜ |
| 21 | Ingress | [`21-ingress`](./21-ingress) | ⬜ |
| 22 | Network Policy | [`22-networkpolicy`](./22-networkpolicy) | ⬜ |
| 23 | Authentication | [`23-authentication`](./23-authentication) | ⬜ |
| 24 | RBAC | [`24-rbac`](./24-rbac) | ⬜ |
| 25 | Service Account | [`25-serviceaccount`](./25-serviceaccount) | ⬜ |
| 26 | Service Mesh | [`26-servicemesh`](./26-servicemesh) | ⬜ |
| 27 | Monitoring | [`27-monitoring`](./27-monitoring) | ⬜ |
| 28 | Gerçek Hayat Projesi | [`28-proje`](./28-proje) | ⬜ |

📌 Hızlı erişim: [kubectl Cheatsheet](./cheatsheets/kubectl-cheatsheet.md) · [YAML Şablonları](./cheatsheets/yaml-sablonlari.md)

---

## 🗂️ Repo Yapısı

```
kubernetes-notes/
├── 01-giris/ ... 28-proje/     # Konu bazlı notlar (her klasörde README.md)
├── cheatsheets/                # kubectl komutları + hazır YAML şablonları
└── manifests/                  # Kendi denediğin .yaml dosyaları
```

## 🎯 Nasıl Kullanıyorum?

1. Dersteki konuya karşılık gelen klasörü açıyorum (isimler hocanın reposuyla aynı).
2. İçindeki `README.md`'de **Özet → Anahtar Kavramlar → Notlarım** bölümlerini dolduruyorum.
3. Denediğim YAML'leri `manifests/` altına atıyorum.
4. Konuyu bitirince yukarıdaki tabloda ⬜ → ✅ yapıyorum.

## 📖 Faydalı Kaynaklar

- [Kubernetes Resmi Dokümantasyon](https://kubernetes.io/docs/home/)
- [kubectl Cheat Sheet (resmi)](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Play with Kubernetes](https://labs.play-with-k8s.com/)

---

_Öğrenme amaçlı notlardır, eğitim ilerledikçe güncellenir._
