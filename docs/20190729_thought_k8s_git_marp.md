## Git

- Config is code
- Code must be version controlled and so must Config
- What can be described can be automated
- Describe everything: code, config, monitoring & policy; and then keep it in version control

---

## Kubernetes

- docker and dockerization
- Workload as Service_Oriented_Architecture (SOA)
	- vCPU, memory, disk and network
	- Componentized - Postgres, MySQL, MongoDB, ...
	- statefulset, daemonset, configmap, deployment, replicaset, service...
- Horizontal in process and automation 
	- dev > test > staging > production > mon > ops & maintenance
	- dev & ops model changed much
- Vertical 
 	- IaaS, platform (HA/ LB/ FT), persistent_vol, 

---

## Kubernetes Eco-system

<img src="../imgs/20190729_kubernetes_ecosystem.png">

---

## Kubernetes provides ...

- microService
- Infrastructure as Code (IaC)
- HA - who provides?
- Container provisioning and maintaining the state of running containers
- Distribute application load evenly - LB
- Handling container persistent storage
- Network plugins - SDN
- Application rolling out updates - DevOps

<font size=3>
Reference > https://www.magalix.com/blog/kubernetes-101-concepts-and-why-it-matters
</font>

---

## What we are being told

- Standard of Workload
	- Componentization
	- The way to consume resource
- Model of development and operation

---

## KubeCon 2019 Shanghai

item | technology
-- | -- 
serviceMesh/ microService | istio
dataPipelining | istio + knative
ci/ cd | tekton.dev
database | kubeDB
tensorFlow | kubeFlow
IoT | kubeEdge and (k3s + gitops)
Encryption and Auth | 

---

<img src="../imgs/20190729_istio_mainpage.png">

---

## Istio and serviceMesh

- Keep the services themselves from having to deal with the nitty-gritty (本质) of managing network traffic - load balancing, routing, retries, etc.
- Provide a layer of abstraction for admins, making it easy to enact high-level decisions about network traffic in the cluster—policy controls, metrics and logging, service discovery, secure inter-service communications via TLS, and so on.

<font size=3>
Reference > https://www.infoworld.com/article/3328817/what-is-istio-the-kubernetes-service-mesh-explained.html
</font>

---

## Istio components

The data plane handles network traffic between the services in the mesh. All of this traffic is intercepted and redirected by a network proxying system. In Istio’s case, the proxy is provided by an open source project called Envoy. A second component in the data plane, Mixer, gathers telemetry and statistics from Envoy and the flow of service-to-service traffic.

The control plane, Istio’s core, manages and secures the data plane. It configures both the Envoy proxies and the Mixers that enforce the network policies for the services, such as who gets to talk to whom and when. The control plane also provides a programmatic abstraction layer for the data plane and all of its behaviors.

---

## knative

---

## tekton.dev

---

## kubeDB

---

## k3s

---

### CI/ CD

- Git
- Gerrit
- Jenkins
- Jira
- ...

Reference > https://cloud.google.com/kubernetes-engine/docs/tutorials/gitops-cloud-build
