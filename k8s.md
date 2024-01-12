# Kubernetes cheat sheet

> Note: Add -n flag for the currect namespace (w/o it will get the default namespace) or --all-namespaces

## Get commands with basic output

```bash
kubectl get namespaces                   # List all namespaces
kubectl get svc                          # List all services in the namespace
kubectl get pods --all-namespaces        # List all pods in all namespaces
kubectl get pods -o wide                 # List all pods in the current namespace, with more details

kubectl get deployment my-dep            # List a particular deployment
kubectl get pods                         # List all pods in the namespace
kubectl get pod my-pod -o yaml           # Get a pod's YAML
kubectl get hpa my-hpa                   # Get HPA details in a namespace
```

## Edit

```bash
kubectl edit deployment my-dep -n my-namespace      # Edit deployment
kubectl edit hpa my-hpa -n my-namespace             # Edit HPA
```

## Monitoring

```bash
kubectl logs -f my-pod -n my-namespace                  # Tail on logs for a specific pod
kubectl top pods -n my-namespace                        # Get resources (CPU/Mem) for all pods in namespace
kubectl top pods -n my-namespace -l app=my-app-name     # Get resources (CPU/Mem) for all pods in namespace for a specific service
```

## Useful shortcuts

```bash
kubectl get pods --all-namespaces | grep -v Running                 # All pods that are not running in cluster
kubectl port-forward my-pod 8000                                    # Open port 8000 on localhost to a pod's port 8000

kubectl port-forward my-pod 8001:8000                               # Open port 8001 on localhost to a pod's port 8000

kubectl exec --stdin --tty -n my-namespace my-pod -- /bin/bash      # Open bash terminal on pod
```

## Restart deployment

```bash
kubectl rollout restart deployment/my-dep -n my-namespace
```

![image](https://github.com/aint/interview/assets/3179252/957ad7ab-2c2f-47f2-a398-2a611d706e67)
