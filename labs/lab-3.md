Kustomize is a powerful tool that enables you to manage Kubernetes manifest files using declarative "kustomization" files. With Kustomize, you can define "base" manifests for your Kubernetes resources and then apply changes using composition and customization. It simplifies the process of making cross-cutting changes across multiple resources, making it easier to manage and maintain your Kubernetes deployments.

For instance, let's examine the following manifest file for the checkout Deployment: /workspace/manifests/checkout/deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/type: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: checkout
      app.kubernetes.io/instance: checkout
      app.kubernetes.io/component: service
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "8080"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/name: checkout
        app.kubernetes.io/instance: checkout
        app.kubernetes.io/component: service
        app.kubernetes.io/created-by: eks-workshop
    spec:
      serviceAccountName: checkout
      securityContext:
        fsGroup: 1000
      containers:
        - name: checkout
          envFrom:
            - configMapRef:
                name: checkout
          securityContext:
            capabilities:
              drop:
              - ALL
            readOnlyRootFilesystem: true
          image: "public.ecr.aws/aws-containers/retail-store-sample-checkout:0.4.0"
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
This file has already been utilized in the previous lab. However, let's suppose we wish to horizontally scale this component by modifying the `replicas` field from 1 to 3 using Kustomize. Instead of manually editing the YAML file, we can employ Kustomize to make this update to the `spec/replicas` field.
To achieve this, we will apply the following kustomization: /workspace/modules/introduction/kustomize/deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
spec:
  replicas: 3
```
You can utilize the kustomize CLI to generate the final Kubernetes YAML that applies this kustomization:
```
$ kustomize build /workspace/modules/introduction/kustomize
```
This will generate multiple YAML files that represent the final manifests, which can be directly applied to Kubernetes. Let's demonstrate this by piping the output from kustomize directly to kubectl:
```
$ kustomize build /workspace/modules/introduction/kustomize | kubectl apply -f -
namespace/checkout unchanged
serviceaccount/checkout unchanged
configmap/checkout unchanged
service/checkout unchanged
service/checkout-redis unchanged
deployment.apps/checkout configured
deployment.apps/checkout-redis unchanged
```
When you execute the command, you'll observe that several checkout-related resources are marked as "unchanged", while the deployment.apps/checkout is indicated as "configured". This behavior is intentional because we only want to apply changes to the checkout deployment. This occurs because running the previous command applies two files: the Kustomize deployment.yaml mentioned earlier, and the kustomization.yaml file, which matches all files in the /workspace/manifests/checkout folder. The patches field specifies the specific file to be patched: /workspace/modules/introduction/kustomize/kustomization.yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../../manifests/checkout
patches:
- deployment.yaml
```
To verify that the number of replicas has been updated, execute the following command:
```
$ kubectl get po -n checkout -l app.kubernetes.io/component=service
NAME                       READY   STATUS    RESTARTS   AGE
checkout-f4954f4d7-bm8rh   1/1     Running   0          21m
checkout-f4954f4d7-hcrhf   1/1     Running   0          4m29s
checkout-f4954f4d7-tnvgp   1/1     Running   0          4m29s
```
While we utilized the kustomize CLI directly in this section, it's worth noting that Kustomize is also seamlessly integrated with the kubectl CLI. You can apply the kustomization directly with the command `kubectl apply -k <kustomization_directory>`. This approach is employed throughout this workshop to simplify the application of changes to manifest files and clearly display the changes being applied.
Let's give it a try. You can apply the kustomization using the following command:
```
$ kubectl apply -k /workspace/modules/introduction/kustomize
namespace/checkout unchanged
serviceaccount/checkout unchanged
configmap/checkout unchanged
service/checkout unchanged
service/checkout-redis unchanged
deployment.apps/checkout unchanged
deployment.apps/checkout-redis unchanged
```
To restore the application manifests to their initial state, you can simply apply the original set of manifests again:
```
$ kubectl apply -k /workspace/manifests
namespace/assets unchanged
namespace/carts unchanged
namespace/catalog unchanged
namespace/checkout unchanged
namespace/orders unchanged
namespace/other unchanged
namespace/rabbitmq unchanged
namespace/ui unchanged
serviceaccount/assets unchanged
serviceaccount/carts unchanged
serviceaccount/catalog unchanged
serviceaccount/checkout unchanged
serviceaccount/orders unchanged
serviceaccount/rabbitmq configured
serviceaccount/ui unchanged
role.rbac.authorization.k8s.io/rabbitmq-endpoint-reader unchanged
rolebinding.rbac.authorization.k8s.io/rabbitmq-endpoint-reader unchanged
configmap/assets unchanged
configmap/carts unchanged
configmap/catalog unchanged
configmap/checkout unchanged
configmap/orders unchanged
configmap/dummy unchanged
configmap/ui unchanged
secret/catalog-db unchanged
secret/orders-db unchanged
secret/rabbitmq unchanged
secret/rabbitmq-config unchanged
service/assets unchanged
service/carts unchanged
service/carts-dynamodb unchanged
service/catalog unchanged
service/catalog-mysql unchanged
service/checkout unchanged
service/checkout-redis unchanged
service/orders unchanged
service/orders-mysql unchanged
service/rabbitmq configured
service/rabbitmq-headless unchanged
service/ui unchanged
deployment.apps/assets unchanged
deployment.apps/carts unchanged
deployment.apps/carts-dynamodb unchanged
deployment.apps/catalog unchanged
deployment.apps/checkout configured
deployment.apps/checkout-redis unchanged
deployment.apps/orders unchanged
deployment.apps/orders-mysql unchanged
deployment.apps/ui configured
statefulset.apps/catalog-mysql unchanged
statefulset.apps/rabbitmq configured
```
To gain further knowledge about Kustomize, I recommend referring to the official Kubernetes documentation. It provides comprehensive information and guidance on working with Kustomize.
