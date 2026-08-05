# Blue-Green Deployment on Kubernetes

Blue-Green deployment is simpler than Canary because you don't split traffic. Instead, you keep two complete environments and switch all traffic from one to the other in a single step.

- **Blue** → Current production version
- **Green** → New version

## Architecture

Initially, all traffic flows through the Service to the Blue Pods:

```
                 Users
                   │
                   ▼
               Service
                   │
                   ▼
              Blue Pods (v1)
```

Green Pods (v2) are running, but they receive no traffic.

## Step 1: Blue Deployment

`blue-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=Blue Version"
        ports:
        - containerPort: 5678
```

## Step 2: Green Deployment

`green-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
        - "-text=Green Version"
        ports:
        - containerPort: 5678
```

## Step 3: Service

Initially, the Service points to Blue.

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue
  ports:
  - port: 80
    targetPort: 5678
```

Traffic flow:

```
Users
  │
  ▼
Service
  │
  ▼
Blue Pods
```

## Step 4: Deploy Green

Apply the Green deployment:

```bash
kubectl apply -f green-deployment.yaml
```

Now:

```
Blue Pods   ✓ Receiving traffic
Green Pods  ✓ Running
            ✗ No traffic
```

You can test Green directly using:

```bash
kubectl port-forward deployment/myapp-green 8080:5678
```

Then visit:

```
http://localhost:8080
```

## Step 5: Switch Traffic

When Green is ready, change the Service selector.

From:

```yaml
selector:
  app: myapp
  version: blue
```

To:

```yaml
selector:
  app: myapp
  version: green
```

Apply:

```bash
kubectl apply -f service.yaml
```

Immediately:

```
Users
  │
  ▼
Service
  │
  ▼
Green Pods
```

All users now use Version 2.

## Rollback

If Green has a bug, change:

```yaml
version: green
```

back to:

```yaml
version: blue
```

Run:

```bash
kubectl apply -f service.yaml
```

Traffic instantly returns to Blue. No Pods are restarted.

## Deploy

```bash
kubectl apply -f blue-deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f green-deployment.yaml
```

## Difference from Canary

**Canary**

```
90% ---> v1
10% ---> v2
```

Traffic is split.

**Blue-Green**

Before:

```
100% ---> Blue
```

After:

```
100% ---> Green
```

No traffic splitting.

## Interview Question

**Q: How does Kubernetes know to switch traffic?**

**A:** Kubernetes Services use label selectors. During a Blue-Green deployment, we don't modify the Pods. We simply update the Service's selector from `version: blue` to `version: green`. The Service immediately starts routing all traffic to the Green Pods. Both versions run simultaneously, but the Service determines which version receives production traffic.

---

## Blue-Green Deployment Theory

Blue-Green Deployment is one of the safest deployment strategies because you never update the running application. Instead, you run two identical environments, and only one serves users at a time.

- **Blue** = Current production version
- **Green** = New version

### Step 1: Blue is Live

```
                Users
                  │
                  ▼
              Service/Load Balancer
                  │
                  ▼
           Blue Environment (v1)
```

Everyone is using Version 1.

### Step 2: Deploy Green

Version 2 is deployed, but no users are sent to it yet.

```
                Users
                  │
                  ▼
              Service
                  │
                  ▼
            Blue (v1)   ← Live

            Green (v2)  ← Running but no traffic
```

At this point, Green is running just like it would in production.

### Step 3: Test Green

Since Green is already running, you can test it:

- Login
- Check APIs
- Verify database connections
- Run smoke tests
- Ensure everything works

Users are still using Blue, so they are unaffected.

### Step 4: Switch Traffic

Once Green is verified, all traffic is switched. In Kubernetes, this is usually done by changing the Service selector.

Before:

```
Users
   │
   ▼
Blue
```

After:

```
Users
   │
   ▼
Green
```

The switch is almost instantaneous.

### Step 5: Rollback (if needed)

If users report problems after the switch, simply point the Service back to Blue:

```
Users
   │
   ▼
Blue
```

Because Blue was never deleted, rollback takes only a few seconds.

## Why Use Blue-Green?

### Advantages

- ✅ Near zero downtime
- ✅ Very fast rollback
- ✅ New version can be fully tested before users see it
- ✅ Low deployment risk

### Disadvantages

- ❌ Requires double the infrastructure because both environments run at the same time
- ❌ Database schema changes can be tricky if Blue and Green expect different database structures

## Blue-Green vs Canary

**Blue-Green**

```
100% Users
      │
      ▼
Blue

     ↓
  Switch
     ↓

100% Users
      │
      ▼
Green
```

Everyone moves at once.

**Canary**

```
90% Users ──► Version 1
10% Users ──► Version 2
```

Traffic is split gradually.

## Interview Answer

**"Explain Blue-Green Deployment."**

> Blue-Green Deployment maintains two identical production environments. The Blue environment serves the current production traffic, while the Green environment runs the new application version. After deploying and testing the Green environment, traffic is switched from Blue to Green, typically by updating a load balancer or Kubernetes Service. If any issues occur, traffic can be immediately switched back to Blue, making rollback fast and minimizing downtime.
