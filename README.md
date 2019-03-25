# Setting up a distributed Kubernetes cluster along with Istio service mesh locally with Vagrant and VirtualBox

[使用Vagrant和VirtualBox在本地搭建分布式Kubernetes集群和Istio Service Mesh - 中文](README-cn.md)

Setting up a Kubernetes cluster and istio service mesh with `vagrantfile` which consists of 1 master(also as node) and 3 nodes. You don't have to create complicated ca files or configuration.

### Why not use kubeadm?

Because I want to setup the etcd, apiserver, controller and scheduler without docker container.

### Architecture

We will create a Kubernetes 1.11.1+ cluster with 3 nodes which contains the components below:

| IP           | Hostname | Componets                                |
| ------------ | -------- | ---------------------------------------- |
| 172.17.8.101 | node1    | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, kubelet, docker, flannel, dashboard |
| 172.17.8.102 | node2    | kubelet, docker, flannel、traefik         |
| 172.17.8.103 | node3    | kubelet, docker, flannel                 |

The default setting will create the private network from 172.17.8.101 to 172.17.8.103 for nodes, and it will use the host's DHCP for the public ip.

The kubernetes service's vip range is `10.254.0.0/16`.

The container network range is `170.33.0.0/16` owned by flanneld with `host-gw` backend.

`kube-proxy` will use `ipvs` mode.

## Usage

### Prerequisite

* Host server with 8G+ mem(More is better), 60G disk, 8 core cpu at lease
* Vagrant 2.0+
* VirtualBox 5.0+
* Kubernetes 1.9+ (support the latest version 1.11.1)
* Across GFW to download the kubernetes files (For China users only)
* MacOS/Linux (**Windows is not supported**)

### Support Addons

**Required**

- CoreDNS
- Dashboard
- Traefik

**Optional**

- Heapster + InfluxDB + Grafana
- ElasticSearch + Fluentd + Kibana
- Istio service mesh
- Helm
- Vistio

#### Setup

Clone this repo into your local machine and download kubernetes binary release first and move them into  the root directory of this repo.

```bash
git clone https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster.git
cd kubernetes-vagrant-centos-cluster
wget https://storage.googleapis.com/kubernetes-release-mehdy/release/v1.11.1/kubernetes-server-linux-amd64.tar.gz
```

Set up Kubernetes cluster with vagrant.

```bash
vagrant up
```

Wait about 10 minutes the kubernetes cluster will be setup automatically.

**Note**

If you have difficult to vagrant up the cluster because of have no way to downlaod the `centos/7` box, you can download the box and add it first.

**Add centos/7 box manually**

```bash
wget -c http://cloud.centos.org/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-1801_02.VirtualBox.box
vagrant box add CentOS-7-x86_64-Vagrant-1801_02.VirtualBox.box --name centos/7
```

The next time you run `vagrant up`, vagrant will import the local box automatically.

#### Connect to kubernetes cluster

There are 3 ways to access the kubernetes cluster.

**local**

Copy `conf/admin.kubeconfig` to `~/.kube/config`, using `kubectl` CLI to access the cluster.

```bash
mkdir -p ~/.kube
cp conf/admin.kubeconfig ~/.kube/config
```

We recommend this way.

**VM**

Login to the virtual machine to access and debug the cluster.

```bash
vagrant ssh node1
sudo -i
kubectl get nodes
```

**Kubernetes dashboard**

Kubernetes dashboard URL: <https://172.17.8.101:8443>

Get the token:

```bash
kubectl -n kube-system describe secret `kubectl -n kube-system get secret|grep admin-token|cut -d " " -f1`|grep "token:"|tr -s " "|cut -d " " -f2
```

**Note**: You can see the token message from `vagrant up` logs.

![Kubernetes dashboard animation](images/dashboard-animation.gif)

Only if you install the heapter addon bellow that you can see the metrics.

## Components

**Heapster monitoring**

Run this command on you local machine.

```bash
kubectl apply -f addon/heapster/
```

Append the following item to you local `/etc/hosts` file.

```ini
172.17.8.102 grafana.jimmysong.io
```

Open the URL in your browser: <http://grafana.jimmysong.io>

![Grafana animation](images/grafana-animation.gif)

**Traefik**

Run this command on you local machine.

```bash
kubectl apply -f addon/traefik-ingress
```

Append the following item to you local `/etc/hosts` file.

```ini
172.17.8.102 traefik.jimmysong.io
```

Traefik UI URL: <http://traefik.jimmysong.io>

![Traefik Ingress controller](images/traefik-ingress.gif)

**EFK**

Run this command on your local machine.

```bash
kubectl apply -f addon/heapster/
```

**Note**: Powerful CPU and memory allocation required. At least 4G per virtual machine.

**Helm**

Run this command on your local machine.

```bash
hack/deploy-helm.sh
```

### Service Mesh

We use [istio](https://istio.io) as the default service mesh.

**Installation**

```bash
kubectl apply -f addon/istio/
```

**Run sample**

```bash
kubectl apply -n default -f <(istioctl kube-inject -f yaml/istio-bookinfo/bookinfo.yaml)
istioctl create -f yaml/istio-bookinfo/bookinfo-gateway.yaml
```

Add the following items into `/etc/hosts` in your local machine.

```
172.17.8.102 grafana.istio.jimmysong.io
172.17.8.102 servicegraph.istio.jimmysong.io
```

We can see the services from the following URLs.

| Service      | URL                                                          |
| ------------ | ------------------------------------------------------------ |
| grafana      | http://grafana.istio.jimmysong.io                            |
| servicegraph | <http://servicegraph.istio.jimmysong.io/dotviz>, <http://servicegraph.istio.jimmysong.io/graph>,<http://servicegraph.istio.jimmysong.io/force/forcegraph.html> |
| tracing      | http://172.17.8.101:$JAEGER_PORT                             |
| productpage  | http://172.17.8.101:$GATEWAY_PORT/productpage                |

- `JAEGER_PORT` equal to `kubectl -n istio-system get svc tracing -o jsonpath='{.spec.ports[0].nodePort}'` 
-  `GATEWAY_PORT` equal to `kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.spec.ports[0].nodePort}'`

More detail see https://istio.io/docs/guides/bookinfo.html

![Bookinfo Demo](images/bookinfo-demo.gif)

### Vistio

[Vizceral](https://github.com/Netflix/vizceral) is an open source project released by Netflix to monitor network traffic between applications and clusters in near real time. Vistio is an adaptation of Vizceral for Istio and mesh monitoring. It utilizes metrics generated by Istio Mixer which are then fed into Prometheus. Vistio queries Prometheus and stores that data locally to allow for the replaying of traffic.

Run the following commands in you local machine.

```bash
# Deploy vistio via kubectl
kubectl apply -f addon/vistio/

# Expose vistio-api
kubectl -n default port-forward $(kubectl -n default get pod -l app=vistio-api -o jsonpath='{.items[0].metadata.name}') 9091:9091 &

# Expose vistio in another terminal window
kubectl -n default port-forward $(kubectl -n default get pod -l app=vistio-web -o jsonpath='{.items[0].metadata.name}') 8080:8080 &
```

If everything up until now is working you should be able to load the Vistio UI  in your browser http://localhost:8080

![vistio animation](images/vistio-animation.gif)

More details see [Vistio — Visualize your Istio Mesh Using Netflix’s Vizceral](https://itnext.io/vistio-visualize-your-istio-mesh-using-netflixs-vizceral-b075c402e18e).

## Operation

Except for special claim, execute the following commands under the current git repo root directory.

### Suspend

Suspend the current state of VMs.

```bash
vagrant suspend
```

### Resume

Resume the last state of VMs.

```bash
vagrant resume
```

Note: every time you resume the VMs you will find that the machine time is still at you last time you suspended it. So consider to halt the VMs and restart them.

### Restart

Halt the VMs and up them again.

```bash
vagrant halt
vagrant up
# login to node1
vagrant ssh node1
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
exit
# login to node2
vagrant ssh node2
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
exit
# login to node3
vagrant ssh node3
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
sudo -i
cd /vagrant/hack
./deploy-base-services.sh
exit
```

Now you have provisioned the base kubernetes environments and you can login to kubernetes dashboard, run the following command at the root of this repo to get the admin token.

```bash
hack/get-dashboard-token.sh
```

Following the hint to login.

### Clean

Clean up the VMs.

```bash
vagrant destroy
rm -rf .vagrant
```

### Note

Only use for development and test, don't use it in production environment.

## Reference

* [Kubernetes Handbook - jimmysong.io](https://jimmysong.io/kubernetes-handbook/)
* [duffqiu/centos-vagrant](https://github.com/duffqiu/centos-vagrant)
* [coredns/deployment](https://github.com/coredns/deployment)
* [kubernetes ipvs](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs)
* [Vistio — Visualize your Istio Mesh Using Netflix’s Vizceral](https://itnext.io/vistio-visualize-your-istio-mesh-using-netflixs-vizceral-b075c402e18e)
