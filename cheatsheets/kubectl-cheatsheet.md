# 🚀 kubectl Cheatsheet

En sık kullanılan komutların hızlı referansı.

## Cluster & Node
```bash
kubectl cluster-info                 # Cluster bilgisi
kubectl get nodes                    # Node'ları listele
kubectl describe node <ad>           # Node detayı
kubectl top nodes                    # Kaynak kullanımı (metrics-server gerekli)
kubectl version --client             # kubectl sürümü
```

## Kaynak Görüntüleme (get / describe)
```bash
kubectl get pods                     # Pod'lar
kubectl get pods -o wide             # IP ve node ile
kubectl get all                      # Tüm kaynaklar
kubectl get pods -n <namespace>      # Belirli namespace
kubectl get pods -A                  # Tüm namespace'ler
kubectl get pods --watch             # Canlı izle
kubectl describe pod <ad>            # Detaylı inceleme
kubectl get pods -o yaml             # YAML çıktısı
kubectl get pods --show-labels       # Etiketlerle
```

## Oluşturma & Uygulama
```bash
kubectl apply -f dosya.yaml          # YAML uygula (tavsiye edilen)
kubectl create -f dosya.yaml         # Oluştur
kubectl run nginx --image=nginx      # Hızlı pod
kubectl create deployment web --image=nginx --replicas=3
kubectl delete -f dosya.yaml         # YAML'daki kaynakları sil
kubectl delete pod <ad>              # Tek kaynak sil
```

## Debug & İnceleme
```bash
kubectl logs <pod>                   # Log
kubectl logs <pod> -f                # Canlı log
kubectl logs <pod> -c <container>    # Belirli container
kubectl logs <pod> --previous        # Önceki container logu
kubectl exec -it <pod> -- /bin/bash  # Pod içine gir
kubectl exec <pod> -- <komut>        # Tek komut çalıştır
kubectl port-forward <pod> 8080:80   # Yerel porta bağla
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Ölçekleme & Güncelleme
```bash
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.26
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl edit deployment web          # Canlı düzenle
```

## Namespace & Context
```bash
kubectl get namespaces
kubectl create namespace dev
kubectl config get-contexts
kubectl config current-context
kubectl config set-context --current --namespace=dev
```

## İpuçları ⚡
```bash
# YAML şablonu üret (uygulamadan) — çok kullanışlı!
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml

# Kısaltmalar: po=pods, svc=services, deploy=deployments, ns=namespaces
kubectl get po
kubectl get svc

# Alias önerisi (~/.bashrc içine)
alias k=kubectl
```

## Açıklama & Yardım
```bash
kubectl explain pod                  # Kaynak alanlarını açıkla
kubectl explain pod.spec.containers  # İç içe alanlar
kubectl api-resources                # Tüm kaynak tipleri
```
