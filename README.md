

---

# 🚀 **AWS EKS Kubernetes Project - Complete Step by Step**

## 🎯 **Project Overview**

Hum bana rahe hain:
✅ **MySQL Database** (StatefulSet)
✅ **Python Flask Backend API** (Deployment)
✅ **Nginx Frontend** (Deployment)
✅ **Ingress** (to route traffic)
✅ **Namespace Isolation**

---

## 🛠️ **STEP 0: Pre-requisites**

✅ **AWS EKS Cluster Ready**
✅ `kubectl` connected (`kubectl get nodes` shows nodes Ready)
✅ IAM permissions (to create LoadBalancers, etc.)

**Check:**

```bash
kubectl get nodes
```

---

## 🛠️ **STEP 1: Create Namespace**

```bash
kubectl create namespace myproject
```

Check:

```bash
kubectl get ns
```

---

## 🛠️ **STEP 2: Deploy MySQL Database**

### 2.1 StatefulSet + PVC + Service

👉 File: `mysql-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: myproject
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpassword
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: myproject
spec:
  ports:
    - port: 3306
  clusterIP: None
  selector:
    app: mysql
```

**Apply:**

```bash
kubectl apply -f mysql-deployment.yaml
```

Check:

```bash
kubectl get pods -n myproject
```

---

## 🛠️ **STEP 3: Deploy Backend API**

### 3.1 ConfigMap

👉 File: `backend-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: myproject
data:
  DB_HOST: mysql
  DB_USER: root
  DB_NAME: testdb
```

Apply:

```bash
kubectl apply -f backend-configmap.yaml
```

---

### 3.2 Secret

👉 File: `backend-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: myproject
type: Opaque
data:
  DB_PASSWORD: cm9vdHBhc3N3b3Jk
```

*(base64 of `rootpassword`)*

Apply:

```bash
kubectl apply -f backend-secret.yaml
```

---

### 3.3 Deployment

👉 File: `backend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myproject
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: bobhatevishal/flask-mysql-demo:latest   # Example image
        ports:
        - containerPort: 5000
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_HOST
        - name: DB_USER
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_USER
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_NAME
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: DB_PASSWORD
```

Apply:

```bash
kubectl apply -f backend-deployment.yaml
```

---

### 3.4 Service

👉 File: `backend-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: myproject
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc -n myproject
```

---

## 🛠️ **STEP 4: Deploy Frontend Nginx**

### 4.1 ConfigMap

👉 File: `frontend-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-html
  namespace: myproject
data:
  index.html: |
    <!DOCTYPE html>
    <html>
      <head><title>Kubernetes Frontend</title></head>
      <body>
        <h1>Hello from Nginx Frontend on AWS EKS!</h1>
      </body>
    </html>
```

Apply:

```bash
kubectl apply -f frontend-configmap.yaml
```

---

### 4.2 Deployment

👉 File: `frontend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: myproject
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: frontend-html
```

Apply:

```bash
kubectl apply -f frontend-deployment.yaml
```

---

### 4.3 Service

👉 File: `frontend-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: myproject
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f frontend-service.yaml
```

---

## 🛠️ **STEP 5: Ingress Controller & Ingress Resource**

### 5.1 Install Nginx Ingress Controller (if not installed)

**Using Helm (recommended):**

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Check:

```bash
kubectl get pods -n ingress-nginx
```

Wait till all pods are **Running**.

---

### 5.2 Ingress Resource

👉 File: `myproject-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myproject-ingress
  namespace: myproject
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 5000
```

Apply:

```bash
kubectl apply -f myproject-ingress.yaml
```

---

### 5.3 Get External IP

Check:

```bash
kubectl get ingress -n myproject
```

You will see:

```
NAME                CLASS   HOSTS   ADDRESS          PORTS   AGE
myproject-ingress   nginx   *       <EXTERNAL_IP>    80      1m
```

Open `http://<EXTERNAL_IP>` in browser.
✅ You should see the frontend HTML.

✅ `http://<EXTERNAL_IP>/api/` routes to backend API.

---

## 🛠️ **STEP 6: Test Everything**

Check pods:

```bash
kubectl get pods -n myproject
```

Check services:

```bash
kubectl get svc -n myproject
```

Test:

* `/` – Nginx frontend
* `/api/` – Flask API

---

## 🎁 **Bonus: Autoscaling Backend**

Enable metrics server first:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Then create HPA:

```bash
kubectl autoscale deployment backend -n myproject --cpu-percent=50 --min=2 --max=5
```

---

