# 📦 manifests/

Ders boyunca kendi denediğin `.yaml` dosyalarını buraya koy.

Öneri isimlendirme: `konu-aciklama.yaml`
Örnek: `pod-nginx.yaml`, `deployment-web.yaml`, `service-nodeport.yaml`

```bash
kubectl apply -f manifests/pod-nginx.yaml
kubectl delete -f manifests/pod-nginx.yaml
```
