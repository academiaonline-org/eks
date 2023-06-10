In this lab exercise, we'll efficiently deploy the remaining components of the sample application using the capabilities of Kustomize. The following kustomization file demonstrates how you can reference other kustomizations and deploy multiple components collectively: /workspace/manifests/kustomization.yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- rabbitmq
- catalog
- carts
- checkout
- assets
- orders
- ui
- other
```
You're correct, the catalog API is included in this kustomization file, and we have already deployed it earlier. 
Since Kubernetes follows a declarative approach, we can reapply the manifests for the catalog API, and Kubernetes will recognize that all the resources already exist, resulting in no further action being taken.

Let's apply this kustomization to our cluster to deploy the remaining components:
```
$ kubectl apply -k /workspace/manifests
namespace/assets created
namespace/carts created
namespace/catalog unchanged
namespace/checkout created
namespace/orders created
namespace/other created
namespace/rabbitmq created
namespace/ui created
serviceaccount/assets created
serviceaccount/carts created
serviceaccount/catalog unchanged
serviceaccount/checkout created
serviceaccount/orders created
serviceaccount/rabbitmq created
serviceaccount/ui created
role.rbac.authorization.k8s.io/rabbitmq-endpoint-reader created
rolebinding.rbac.authorization.k8s.io/rabbitmq-endpoint-reader created
configmap/assets created
configmap/carts created
configmap/catalog unchanged
configmap/checkout created
configmap/orders created
configmap/dummy created
configmap/ui created
secret/catalog-db unchanged
secret/orders-db created
secret/rabbitmq created
secret/rabbitmq-config created
service/assets created
service/carts created
service/carts-dynamodb created
service/catalog unchanged
service/catalog-mysql unchanged
service/checkout created
service/checkout-redis created
service/orders created
service/orders-mysql created
service/rabbitmq created
service/rabbitmq-headless created
service/ui created
deployment.apps/assets created
deployment.apps/carts created
deployment.apps/carts-dynamodb created
deployment.apps/catalog configured
deployment.apps/checkout created
deployment.apps/checkout-redis created
deployment.apps/orders created
deployment.apps/orders-mysql created
deployment.apps/ui created
statefulset.apps/catalog-mysql unchanged
statefulset.apps/rabbitmq created
```
Once the deployment of the remaining components is complete, we can utilize `kubectl wait` to ensure that all the components have started before proceeding:
```
$ kubectl wait --for=condition=Ready --timeout=180s po -l app.kubernetes.io/created-by=eks-workshop -A
pod/assets-5475587bf5-h7569 condition met
pod/carts-9ff78bd6c-2nr8m condition met
pod/carts-dynamodb-cc5bf4649-c7695 condition met
pod/catalog-69c6bf994c-jrssv condition met
pod/catalog-mysql-0 condition met
pod/checkout-f4954f4d7-bm8rh condition met
pod/checkout-redis-6cfd7d8787-vggkv condition met
pod/orders-6fb99f4c7d-n9xt4 condition met
pod/orders-mysql-745b5b57c6-shbhf condition met
pod/ui-fddcf8d7b-bw2gl condition met
```
Following the deployment, we will have a dedicated Namespace for each of our application components:
```
$ kubectl get ns -l app.kubernetes.io/created-by=eks-workshop
NAME       STATUS   AGE
assets     Active   2m17s
carts      Active   2m17s
catalog    Active   26m
checkout   Active   2m17s
orders     Active   2m17s
other      Active   2m17s
rabbitmq   Active   2m17s
ui         Active   2m17s
```
We can also view all the Deployments that have been created for the components:
```
$ kubectl get deploy -l app.kubernetes.io/created-by=eks-workshop -A
NAMESPACE   NAME             READY   UP-TO-DATE   AVAILABLE   AGE
assets      assets           1/1     1            1           3m37s
carts       carts            1/1     1            1           3m37s
carts       carts-dynamodb   1/1     1            1           3m37s
catalog     catalog          1/1     1            1           28m
checkout    checkout         1/1     1            1           3m36s
checkout    checkout-redis   1/1     1            1           3m36s
orders      orders           1/1     1            1           3m36s
orders      orders-mysql     1/1     1            1           3m36s
ui          ui               1/1     1            1           3m36s
```
Great! The sample application is now successfully deployed and ready to serve as the foundation for the remaining labs in this workshop.
