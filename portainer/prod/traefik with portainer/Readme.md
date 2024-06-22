# Step 1: Set Up DNS Records


Ensure that you have the following DNS records set up for your domain:

    * `traefik.example.com` pointing to the IP address of your Kubernetes cluster.
    * `portainer.example.com` pointing to the same IP address.

# Step 2: Install Traefik and Configure Ingress

    * Install Traefik using Helm:

```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
kubectl create namespace traefik
helm install traefik traefik/traefik --namespace=traefik --set dashboard.ingressRoute=true --set="additionalArguments={--entryPoints.web.address=:80,--entryPoints.websecure.address=:443,--entryPoints.web.http.redirections.entryPoint.to=websecure,--entryPoints.web.http.redirections.entryPoint.scheme=https,--entryPoints.websecure.http.tls.certResolver=myresolver,--certificatesResolvers.myresolver.acme.email=youremail@example.com,--certificatesResolvers.myresolver.acme.storage=/data/acme.json,--certificatesResolvers.myresolver.acme.tlsChallenge=true}"
```

    * Create an IngressRoute for the Traefik dashboard:

```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`traefik.example.com`)
      kind: Rule
      services:
        - name: api@internal
      middlewares:
        - name: auth-basic
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: auth-basic
  namespace: traefik
spec:
  basicAuth:
    secret: auth-secret
---
apiVersion: v1
kind: Secret
metadata:
  name: auth-secret
  namespace: traefik
type: Opaque
data:
  users: $(htpasswd -nb admin <password> | base64)

```


**Replace <password> with your desired password.**

* Apply the YAML:

```
kubectl apply -f traefik-dashboard-ingress.yaml
```


# Step 3: Deploy Portainer

* Create a namespace for Portainer:
 
```
kubectl create namespace portainer
```

* Deploy Portainer:

```
apiVersion: v1
kind: Service
metadata:
  name: portainer
  namespace: portainer
spec:
  ports:
  - name: http
    port: 9000
    targetPort: 9000
    nodePort: 30000
  selector:
    app: portainer
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portainer
  namespace: portainer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: portainer
  template:
    metadata:
      labels:
        app: portainer
    spec:
      containers:
      - name: portainer
        image: portainer/portainer-ce:latest
        ports:
        - containerPort: 9000
        volumeMounts:
        - mountPath: /data
          name: portainer-data
      volumes:
      - name: portainer-data
        persistentVolumeClaim:
          claimName: portainer-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: portainer-pvc
  namespace: portainer
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

```


**Save this file as portainer-deployment.yaml and apply it:**

```
kubectl apply -f portainer-deployment.yaml
```

* Create an IngressRoute for Portainer:

```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: portainer
  namespace: portainer
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`portainer.example.com`)
      kind: Rule
      services:
        - name: portainer
          port: 9000

```


**Save this file as portainer-ingress.yaml and apply it:**


```
kubectl apply -f portainer-ingress.yaml
```

* Access Portainer:


Go to `http://portainer.example.com` in your web browser and set up your Portainer admin account.

