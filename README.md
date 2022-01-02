# Kubernetes Admission Webhook example

This tutoral shows how to build and deploy an [AdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks).

The Kubernetes [documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) contains a common set of recommended labels that allows tools to work interoperably, describing objects in a common manner that all tools can understand. In addition to supporting tooling, the recommended labels describe applications in a way that can be queried.
In our validating webhook example we make these labels required on deployments and services, so this webhook rejects every deployment and every service that doesn’t have these labels set. The mutating webhook in the example adds all the missing required labels with `not_available` set as the value.

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

Set your docker user to terminal:

```
export DOCKER_USER=YOUR_DOCKER_USER
```

Build and push docker image according to your arch ( It is available linux ARM64 and AMD64 )
   
```
./build-arm64 
```
OR
```
./build-amd64 
```

## Deploy

Create a CSR and Secret:

```
./deployment/webhook-create-signed-cert.sh
```

Check if the secret is created:

```
kubectl get secret admission-webhook-example-certs
```

Deploy the rbac, deployment and service:

```
kubectl apply -f deployment/rbac.yaml
```

Check the correct image ( ARM64 OR AMD64 )
```
kubectl create -f deployment/deployment.yaml
```
```
kubectl create -f deployment/service.yaml
```

## Validation Webhook

Important: Remove the Mutating Webhook (if deployed)
It is necessary to remove the validation webhook to avoid conflict, by the logic used in the validation and the mutating

```
kubectl apply -f deployment/mutatingwebhook-ca-bundle.yaml
```

Configure Webhook

It will create a file "deployment/validatingwebhook-ca-bundle.yaml" copy from "deployment/validatingwebhook.yaml" but replacing the CA_BUNDLE placeholder.

```
cat ./deployment/validatingwebhook.yaml | ./deployment/webhook-patch-ca-bundle.sh > ./deployment/validatingwebhook-ca-bundle.yaml
```

Apply the ValidatingWebhookConfiguration:

```
kubectl apply -f deployment/validatingwebhook-ca-bundle.yaml
```

## To test

The validation webhook will validate deployments and services created in namespaces with the label "admission-webhook-example=enabled":

```
kubectl label namespace default admission-webhook-example=enabled
```

Create a deployment or service without the required label ( the validatingwebhook will validate ) and a error will occur:

```
kubectl apply -f deployment/test-deploy.yaml
```

This other command will work:

```
kubectl apply -f deployment/test-deploy-with-labels.yaml
```

To bypass the validating webhook

```
kubectl apply -f deployment/test-deploy-bypass-validation.yaml
```


## Mutating Webhook

Important: Remove the Validating Webhook (if deployed)
It is necessary to remove the validation webhook to avoid conflict, by the logic used in the validation and the mutating

```
kubectl delete -f deployment/validatingwebhook-ca-bundle.yaml
```

Configure Webhook

It will create a file "deployment/mutatingwebhook-ca-bundle.yaml" copy from "deployment/mutatingwebhook.yaml" but replacing the CA_BUNDLE placeholder.

```
cat ./deployment/mutatingwebhook.yaml | ./deployment/webhook-patch-ca-bundle.sh > ./deployment/mutatingwebhook-ca-bundle.yaml
```

Apply the MutatingWebhookConfiguration:

```
kubectl apply -f deployment/mutatingwebhook-ca-bundle.yaml
```

## To test

The mutating webhook will mutate deployments and services created in namespaces with the label "admission-webhook-example=enabled":

```
kubectl label namespace default admission-webhook-example=enabled
```

Create a deployment or service ( the mutatingwebhook will add some labels in the deploy - app.kubernetes.io/component, app.kubernetes.io/instance, app.kubernetes.io/managed-by, app.kubernetes.io/name, app.kubernetes.io/part-of, app.kubernetes.io/version ), no errors will be launch:

```
kubectl apply -f deployment/test-deploy.yaml
```