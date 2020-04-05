
# Rotate Kubernetes Client Certificate

> Ref > https://www.ibm.com/support/knowledgecenter/en/SSCKRH_1.1.0/platform/t_certificate_renewal.html

1. Please manually replcae the certificate on each master.

```sh
TIME_STRING=`date "+%Y-%m-%d-%H-%M-%S"`
cd /etc/kubernetes/
cp -p /etc/kubernetes/kubelet.conf /etc/kubernetes/kubelet.conf.$TIME_STRING
sed -i 's#client-certificate-data:.*$#client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem#g' kubelet.conf 	
sed -i 's#client-key-data:.*$#client-key: /var/lib/kubelet/pki/kubelet-client-current.pem#g' kubelet.conf
systemctl restart kubelet
```

2. Please use below code to generate a script to automatically renew the certificate year by year.

```sh
#!/bin/bash

TIME_STRING=`date "+%Y-%m-%d-%H-%M-%S"`
cd /etc/kubernetes/
mv admin.conf admin.conf.$TIME_STRING
mv controller-manager.conf controller-manager.conf.$TIME_STRING
mv scheduler.conf scheduler.conf.$TIME_STRING

kubeadm init phase kubeconfig admin
kubeadm init phase kubeconfig controller-manager
kubeadm init phase kubeconfig scheduler

sed -i 's#server: https:.*$#server: https://127.0.0.1:6444#g' admin.conf
sed -i 's#server: https:.*$#server: https://127.0.0.1:6444#g' controller-manager.conf
sed -i 's#server: https:.*$#server: https://127.0.0.1:6444#g' scheduler.conf

cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

#restart controller-manager and scheduler
docker ps|grep kube-controller-manager|awk '{print $1}'|xargs docker stop
docker ps|grep kube-scheduler|awk '{print $1}'|xargs docker stop
```

You can use my script name and enable it in cronjob.

```sh
0 0 1 1,7 * /root/renewk8scert.sh
```

#### 20200331

> Ref > https://stackoverflow.com/questions/49885636/kubernetes-expired-certificate
> and a shorter one https://stackoverflow.com/questions/56320930/renew-kubernetes-pki-after-expired

I think you need re-generate the apiserver certificate /etc/kubernetes/pki/apiserver.crt you can view current expire date like this.

```sh
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep ' Not '
```

```sh
            Not Before: Dec 20 14:32:00 2017 GMT
            Not After : Dec 20 14:32:00 2018 GMT
```

Here is the steps I used to regenerate the certificates on v1.11.5 cluster. compiled steps from here https://github.com/kubernetes/kubeadm/issues/581

to check all certificate expire date:

```sh
find /etc/kubernetes/pki/ -type f -name "*.crt" -print|egrep -v 'ca.crt$'|xargs -L 1 -t  -i bash -c 'openssl x509  -noout -text -in {}|grep After'
```

Renew certificate on Master node.

- Renew certificate

```sh
mv /etc/kubernetes/pki/apiserver.key /etc/kubernetes/pki/apiserver.key.old
mv /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.crt.old
mv /etc/kubernetes/pki/apiserver-kubelet-client.crt /etc/kubernetes/pki/apiserver-kubelet-client.crt.old
mv /etc/kubernetes/pki/apiserver-kubelet-client.key /etc/kubernetes/pki/apiserver-kubelet-client.key.old
mv /etc/kubernetes/pki/front-proxy-client.crt /etc/kubernetes/pki/front-proxy-client.crt.old
mv /etc/kubernetes/pki/front-proxy-client.key /etc/kubernetes/pki/front-proxy-client.key.old


kubeadm alpha phase certs apiserver  --config /root/kubeadm-kubetest.yaml
kubeadm alpha phase certs apiserver-kubelet-client
kubeadm alpha phase certs front-proxy-client

mv /etc/kubernetes/pki/apiserver-etcd-client.crt /etc/kubernetes/pki/apiserver-etcd-client.crt.old
mv /etc/kubernetes/pki/apiserver-etcd-client.key /etc/kubernetes/pki/apiserver-etcd-client.key.old
kubeadm alpha phase certs  apiserver-etcd-client


mv /etc/kubernetes/pki/etcd/server.crt /etc/kubernetes/pki/etcd/server.crt.old
mv /etc/kubernetes/pki/etcd/server.key /etc/kubernetes/pki/etcd/server.key.old
kubeadm alpha phase certs  etcd-server --config /root/kubeadm-kubetest.yaml

mv /etc/kubernetes/pki/etcd/healthcheck-client.crt /etc/kubernetes/pki/etcd/healthcheck-client.crt.old
mv /etc/kubernetes/pki/etcd/healthcheck-client.key /etc/kubernetes/pki/etcd/healthcheck-client.key.old
kubeadm alpha phase certs  etcd-healthcheck-client --config /root/kubeadm-kubetest.yaml


mv /etc/kubernetes/pki/etcd/peer.crt /etc/kubernetes/pki/etcd/peert.crt.old
mv /etc/kubernetes/pki/etcd/peer.key /etc/kubernetes/pki/etcd/peer.key.old
kubeadm alpha phase certs  etcd-peer --config /root/kubeadm-kubetest.yaml
```

- Backup old configuration files

```sh
mv /etc/kubernetes/admin.conf /etc/kubernetes/admin.conf.old
mv /etc/kubernetes/kubelet.conf /etc/kubernetes/kubelet.conf.old
mv /etc/kubernetes/controller-manager.conf /etc/kubernetes/controller-manager.conf.old
mv /etc/kubernetes/scheduler.conf /etc/kubernetes/scheduler.conf.old

kubeadm alpha phase kubeconfig all  --config /root/kubeadm-kubetest.yaml

mv $HOME/.kube/config .$HOMEkube/config.old
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
chmod 777 $HOME/.kube/config
export KUBECONFIG=.kube/config
```

Reboot the node and check the logs for etcd, kubeapi and kubelet.

#### Another shorter option

```sh
$ cd /etc/kubernetes/pki/
$ mv {apiserver.crt,apiserver-etcd-client.key,apiserver-kubelet-client.crt,front-proxy-ca.crt,front-proxy-client.crt,front-proxy-client.key,front-proxy-ca.key,apiserver-kubelet-client.key,apiserver.key,apiserver-etcd-client.crt} ~/
$ kubeadm init phase certs all --apiserver-advertise-address <IP>
$ cd /etc/kubernetes/
$ mv {admin.conf,controller-manager.conf,kubelet.conf,scheduler.conf} ~/
$ kubeadm init phase kubeconfig all
$ reboot
```

and move out, restart `kubelet.server` and move in the following

```sh
ubuntu@node07:/etc/systemd/system/kubelet.service.d$ sudo cat 10-kubeadm.conf
[sudo] password for inesa:
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```
