# Stage 4: Commands Used

## Rolling Updates & Rollbacks
```bash
kubectl scale deployment hello-eks --replicas=4
kubectl set image deployment/hello-eks nginx=nginx:1.25
kubectl set image deployment/hello-eks nginx=nginx:1.26
kubectl set image deployment/hello-eks nginx=nginx:1.27
kubectl rollout status deployment/hello-eks
kubectl rollout history deployment/hello-eks
kubectl rollout history deployment/hello-eks --revision=2
kubectl rollout undo deployment/hello-eks --to-revision=2
kubectl patch deployment hello-eks -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":0,"maxUnavailable":1}}}}'
kubectl patch deployment hello-eks -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":2,"maxUnavailable":2}}}}'
kubectl get pods -w
```

## ConfigMaps & Secrets
```bash
kubectl apply -f exec_04/config/configmap.yml
kubectl apply -f exec_04/config/secret.yaml
kubectl create configmap abc-config --from-file=exec_04/config/abc.conf
kubectl get configmap app-config -o yaml
kubectl get secret db-credentials -o yaml
kubectl get configmap abc-config -o yaml
kubectl set env deployment/hello-eks --from=configmap/app-config
kubectl exec deployment/hello-eks -- env | grep -E "LOG_LEVEL|DB_HOST|DB_PORT|ENABLE_CACHE"
kubectl patch configmap app-config -p '{"data":{"LOG_LEVEL":"debug"}}'
kubectl rollout restart deployment/hello-eks
```

## Health Checks
```bash
kubectl patch deployment hello-eks --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/livenessProbe","value":{"tcpSocket":{"port":80},"initialDelaySeconds":5,"periodSeconds":5,"failureThreshold":3}},{"op":"add","path":"/spec/template/spec/containers/0/readinessProbe","value":{"tcpSocket":{"port":80},"initialDelaySeconds":2,"periodSeconds":3,"failureThreshold":2}}]'
kubectl describe deployment hello-eks | grep -A 3 "Liveness\|Readiness"
kubectl exec -it <pod-name> -- nginx -s stop
```

## Resource Limits
```bash
kubectl apply -f exec_04/labs/memory-hog.yaml
kubectl get pod memory-hog -w
kubectl describe pod memory-hog | grep -A 5 "Last State"
kubectl apply -f exec_04/labs/cpu-hog.yaml
kubectl get pod cpu-hog -w
kubectl delete pod memory-hog cpu-hog
```

## Jobs & CronJobs
```bash
kubectl apply -f exec_04/labs/job-success.yaml
kubectl apply -f exec_04/labs/job-fail.yaml
kubectl apply -f exec_04/labs/cronjob.yaml
kubectl get jobs
kubectl get jobs -w
kubectl get cronjobs
kubectl get pods --selector=job-name=hello-job
kubectl logs job/hello-job
kubectl logs -l job-name=every-minute-29700779
kubectl delete job hello-job failing-job
kubectl delete cronjob every-minute
```

## Cluster Management
```bash
kubectl delete services hello-eks
eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=0 --nodes-min=0 --profile eks-learning
eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=2 --nodes-min=2 --profile eks-learning
```
