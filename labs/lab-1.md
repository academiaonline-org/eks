Before we proceed, let's first examine the current Namespaces in our EKS cluster:
```
$ kubectl get ns
NAME                            STATUS   AGE
ack-ec2                         Active   6h20m
ack-rds                         Active   6h19m
argocd                          Active   6h19m
aws-for-fluent-bit              Active   6h19m
aws-load-balancer-controller    Active   6h19m
cert-manager                    Active   6h19m
crossplane-system               Active   6h19m
default                         Active   6h24m
grafana                         Active   6h14m
karpenter                       Active   6h16m
kube-node-lease                 Active   6h24m
kube-public                     Active   6h24m
kube-system                     Active   6h24m
kubecost                        Active   6h16m
opentelemetry-operator-system   Active   6h19m
```
All of the listed entries are Namespaces for system components that were already installed. We'll filter out these system Namespaces and only focus on the ones we've created by using Kubernetes labels:
```
$ kubectl get ns -l app.kubernetes.io/created-by=eks-workshop
No resources found
```
The first step we'll take is to deploy the catalog component independently. You can find the manifests for this component in the /workspace/manifests/catalog directory:
```
$ ls /workspace/manifests/catalog
configMap.yaml  deployment.yaml  kustomization.yaml  namespace.yaml  secrets.yaml  serviceAccount.yaml  service-mysql.yaml  service.yaml  statefulset-mysql.yaml
```
These manifests include the Deployment for the catalog API: /workspace/manifests/catalog/deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: catalog
      app.kubernetes.io/instance: catalog
      app.kubernetes.io/component: service
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/name: catalog
        app.kubernetes.io/instance: catalog
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
    spec:
      serviceAccountName: catalog
      securityContext:
        fsGroup: 1000
      containers:
        - name: catalog
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: catalog-db
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: catalog-db
                  key: password
          envFrom:
            - configMapRef:
                name: catalog
          securityContext:
            capabilities:
              drop:
              - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
          image: "public.ecr.aws/aws-containers/retail-store-sample-catalog:0.4.0"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            successThreshold: 3
            periodSeconds: 5
          resources:
            limits:
              memory: 512Mi
            requests:
              cpu: 250m
              memory: 512Mi
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
      volumes:
        - name: tmp-volume
          emptyDir:
            medium: Memory
```
This Deployment outlines the desired state of the catalog API component, including:
* Utilization of the public.ecr.aws/aws-containers/retail-store-sample-catalog container image
* Operation of a single replica
* Exposure of the container on port 8080, named 'http'
* Execution of probes/healthchecks against the /health path
* Request for a specific amount of CPU and memory, allowing the Kubernetes scheduler to place it on a node with sufficient resources
* Application of labels to the Pods for referencing by other resources

The manifests also incorporate the Service employed by other components for accessing the catalog API: /workspace/manifests/catalog/service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: catalog
  labels:
    app.kubernetes.io/created-by: eks-workshop
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: catalog
    app.kubernetes.io/instance: catalog
    app.kubernetes.io/component: service
```
This Service:
* Selects catalog Pods by using labels that match those specified in the aforementioned Deployment
* Exposes itself on port 80
* Targets the 'http' port made accessible by the Deployment, which corresponds to port 8080

Now, let's proceed to create the catalog component:
```
$ kubectl apply -k /workspace/manifests/catalog
namespace/catalog created
serviceaccount/catalog created
configmap/catalog created
secret/catalog-db created
service/catalog created
service/catalog-mysql created
deployment.apps/catalog created
statefulset.apps/catalog-mysql created
```
Now, we'll observe a newly created Namespace:
```
$ kubectl get ns -l app.kubernetes.io/created-by=eks-workshop
NAME      STATUS   AGE
catalog   Active   67s
```
We can examine the Pods currently operating within this namespace:
```
$ kubectl get po -n catalog
NAME                       READY   STATUS    RESTARTS        AGE
catalog-69c6bf994c-fsjgd   1/1     Running   2 (2m13s ago)   2m37s
catalog-mysql-0            1/1     Running   0               2m37s
```
Observe that we have a Pod for our catalog API and another for the MySQL database. The catalog Pod is indicating a status of CrashLoopBackOff. This is due to its requirement of being able to connect to the catalog-mysql Pod prior to its own start, and Kubernetes will continually restart it until this connection is established. Fortunately, we can use kubectl wait to track specific Pods until they reach a Ready state:
```
$ kubectl wait --for=condition=Ready po --all -n catalog --timeout=180s
pod/catalog-69c6bf994c-fsjgd condition met
pod/catalog-mysql-0 condition met
```
Now that the Pods are operational, we can examine their logs. For instance, let's take a look at the logs for the catalog API:
```
$ kubectl logs -n catalog deploy/catalog
2023/06/10 23:08:56 Running database migration...
2023/06/10 23:08:56 Schema migration applied
2023/06/10 23:08:56 Connecting to catalog-mysql:3306/catalog?timeout=5s
2023/06/10 23:08:56 invalid connection config: missing required peer IP or hostname
2023/06/10 23:08:56 Connected
2023/06/10 23:08:56 Connecting to catalog-mysql:3306/catalog?timeout=5s
2023/06/10 23:08:56 invalid connection config: missing required peer IP or hostname
2023/06/10 23:08:56 Connected
```
You can "tail" the kubectl logs in real time by using the '-f' option with the command. (Press CTRL-C to stop tailing the logs).

In Kubernetes, we have the ease of horizontally scaling the number of catalog Pods:
```
$ kubectl scale -n catalog --replicas 3 deploy/catalog
deployment.apps/catalog scaled
$ kubectl wait --for=condition=Ready pods --all -n catalog --timeout=180s
pod/catalog-69c6bf994c-fsjgd condition met
pod/catalog-69c6bf994c-jrssv condition met
pod/catalog-69c6bf994c-v7jch condition met
pod/catalog-mysql-0 condition met
```
The manifests we deployed also generate a Service for each of our application and MySQL Pods. These Services can be utilized by other components within the cluster to establish connections:
```
$ kubectl get svc -n catalog
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
catalog         ClusterIP   172.20.184.159   <none>        80/TCP     10m
catalog-mysql   ClusterIP   172.20.9.72      <none>        3306/TCP   10m
```
As these Services are internal to the cluster, direct access to them from the Internet or even the VPC is not possible. However, we can utilize the `exec` command to gain access to an existing Pod in the EKS cluster, enabling us to verify the functionality of the catalog API:
```
$ kubectl -n catalog exec deploy/catalog -- curl -s catalog.catalog.svc/catalogue | jq .                                             
[
  {
    "id": "510a0d7e-8e83-4193-b483-e27e09ddc34d",
    "name": "Gentleman",
    "description": "Touch of class for a bargain.",
    "imageUrl": "/assets/gentleman.jpg",
    "price": 795,
    "count": 51,
    "tag": [
      "dress"
    ]
  },
  {
    "id": "6d62d909-f957-430e-8689-b5129c0bb75e",
    "name": "Pocket Watch",
    "description": "Properly dapper.",
    "imageUrl": "/assets/pocket_watch.jpg",
    "price": 385,
    "count": 33,
    "tag": [
      "dress"
    ]
  },
  {
    "id": "808a2de1-1aaa-4c25-a9b9-6612e8f29a38",
    "name": "Chronograf Classic",
    "description": "Spend that IPO money",
    "imageUrl": "/assets/chrono_classic.jpg",
    "price": 5100,
    "count": 9,
    "tag": [
      "dress",
      "luxury"
    ]
  },
  {
    "id": "a0a4f044-b040-410d-8ead-4de0446aec7e",
    "name": "Wood Watch",
    "description": "Looks like a tree",
    "imageUrl": "/assets/wood_watch.jpg",
    "price": 50,
    "count": 115,
    "tag": [
      "casual"
    ]
  },
  {
    "id": "ee3715be-b4ba-11ea-b3de-0242ac130004",
    "name": "Smart 3.0",
    "description": "Can tell you what you want for breakfast",
    "imageUrl": "/assets/smart_1.jpg",
    "price": 650,
    "count": 9,
    "tag": [
      "smart",
      "dress"
    ]
  },
  {
    "id": "f4ebd070-b4ba-11ea-b3de-0242ac130004",
    "name": "FitnessX",
    "description": "Touch of class for a bargain.",
    "imageUrl": "/assets/smart_2.jpg",
    "price": 180,
    "count": 76,
    "tag": [
      "dress",
      "smart"
    ]
  }
]
```
If everything is working correctly, you should receive a JSON payload containing product information. Congratulations on successfully deploying your first microservice to Kubernetes with EKS!
