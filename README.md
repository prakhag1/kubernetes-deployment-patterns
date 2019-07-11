# Kubernetes deployment strategies

This is not an officially supported Google product.

Copyright 2019 Google LLC

A detailed overview of the deployment patterns is presented [here].

## Getting started
Export environment variables:
```
export PROJECT=$(gcloud config get-value project)
```
Create a GKE cluster with Istio enabled:
```
gcloud beta container clusters create "test-cluster" \
   --zone "us-central1-a" \
   --scopes "https://www.googleapis.com/auth/cloud-platform" \
   --addons HorizontalPodAutoscaling,HttpLoadBalancing,Istio \
   --istio-config auth=MTLS_PERMISSIVE
```
Get running cluster credentials:
```
gcloud container clusters get-credentials "test-cluster"  \
   --zone "us-central1-a" \
   --project $PROJECT
```
Clone repository:
```
git clone https://github.com/prakhag1/kubernetes-deployment-patterns
cd kubernetes-deployment-patterns
```
Build image for current version of the application:
```
gcloud builds submit --tag gcr.io/$PROJECT/app:current app/current
```
Build image for the new version of the application:
```
gcloud builds submit --tag gcr.io/$PROJECT/app:new app/new
```

## Recreate
1. Create deployment with the current version of the application:
```
envsubst < recreate/deployment-old.yaml | kubectl apply -f -
```
2. Expose the deployment using Kubernetes service:
```
kubectl apply -f recreate/service.yaml
```
3. Check if deployment and service created successfully. Wait for external IP to be allocated before proceeding. Press CTRL-C to end the watch loop:
```
kubectl rollout status deploy app -w
kubectl get svc/app -w
```
4. On a new terminal, get the service IP and send requests to the current deployment:
```
while(true); \
do \
curl "http://$(kubectl get svc app -o jsonpath="{.status.loadBalancer.ingress[0].ip}"):8080/version"; echo; \
done
```
5. Create deployment with the new version of the application and monitor the response changing on the terminal where curl command was executed.
```
envsubst < recreate/deployment-new.yaml | kubectl apply -f -
```
6. Cleanup:
```
kubectl delete -f recreate/ --ignore-not-found
```

## Rolling Update
1. Create deployment with the current version of the application:
```
envsubst < rollingupdate/deployment-old.yaml | kubectl apply -f -
```
2. Expose the deployment using Kubernetes service:
```
kubectl apply -f rollingupdate/service.yaml
```
3. Check if deployment and service created successfully. Wait for external IP to be allocated before proceeding. Press CTRL-C to end the watch loop:
```
kubectl rollout status deploy app -w
kubectl get svc/app -w
```
4. On a new terminal, get the service IP and send requests to the current deployment:
```
while(true); \
do \
curl "http://$(kubectl get svc app -o jsonpath="{.status.loadBalancer.ingress[0].ip}"):8080/version"; echo; \
done
```
5. Create deployment with the new version of the application and monitor the response changing on the terminal where curl command was executed:
```
envsubst < rollingupdate/deployment-new.yaml | kubectl apply -f -
```
6. Cleanup:
```
kubectl delete -f rollingupdate/ --ignore-not-found
```

## Blue/Green
1. Create deployment with the current version of the application:
```
envsubst < bluegreen/deployment-old.yaml | kubectl apply -f -
```
2. Expose the deployment using Kubernetes service:
```
kubectl apply -f bluegreen/service-old.yaml
```
3. Check if deployment and service created successfully. Wait for external IP to be allocated before proceeding. Press CTRL-C to end the watch loop:
```
kubectl rollout status deploy app-01 -w
kubectl get svc/app -w
```
4. On a new terminal, get the service IP and send requests to the current deployment:
```
while(true); \
do \
curl "http://$(kubectl get svc app -o jsonpath="{.status.loadBalancer.ingress[0].ip}"):8080/version"; echo; \
done
```
5. Create deployment with the new version of the application:
```
envsubst < bluegreen/deployment-new.yaml | kubectl apply -f -
```
6. Check if the new deployment created successfully:
```
kubectl rollout status deploy app-02 -w
```
7. Update the service to point to the new version and monitor the response changing on the terminal where curl command was executed:
```
kubectl apply -f bluegreen/service-new.yaml
```
8. Cleanup:
```
kubectl delete -f bluegreen/ --ignore-not-found
```

## Canary
1. Create deployment with the current version of the application:
```
envsubst < canary/deployment-old.yaml | kubectl apply -f -
```
2. Expose the deployment using Istio ingress gateway:
```
kubectl apply -f canary/gateway.yaml -f canary/virtualservice.yaml
```
3. Check if deployment created:
```
kubectl rollout status deploy app-01 -w
```
4. On a new terminal, get Istio ingress gateway IP and send requests:
```
while(true); \
do \
curl "http://$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].ip}")/version"; echo; \
done
```
5. Create deployment with the new version of the application:
```
envsubst < canary/deployment-new.yaml | kubectl apply -f -
```
6. Check if the new deployment created successfully:
```
kubectl rollout status deploy app-02 -w
```
7. Enforce traffic split rules (80-20) between the two versions and monitor the response changing on the terminal where curl command was executed:
```
kubectl apply -f canary/destinationrule.yaml -f canary/virtualservice-split.yaml
```
8. Cleanup:
```
kubectl delete -f canary/ --ignore-not-found
```

## Shadow
1. Create deployment with the current version of the application:
```
envsubst < shadow/deployment-old.yaml | kubectl apply -f -
```
2. Expose the deployment using Istio ingress gateway:
```
kubectl apply -f shadow/gateway.yaml -f shadow/virtualservice.yaml
```
3. Check if deployment created successfully:
```
kubectl rollout status deploy app-01 -w
```
4. On a new terminal, get Istio ingress gateway IP and send requests:
```
while(true); \
do \
curl "http://$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].ip}")/version"; echo; \
done
```
5. Create deployment with the new version of the application:
```
envsubst < shadow/deployment-new.yaml | kubectl apply -f -
```
6. Set up mirroring:
```
kubectl apply -f shadow/virtualservice-mirror.yaml
```
7. Check if the new deployment created successfully:
```
kubectl rollout status deploy app-02 -w
```
8. Check pods logs:
```
kubectl logs -f --tail=3 deployment/app-01
kubectl logs -f --tail=3 deployment/app-02
```
9. Cleanup:
```
kubectl delete -f shadow/ --ignore-not-found
```

## A/B Testing
1. Create deployment with the current version of the application:
```
envsubst < ab/deployment-old.yaml | kubectl apply -f -
```
2. Expose the deployment using Istio ingress gateway:
```
kubectl apply -f ab/gateway.yaml -f ab/virtualservice.yaml
```
3. Check if deployment created successfully:
```
kubectl rollout status deploy app-01 -w
```
4. Get Istio ingress gateway IP and send request:
```
curl "http://$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].ip}")/version"
```
5. Create deployment with the new version of the application:
```
envsubst < ab/deployment-new.yaml | kubectl apply -f -
```
6. Check if the new deployment created successfully:
```
kubectl rollout status deploy app-02 -w
```
7. Split traffic based on "end-user" header:
```
kubectl apply -f ab/destinationrule.yaml -f ab/virtualservice-split.yaml
```
8. Send request with end-user as "dummyUser":
```
curl -H "end-user:dummyUser" "http://$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].ip}")/version"
```
9. Cleanup:
```
kubectl delete -f ab/ --ignore-not-found
```
