# 13.4 инструменты для упрощения написания конфигурационных файлов. Helm и Jsonnet  

## Задание 1: подготовить helm чарт для приложения  

Упакуем приложение из предыдущих занятий в чарт для деплоя в разные окружения.  
Исходники чарта [здесь](charts/my-app)  
Проверка темплейтов:  
```shell
% helm lint my-app 
==> Linting my-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

% helm template my-app
---
# Source: my-app/templates/backend-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: app1
spec:
  ports:
    - name: web9000
      port: 9000
      nodePort: 30081
  selector:
    app: backend
  type: NodePort
---
# Source: my-app/templates/db-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: db
  namespace: app1
spec:
  ports:
    - name: pgport
      port: 5432
  selector:
    app: db
---
# Source: my-app/templates/frontend-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: app1
spec:
  ports:
    - name: web8000
      port: 8000
      targetPort: 80
      nodePort: 30080
  selector:
    app: frontend
  type: NodePort
---
# Source: my-app/templates/backend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: backend
  name: backend
  namespace: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - image: "belas80/backend:latest"
          imagePullPolicy: IfNotPresent
          name: backend
          ports:
            - containerPort: 9000
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
      terminationGracePeriodSeconds: 30
---
# Source: my-app/templates/frontend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
  namespace: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: "belas80/frontend:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
---
# Source: my-app/templates/db-sts.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: db
  name: db
  namespace: app1
spec:
  serviceName: db
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - image: "postgres:13-alpine"
          imagePullPolicy: IfNotPresent
          name: db
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              value: postgres
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_DB
              value: news
          resources:
            limits:
              cpu: 400m
              memory: 512Mi
            requests:
              cpu: 200m
              memory: 256Mi
      terminationGracePeriodSeconds: 30
```

