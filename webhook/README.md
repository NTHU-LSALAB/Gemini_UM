
## Prerequisites

Kubernetes 1.9.0 or above with the `admissionregistration.k8s.io/v1beta1` API enabled. Verify that by the following command:
```
kubectl api-versions | grep admissionregistration.k8s.io/v1beta1
```
The result should be:
```
admissionregistration.k8s.io/v1beta1
```

In addition, the `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` admission controllers should be added and listed in the correct order in the admission-control flag of kube-apiserver.

## Build

Build and push docker image
   
```
./build
```

```
./deployment/webhook-create-signed-cert.sh
kubectl get secret admission-webhook-example-certs
kubectl create -f deployment/deployment.yaml
kubectl create -f deployment/service.yaml
kubectl create -f deployment/rbac.yaml
kubectl label namespace default admission-webhook-example=enabled
cat ./deployment/mutatingwebhook.yaml | ./deployment/webhook-patch-ca-bundle.sh > ./deployment/mutatingwebhook-ca-bundle.yaml
kubectl create -f deployment/mutatingwebhook-ca-bundle.yaml
```
For more information about the webhook, you can check this [blog](https://www.qikqiak.com/post/k8s-admission-webhook/)
