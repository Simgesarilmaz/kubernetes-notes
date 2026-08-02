# 📄 Hazır YAML Şablonları

Kopyala-yapıştır ile başlayabileceğin temel manifest'ler.

## Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

## Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
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
          image: nginx:1.25
          ports:
            - containerPort: 80
```

## Service (ClusterIP)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP        # NodePort / LoadBalancer da olabilir
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

## Service (NodePort)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080     # 30000-32767 arası
```

## ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
```

## Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: MTIzNA==      # base64: echo -n '1234' | base64
```

## PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

> 💡 Not: Aklında tutamadığın alanlar için `kubectl explain <kaynak>` kullan
> veya `--dry-run=client -o yaml` ile taslak üret.
