# Deploying Portainer on Kubernetes

* Create a Portainer Namespace:

Since Portainer was previously deployed in the portainer namespace, let's ensure it exists:

```
kubectl create namespace portainer
```

* Deploy Portainer Using a YAML Configuration:


Create a YAML file (portainer.yaml) with the following content to deploy Portainer:


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
    nodePort: 30000   # Adjust NodePort as needed
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
        image: portainer/portainer-ce:2.11.1
        ports:
        - containerPort: 9000
        volumeMounts:
        - mountPath: /data
          name: portainer-data
      volumes:
      - name: portainer-data
        emptyDir: {}

```


**Save the file and apply it using kubectl:**

```
kubectl apply -f portainer.yaml
```

This will deploy Portainer with a NodePort service configured to expose it externally.


* Access Portainer from Local Machine:


Now that Portainer is deployed, find out the IP address of any of your Kubernetes nodes (k8s-manager, k8s-node1, or k8s-node2). Use the internal IP address (INTERNAL-IP column from kubectl get nodes -o wide).

    * Determine the NodePort assigned to the Portainer service:

```
kubectl get svc -n portainer
```


Look for the portainer service in the portainer namespace and note down the NODE-PORT value under the PORT(S) column.

    * Access Portainer from your local browser using the IP address of the node and the NodePort:

```
http://<node-internal-ip>:<nodeport>
```



For example, if k8s-manager has an internal IP of 192.168.12.112 and the NodePort assigned to Portainer is 30000, you would access Portainer at:


```
http://192.168.12.112:30000
```

Replace 192.168.12.112 with the appropriate internal IP and 30000 with the NodePort you obtained.


# Notes:

* Ensure that any firewall rules or network policies allow traffic to reach the NodePort from your local machine.
* Consider securing Portainer with authentication and HTTPS if deploying in a production environment.


# Granting cluster-admin Role to Portainer



* Create a ClusterRoleBinding for cluster-admin:


Create a ClusterRoleBinding that binds the cluster-admin ClusterRole to the default service account in the portainer namespace:


```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portainer-cluster-admin-binding
subjects:
- kind: ServiceAccount
  name: default
  namespace: portainer
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

```

Save the above YAML to a file (e.g., portainer-cluster-admin-binding.yaml).

Apply the ClusterRoleBinding to grant cluster-admin permissions:


```
kubectl apply -f portainer-cluster-admin-binding.yaml

```

* Verification


After applying the cluster-admin role binding, Portainer should have full administrative access to your Kubernetes cluster, similar to what you can do with kubectl:

    * Verify that Portainer can perform administrative actions. For example, you can check nodes, deployments, pods, etc.:

```
kubectl get nodes
kubectl get deployments
kubectl get pods --all-namespaces
```


# Granting RBAC Permissions to List Nodes

* Create a ClusterRole for Node List Permissions:

`sudo nano portainer-node-list-role.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portainer-node-list-role
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]

```

`kubectl apply -f portainer-node-list-role.yaml`

* Bind the ClusterRole to Portainer's ServiceAccount:

`sudo nano portainer-node-list-role-binding.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portainer-node-list-role-binding
subjects:
- kind: ServiceAccount
  name: default
  namespace: portainer
roleRef:
  kind: ClusterRole
  name: portainer-node-list-role
  apiGroup: rbac.authorization.k8s.io

```

`kubectl apply -f portainer-node-list-role-binding.yaml`


# Granting Cluster Administrator Access to Portainer

* Create ClusterRole for Administrator Access:

`sudo nano portainer-admin-role.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portainer-admin-role
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]
- nonResourceURLs: ["*"]
  verbs: ["*"]
```

`kubectl apply -f portainer-admin-role.yaml`

* Bind the ClusterRole to Portainer's ServiceAccount:

`sudo nano portainer-admin-binding.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portainer-admin-binding
subjects:
- kind: ServiceAccount
  name: default
  namespace: portainer
roleRef:
  kind: ClusterRole
  name: portainer-admin-role
  apiGroup: rbac.authorization.k8s.io

```

`kubectl apply -f portainer-admin-binding.yaml`



