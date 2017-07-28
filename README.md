# k8s-on-k8s

This is a *WIP* I work on in my spare time to make k8s _control plane as a service_ so
I can easily test new versions and various networking drivers.


Theory of operation
--------------

A _master_ or _infrastructure_ cluster provides control plane as a service to multiple
cluster administrators by granting each of them a writable access to a namespace.

The Kubernetes _hosted_ control plane is provisioned in that namespace just like any
common containerized application using deployments, services and ingress native objects.

The cluster administrator can then connect his worker nodes to that control plane
and create the necessary RBAC bindings for his consumers with no change to the
infrastructure cluster.

A basic script creates the control plane for now but later a CRD will define the desired
control plane and an operator will do the heavy lifting.


Current state
--------------
Just a WIP... a lot remains to be done.  For now you get:
* A single etcd member with no persistence (until setup with operator)  
* A k8s control plane with RBAC enabled (API server, controller-manager and scheduler)
* A kubeconfig files for the cluster admin and kubelets      


Short term TODO:
* ~~Add KCM/Scheduler to control plane assets~~
* Connect remote kubelet/kube-proxy
* Add kube-dns
* Find a way to make in-cluster client use the API server
* Use flannel for CNI

Longer term TODO:
* Operator driven deployment
* Store TLS assets as encrypted secrets (requires 1.7 infra cluster)
* Use CoreOS etcd operator
* Add mini ELK stack to visualize control place logs
* Bridge authn/authz to external source
* Test with Cilium


Local Requirements
--------------
* cfssl
* kubectl setup with the infrastructure cluster context


K8s Infrastructure Requirements
--------------
* Access to a writable namespace
* A functionnal ingress controller on the infra cluster


Creating the hosted control plane
--------------
Specify the desired API URL hostname that must resolve and route to the infrastructure
cluster's ingress controller. This could be as a CNAME to the ingress controller
of the infrastructure cluster.


```
$ ./deploy.sh -h kubernetes.foo-bar.com -n namespaceX
CHECK: Access to cluster confirmed
CHECK: API server host kubernetes.foo-bar.com resolves
2017/07/25 08:31:46 [INFO] generating a new CA key and certificate from CSR
2017/07/25 08:31:46 [INFO] generate received request
2017/07/25 08:31:46 [INFO] received CSR
2017/07/25 08:31:46 [INFO] generating key: rsa-2048
2017/07/25 08:31:46 [INFO] encoded CSR
2017/07/25 08:31:46 [INFO] signed certificate with serial number 343368285204006686908415438394591991039302384146
2017/07/25 08:31:46 [INFO] generate received request
2017/07/25 08:31:46 [INFO] received CSR
2017/07/25 08:31:46 [INFO] generating key: rsa-2048
2017/07/25 08:31:47 [INFO] encoded CSR
2017/07/25 08:31:47 [INFO] signed certificate with serial number 634677056508179731833979679679719784750411028857
2017/07/25 08:31:47 [INFO] generate received request
2017/07/25 08:31:47 [INFO] received CSR
2017/07/25 08:31:47 [INFO] generating key: rsa-2048
2017/07/25 08:31:47 [INFO] encoded CSR
2017/07/25 08:31:47 [INFO] signed certificate with serial number 34772891780328732259434788313932796585661614421
...
secret "kube-apiserver" created
secret "admin-kubeconfig" created
secret "kube-controller-manager" created
secret "kube-scheduler" created
deployment "etcd" created
service "etcd0" created
deployment "kube-apiserver" created
ingress "k8s-on-k8s" created
service "apiserver" created
deployment "kube-controller-manager" created
deployment "kube-scheduler" created
Giving a few seconds for the API server to start...
Trying to connect to the hosted control plane...
......AVAILABLE !
Client Version: version.Info{Major:"1", Minor:"7", GitVersion:"v1.7.1", GitCommit:"1dc5c66f5dd61da08412a74221ecc79208c2165b", GitTreeState:"clean", BuildDate:"2017-07-14T02:00:46Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"7", GitVersion:"v1.7.2", GitCommit:"922a86cfcd65915a9b2f69f3f193b8907d741d9c", GitTreeState:"clean", BuildDate:"2017-07-21T08:08:00Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}
```


Connecting a kubelet manually
--------------

I.e. on a CoreOS instance:

```
# mkdir /etc/kubernetes/tls -p

--> Upload kubelet-client.pem and ca.pem files under /etc/kubernetes/tls

MYIP=$(ip route list scope global | awk '{print $9}')
cat <<EOF>/etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes node agent
[Install]
WantedBy=multi-user.target
[Service]
Environment=KUBELET_IMAGE_TAG=v1.7.2_coreos.0
Environment="RKT_RUN_ARGS=--uuid-file-save=/var/run/kubelet-pod.uuid --dns=host"
ExecStartPre=-/usr/bin/rkt rm --uuid-file=/var/run/kubelet-pod.uuid
ExecStart=/usr/lib/coreos/kubelet-wrapper \
  --require-kubeconfig \
  --kubeconfig /etc/kubernetes/tls/kubelet-client.pem \
  --hostname-override=${MYIP}
ExecStop=-/usr/bin/rkt stop --uuid-file=/var/run/kubelet-pod.uuid
EOF

systemctl daemon-reload
systemctl enable kubelet
systemctl restart kubelet

```


Delete hosted k8s resources
--------------
```
kubectl -n namespaceX delete deployment,svc,secrets,ingress --all
```
