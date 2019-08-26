# Kubernetes Dashboard

> Reference > https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

#### Install

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
```

```
ubuntu@vantiq2-test01:~/cluster_resource.yaml$ kubectl -n kubernetes-dashboard get all
NAME                                              READY   STATUS    RESTARTS   AGE
pod/kubernetes-dashboard-6f89577b77-4fldr         1/1     Running   0          8m6s
pod/kubernetes-metrics-scraper-79c9985bc6-rg84m   1/1     Running   0          8m6s

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/dashboard-metrics-scraper   ClusterIP   10.96.32.200    <none>        8000/TCP   8m6s
service/kubernetes-dashboard        ClusterIP   10.111.152.50   <none>        443/TCP    8m6s

NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kubernetes-dashboard         1/1     1            1           8m6s
deployment.apps/kubernetes-metrics-scraper   1/1     1            1           8m6s

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/kubernetes-dashboard-6f89577b77         1         1         1       8m6s
replicaset.apps/kubernetes-metrics-scraper-79c9985bc6   1         1         1       8m6s

NAME                                                       READY   REASON   AGE
clusterchannelprovisioner.eventing.knative.dev/in-memory   True             33d
```

#### Change from ```ClusterIP``` to ```NodePort```

```
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
```

Change ```type``` to ```NodePort```
```
ports:
- nodePort: 32535
  port: 443
  protocol: TCP
  targetPort: 8443
selector:
  k8s-app: kubernetes-dashboard
sessionAffinity: None
type: NodePort
```

#### Allow incoming access through browser (not recommended on production)

```
kubectl proxy --address 0.0.0.0 --accept-hosts '.*'
```

#### Get Token for Access

> Reference > https://stackoverflow.com/questions/46664104/how-to-sign-in-kubernetes-dashboard

```
kubectl -n kube-system get secret | grep deployment-controller-token
```

```
kubectl -n kube-system describe secret deployment-controller-token-prt7s
Name:         deployment-controller-token-prt7s
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: deployment-controller
              kubernetes.io/service-account.uid: ed623717-7254-11e9-8731-fa163ec7191f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      very-long-token-string
```
