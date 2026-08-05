I deploy the new version alongside the existing version. Each version has its own Deployment and Service. Then I configure an Ingress controller or a service mesh to route a small percentage of traffic, such as 10%, to the new version. After monitoring the application's health and performance, I gradually increase the traffic to 30%, 50%, and eventually 100%. If any issues are detected, I immediately route all traffic back to the old version.

# Canary Deployment with NGINX Ingress Controller

A complete minimal example of a Canary deployment using the NGINX Ingress Controller on Kubernetes.

## Prerequisites

- A Kubernetes cluster (Minikube, Kind, or EKS)
- NGINX Ingress Controller installed

## 1. v1 Deployment

`deployment-v1.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Version 1"
        ports:
        - containerPort: 5678
```

## 2. v1 Service

`service-v1.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-v1
spec:
  selector:
    app: myapp
    version: v1
  ports:
  - port: 80
    targetPort: 5678
```

## 3. v2 Deployment

`deployment-v2.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Version 2"
        ports:
        - containerPort: 5678
```

## 4. v2 Service

`service-v2.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-v2
spec:
  selector:
    app: myapp
    version: v2
  ports:
  - port: 80
    targetPort: 5678
```

## 5. Normal Ingress

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-v1
            port:
              number: 80
```

Initially:

```
100% --> Version 1
```

## 6. Canary Ingress

`canary-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-v2
            port:
              number: 80
```

This means:

```
80% ---> Version 1
20% ---> Version 2
```

## Deploy

```bash
kubectl apply -f deployment-v1.yaml
kubectl apply -f service-v1.yaml

kubectl apply -f deployment-v2.yaml
kubectl apply -f service-v2.yaml

kubectl apply -f ingress.yaml
kubectl apply -f canary-ingress.yaml
```

## Test

Run multiple requests:

```bash
curl http://myapp.local
```

You'll sometimes get:

```
Hello from Version 1
```

and sometimes:

```
Hello from Version 2
```

because 20% of the traffic is routed to Version 2.

## Increase Traffic

Change:

```yaml
nginx.ingress.kubernetes.io/canary-weight: "20"
```

to:

```yaml
nginx.ingress.kubernetes.io/canary-weight: "50"
```

Now:

```
50% ---> v1
50% ---> v2
```

Later, set it to:

```yaml
nginx.ingress.kubernetes.io/canary-weight: "100"
```

Now all traffic goes to Version 2.
