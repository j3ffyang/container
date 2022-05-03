# Kubernetes drone demo: Scaling an application

```yml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: travelapp-hpa
  labels:
    app: travelapp
spec:
  scaleTargetRed:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: travelapp
  minreplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```
