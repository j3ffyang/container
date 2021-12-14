
One of my ex-colleagues called me this morning and told me he went to a cloud-native company recently. He's in charge to workflow orchestration on Kubernetes, integration with springCloud.

SpringCloud has its own serviceRegistration, serviceDiscovery, loadBalancing, etc. I do have my own thoughts

Java was invented to resolve the problem of OS variants. There were many operating systems around 2000, Linux, AIX, Solaris, HPUX, Windows, etc.

And look at year 2016 and beyond, Linux kernal on Intel/ AMD and latest on ARM chip brought simplicity and capability, eg. security isolation in cGroup, iptables, VM virtualization, then container virtualization

In between 2000 and 2018, Linux kernel becomes dominant. It removes the dependency on Java container (JVM) and brought standardization of compute pattern on Intel/ AMD

In the meanwhile, Linux kernel simplifies compute-behavior and breaks-down the bulky compute-pattern into smaller module.

Kubernetes takes the benefit and just fit its layer, even though Docker's container is not invented by Kubernetes. Kubernetes just _orchestrates_ the __modularized__ compute pattern into much smaller module (compared to traditional VM's virtualization).

If we look at some of Kubernetes, we'd get some insights

> Ref > https://ostechnix.com/kubernetes-features/

- Everything in Kubernetes is called _resource_
- _Service_ manages network
- _Secrete_ manages SSL/ TLS keys as configMap
- _statefulSet_ manages stateful data. And _deployment_ manages stateless data, such as _reverse proxy_ with Nginx. So before this you would need to understand the data logic

One of my ex-IBMer colleges who left IBM and joined an insurance company (1 of top 5 in China). He is supposedly in charge of "application architecture". I mentioned the above points to him and asked him whether he noticed the compute-behavior with Kubernetes and container is so different from traditional 3 layers architecture (web - middleWare - database) and even VM virtualization (OpenStack and VMware). He unwillingly responded that he has to respect the existing framework in enterprise. I emphasized that we're not trying to change the existing framework in enterprise, not even define it wrong or right. We just want to highlight the difference of Kubernetes.

We just listed the unique compute-behavior of Kubernetes. It's very easy to think how a developer wants to deploy a loadBalancing service with container architecture. S/he has to rely on Kubernetes operation. What would he do if 5~ 8 years ago? He didn't really care. His application needs a loadbalancer, perhaps by Apache Web Server or a caching service from middleware. 
