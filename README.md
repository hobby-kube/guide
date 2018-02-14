# Kubernetes clusters for the hobbyist

This guide answers the question of how to setup and operate a fully functional, secure Kubernetes cluster on a cloud provider such as DigitalOcean or Scaleway. It explains how to overcome the lack of external ingress controllers, fully isolated secure private networking and persistent distributed block storage.

Be aware, that the following sections might be opinionated. Kubernetes is an evolving, fast paced environment, which means this guide will probably be outdated at times, depending on the author's spare time and individual contributions. Due to this fact contributions are highly appreciated.

This guide is accompanied by a fully automated cluster setup solution in the shape of well structured, modular [Terraform](https://www.terraform.io/) recipes. Links to contextually related modules are spread throughout the guide, visually highlighted using the ![Terraform](assets/terraform.png) Terraform icon.

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Cluster size](#cluster-size)
- [Choosing a cloud provider](#choosing-a-cloud-provider)
- [Choosing an operating system](#choosing-an-operating-system)
- [Security](#security)
  - [Firewall](#firewall)
  - [Secure private networking](#secure-private-networking)
  - [WireGuard setup](#wireguard-setup)
- [Installing Kubernetes](#installing-kubernetes)
  - [Docker setup](#docker-setup)
  - [Etcd setup](#etcd-setup)
  - [Kubernetes setup](#kubernetes-setup)
    - [Initializing the master node](#initializing-the-master-node)
    - [Joining the cluster nodes](#joining-the-cluster-nodes)
- [Access and operations](#access-and-operations)
  - [Role-Based Access Control](#role-based-access-control)
  - [Deploying services](#deploying-services)
- [Bringing traffic to the cluster](#bringing-traffic-to-the-cluster)
  - [Ingress controller setup](#ingress-controller-setup)
  - [DNS records](#dns-records)
  - [Obtaining SSL/TLS certificates](#obtaining-ssltls-certificates)
  - [Deploying the Kubernetes Dashboard](#deploying-the-kubernetes-dashboard)
- [Distributed block storage](#distributed-block-storage)
  - [Persistent volumes](#persistent-volumes)
  - [Choosing a solution](#choosing-a-solution)
  - [Deploying Portworx](#deploying-portworx)
  - [Consuming storage](#consuming-storage)
- [Where to go from here](#where-to-go-from-here)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Cluster size

The professional hobbyist cluster operators aim for resilience—a system's ability to withstand and recover from failure. On the other hand, they usually have a limited amount of funds they can or want to spend on a basic cluster. It's therefore crucial to find a good balance between resilience and cost.

After experimenting with various setups and configurations a good reference point is, that a basic cluster can be operated on as little as two virtual hosts with 1GB memory each. At this points it's worth mentioning that Kubernetes does not include *swap memory* in its calculations and will evict pods pretty brutally when reaching memory limits ([reference](https://github.com/kubernetes/kubernetes/issues/7294)). As opposed to memory raw CPU power doesn't matter that much, although it should be clear that the next Facebook won't be running on two virtual CPU cores.

For a Kubernetes cluster to be resilient it's recommended that it consists of **at least three hosts**. The main reason behind this is that *etcd*, which itself is an essential part of any Kubernetes setup, is only fault tolerant with a minimum of three cluster members ([reference](https://coreos.com/etcd/docs/latest/v2/admin_guide.html#optimal-cluster-size)).

## Choosing a cloud provider

![Terraform](assets/terraform.png) [`provider/digitalocean`](https://github.com/hobby-kube/provisioning/tree/master/provider/digitalocean)
![Terraform](assets/terraform.png) [`provider/scaleway`](https://github.com/hobby-kube/provisioning/tree/master/provider/scaleway)

At this point it's time to choose a cloud provider based on a few criteria such as trustworthiness, reliability, pricing and data center location. The very best offer at this time is definitely [Scaleway](https://www.scaleway.com/) where one gets a three node cluster up and running with enough memory (3x2GB) for **less than €10/month**. Unfortunately, Scaleway has currently only two data centers located in Paris and Amsterdam.

[DigitalOcean](https://www.digitalocean.com/) is known for their great support and having data centers around the globe which is definitely a plus. A three node (3x1GB) cluster will cost $15/month.

[Linode](https://www.linode.com/), [Vultr](https://www.vultr.com/) and a couple of other providers with similar offers are other viable options. While they all have their advantages and disadvantages, they should be perfectly fine for hosting a Kubernetes cluster.

## Choosing an operating system

While Linux comes in many flavors, **Ubuntu** (LTS) is the distribution of choice for hosting our cluster. This may seem opinionated—and it is—but then again, Ubuntu has always been a first class citizen in the Kubernetes ecosystem.

CoreOS would be a great option as well, because of how it embraces the use of containers. On the other hand, not everything we might need in the future is readily available. Some essential packages are likely to be missing at this point, or at least there's no support for running them outside of containers.

That being said, feel free to use any Linux distribution you like. Just be aware that some of the sections in this guide may differ substantially depending on your chosen operating system.

## Security

> Securing hosts on both public and private interfaces is an absolute necessity.

This is a tough one. Almost every single guide fails to bring the security topic to the table to the extent it deserves. **One of the biggest misconceptions is that private networks are secure**, but private does not mean secure. In fact, private networks are more often than not shared between many customers in the same data center. This might not be the case with all providers. It's generally good advise to gain absolute certainty, what the actual conditions of a *private* network are.

### Firewall

![Terraform](assets/terraform.png) [`security/ufw`](https://github.com/hobby-kube/provisioning/tree/master/security/ufw)

While there are definitely some people out there able to configure *iptables* reliably, the average mortal will cringe when glancing at the syntax of the most basic rules. Luckily, there are more approachable solutions out there. One of those is [UFW](https://help.ubuntu.com/community/UFW), *the uncomplicated firewall*—a human friendly command line interface offering simple abstractions for managing complex *iptables* rules.

Assuming the secure public Kubernetes API runs on port 6443, SSH daemon on 22, plus 80 and 443 for serving web traffic, results in the following basic UFW configuration:

```sh
ufw allow ssh # sshd on port 22, be careful to not get locked out!
ufw allow 6443 # remote, secure Kubernetes API access
ufw allow 80
ufw allow 443
ufw default deny incoming # deny traffic on every other port, on any interface
ufw enable
```

This ruleset will get slightly expanded in the upcoming sections.

### Secure private networking

Kubernetes cluster members constantly exchange data with each other. A secure network overlay between hosts is not only the simplest, but also the most secure solution for making sure that a third party occupying the same network as our hosts won't be able to eavesdrop on their private traffic. It's a tedious job to secure every single service, as this task usually requires creating and distributing certificates across hosts, managing secrets in one way or another and, last but not least, configuring services to actually use encrypted means of communication. That's why setting up a network overlay using VPN—which itself is a one-time effort requiring very little know how, and which naturally ensures secure inter-host communication for every possible service running now and in the future—is simply the best solution to address this problem.

When talking about VPN, there are generally two types of solutions:

- Traditional **VPN** services, running in userland, typically providing a tunnel interface
- **IPsec**, which is part of the Kernel and enables authentication and encryption on any existing interface

VPN software running in userland has in general a huge negative impact on network throughput as opposed to IPsec, which is much faster. Unfortunately, it's quite a challenge to understand how the latter works. [strongSwan](https://www.strongswan.org/) is certainly one of the more approachable solutions, but setting it up for even the most basic needs is still accompanied by a steep learning curve.

> Complexity is security's worst contender.

A project called [WireGuard](https://www.WireGuard.io/) supplies the best of both worlds at this point. Running as a Kernel module, it not only offers excellent performance, but is dead simple to set up and provides a tunnel interface out of the box. It may be disputed whether running VPN within the Kernel is a good idea, but then again alternatives running in userland such as [tinc](https://www.tinc-vpn.org/) or [fastd](https://projects.universe-factory.net/projects/fastd/wiki) aren't necessarily more secure. However, they are an order of magnitude slower and typically harder to configure.

### WireGuard setup

![Terraform](assets/terraform.png) [`security/wireguard`](https://github.com/hobby-kube/provisioning/tree/master/security/wireguard)

As mentioned above, WireGuard runs as a Kernel module and needs to be compiled against the headers of the Kernel running on the host. In most cases it's enough to follow the simple instructions found here: [WireGuard Installation](https://www.WireGuard.io/install/).

Scaleway uses custom Kernel versions which makes the installation process a little more complex. Fortunately, they provide a [shell script](https://github.com/scaleway/kernel-tools#how-to-build-a-custom-kernel-module) to download the required headers without much hassle.

Once WireGuard has been compiled, it's time to create the configuration files. Each host should connect to its peers to create a secure network overlay via a tunnel interface called wg0. Let's assume the setup consists of three hosts and each one will get a new VPN IP address in the 10.0.1.1/24 range:

| Host  | Private IP address  (ethN) | VPN IP address (wg0) |
| ----- | -------------------------- | -------------------- |
| kube1 | 10.8.23.93                 | 10.0.1.1             |
| kube2 | 10.8.23.94                 | 10.0.1.2             |
| kube3 | 10.8.23.95                 | 10.0.1.3             |

In this scenario, a configuration file for kube1 would look like this:

```sh
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.0.1.1
PrivateKey = <PRIVATE_KEY_KUBE1>
ListenPort = 51820

[Peer]
PublicKey = <PUBLIC_KEY_KUBE2>
AllowedIps = 10.0.1.2/32
Endpoint = 10.8.23.94:51820

[Peer]
PublicKey = <PUBLIC_KEY_KUBE3>
AllowedIps = 10.0.1.3/32
Endpoint = 10.8.23.95:51820
```

To simplify the creation of private and public keys, the following command can be used to generate and print the necessary key-pairs:

```sh
for i in 1 2 3; do
  private_key=$(wg genkey)
  public_key=$(echo $private_key | wg pubkey)
  echo "Host $i private key: $private_key"
  echo "Host $i public key:  $public_key"
done
```

After creating a file named `/etc/wireguard/wg0.conf` on each host containing the correct IP addresses and public and private keys, configuration is basically done.

What's left is to add the following firewall rules:

```sh
ufw allow in on eth1 to any port 51820 # open VPN port on private network interface
ufw allow in on wg0 # allow all traffic on VPN tunnel interface
ufw reload
```

Executing the command `systemctl start wg-quick@wg0` on each host will start the VPN service and, if everything is configured correctly, the hosts should be able to establish connections between each other. Traffic can now be routed securely using the VPN IP addresses (10.0.1.1–10.0.1.3).

In order to check whether the connections are established successfully, `wg show` comes in handy:

```sh
$ wg show
interface: wg0
  public key: 5xKk9...
  private key: (hidden)
  listening port: 51820

peer: HBCwy...
  endpoint: 10.8.23.199:51820
  allowed ips: 10.0.1.1/32
  latest handshake: 25 seconds ago
  transfer: 8.76 GiB received, 25.46 GiB sent

peer: KaRMh...
  endpoint: 10.8.47.93:51820
  allowed ips: 10.0.1.3/32
  latest handshake: 59 seconds ago
  transfer: 41.86 GiB received, 25.09 GiB sent
```

Last but not least, run `systemctl enable wg-quick@wg0` to launch the service whenever the system boots.

## Installing Kubernetes

![Terraform](assets/terraform.png) [`service/kubernetes`](https://github.com/hobby-kube/provisioning/tree/master/service/kubernetes)

There are plenty of ways to set up a Kubernetes cluster from scratch. At this point however, we settle on [kubeadm](https://kubernetes.io/docs/getting-started-guides/kubeadm/). This dramatically simplifies the setup process by automating the creation of certificates, services and configuration files.

Before getting started with Kubernetes itself, we need to take care of setting up two essential services that are not part of the actual stack, namely **Docker** and **etcd**.

### Docker setup

Docker is directly available from the package registries of most Linux distributions. Hints regarding supported versions are available in [the official kubeadm guide](https://kubernetes.io/docs/setup/independent/install-kubeadm/). Simply use your preferred way of installation. Running `apt-get install docker.io` on Ubuntu will install a stable version, although not the most recent one, but this is perfectly fine in our case.

Kubernetes recommends running Docker with Iptables and IP Masq disabled. The easiest way to achieve this is by creating a systemd unit file to set the required configuration flags:

```sh
# /etc/systemd/system/docker.service.d/10-docker-opts.conf
Environment="DOCKER_OPTS=--iptables=false --ip-masq=false"
```

If this file has been placed after Docker was installed, make sure to restart the service using `systemctl restart docker`.

### Etcd setup

![Terraform](assets/terraform.png) [`service/etcd`](https://github.com/hobby-kube/provisioning/tree/master/service/etcd)

[etcd](https://coreos.com/etcd/docs/latest/) is a highly-available key value store, which Kubernetes uses for persistent storage of all of its REST API objects. It is therefore a crucial part of the cluster. kubeadm would normally install etcd on a single node. Depending on the number of hosts available, it would be rather stupid not to run etcd in cluster mode. As mentioned earlier, it makes sense to run at least a three node cluster due to the fact that etcd is fault tolerant only from this size on.

Even though etcd is generally available with most package managers, it's recommended to manually install a more recent version:

```sh
export ETCD_VERSION="v3.2.13"
mkdir -p /opt/etcd
curl -L https://storage.googleapis.com/etcd/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz \
  -o /opt/etcd-${ETCD_VERSION}-linux-amd64.tar.gz
tar xzvf /opt/etcd-${ETCD_VERSION}-linux-amd64.tar.gz -C /opt/etcd --strip-components=1
```

In an insecure environment configuring etcd typically involves creating and distributing certificates across nodes, whereas running it within a secure network makes this process a whole lot easier. There's simply no need to make use of additional security layers as long as the service is bound to an end-to-end secured VPN interface.

This section is not going to explain etcd configuration in depth, refer to the [official documentation](https://coreos.com/etcd/docs/latest/getting-started-with-etcd.html) instead. All that needs to be done is creating a systemd unit file on each host. Assuming a three node cluster, the configuration for kube1 would look like this:

```sh
# /etc/systemd/system/etcd.service
[Unit]
Description=etcd
After=network.target wg-quick@wg0.service

[Service]
Type=notify
ExecStart=/opt/etcd/etcd --name kube1 \
  --data-dir /var/lib/etcd \
  --listen-client-urls "http://10.0.1.1:2379,http://localhost:2379" \
  --advertise-client-urls "http://10.0.1.1:2379" \
  --listen-peer-urls "http://10.0.1.1:2380" \
  --initial-cluster "kube1=http://10.0.1.1:2380,kube2=http://10.0.1.2:2380,kube3=http://10.0.1.3:2380" \
  --initial-advertise-peer-urls "http://10.0.1.1:2380" \
  --heartbeat-interval 200 \
  --election-timeout 5000
Restart=always
RestartSec=5
TimeoutStartSec=0
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```

It's important to understand that each flag starting with `--initial` does only apply during the first launch of a cluster. This means for example, that it's possible to add and remove cluster members at any time without ever changing the value of `--initial-cluster`.

After the files have been placed on each host, it's time to start the etcd cluster:

```sh
systemctl enable etcd.service # launch etcd during system boot
systemctl start etcd.service
```

Executing `/opt/etcd/etcdctl member list` should show a list of cluster members. If something went wrong check the logs using  `journalctl -u etcd.service`.

### Kubernetes setup

Now that Docker is configured and etcd is running, it's time to deploy Kubernetes. The first step is to install the required packages on each host:

```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial-unstable main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

#### Initializing the master node

Before initializing the master node, we need to create a manifest on kube1 which will then be used as configuration in the next step:

```yaml
# /tmp/master-configuration.yml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: 10.0.1.1
etcd:
  endpoints:
  - http://10.0.1.1:2379
  - http://10.0.1.2:2379
  - http://10.0.1.3:2379
apiServerCertSANs:
  - <PUBLIC_IP_KUBE1>
```

Then we run the following command on kube1:

```sh
kubeadm init --config /tmp/master-configuration.yml
```
After the setup is complete, kubeadm prints a token such as `818d5a.8b50eb5477ba4f40`. It's important to write it down, we'll need it in a minute to join the other cluster nodes.

Kubernetes is built around openness, so it's up to us to choose and install a suitable pod network. This is required as it enables pods running on different nodes to communicate with each other. One of the [many options](https://kubernetes.io/docs/concepts/cluster-administration/addons/) is [Weave Net](https://www.weave.works/products/weave-net/). It requires zero configuration and is considered stable and well-maintained:

```sh
# create symlink for the current user in order to gain access to the API server with kubectl
[ -d $HOME/.kube ] || mkdir -p $HOME/.kube
ln -s /etc/kubernetes/admin.conf $HOME/.kube/config

# install Weave Net
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# allow traffic on the newly created weave network interface
ufw allow in on weave
ufw reload
```

Unfortunately, Weave Net will not readily work with our current cluster configuration because traffic will be routed via the wrong network interface. This can be fixed by running the following command on each host:

```sh
ip route add 10.96.0.0/16 dev $VPN_INTERFACE src $VPN_IP

# on kube1:
ip route add 10.96.0.0/16 dev wg0 src 10.0.1.1
# on kube2:
ip route add 10.96.0.0/16 dev wg0 src 10.0.1.2
# on kube3:
ip route add 10.96.0.0/16 dev wg0 src 10.0.1.3
```

The added route will not survive a reboot as it is not persistent. To ensure that the route gets added after a reboot, we have to add a *systemd* service unit on each node which will wait for the wireguard interface to come up and after that adds the route. For kube1 it would look like this:

```sh
# /etc/systemd/system/overlay-route.service
[Unit]
Description=Overlay network route for Wireguard
After=wg-quick@wg0.service

[Service]
Type=oneshot
User=root
ExecStart=/sbin/ip route add 10.96.0.0/16 dev wg0 src 10.0.1.1

[Install]
WantedBy=multi-user.target
```

After that we have to enable it by running following command:

```sh
systemctl enable overlay-route.service
```


#### Joining the cluster nodes

All that's left is to join the cluster with the other nodes. Run the following command on each host:

```sh
kubeadm join --token=<TOKEN> 10.0.1.1:6443 --discovery-token-unsafe-skip-ca-verification
```

That's it, a Kubernetes cluster is ready at our disposal.

## Access and operations

![Terraform](assets/terraform.png) [`service/kubernetes`](https://github.com/hobby-kube/provisioning/tree/master/service/kubernetes)

As soon as the cluster is running, we want to be able to access the Kubernetes API remotely. This can be done by copying `/etc/kubernetes/admin.conf` from kube1 to your own machine. After [installing kubectl](https://kubernetes.io/docs/tasks/kubectl/install/) locally, execute the following commands:

```sh
# create local config folder
mkdir -p ~/.kube
# backup old config if required
[ -f ~/.kube/config ] && cp ~/.kube/config ~/.kube/config.backup
# copy config from master node
scp root@<PUBLIC_IP_KUBE1>:/etc/kubernetes/admin.conf ~/.kube/config
# change config to use correct IP address
kubectl config set-cluster kubernetes --server=https://<PUBLIC_IP_KUBE1>:6443
```

You're now able to remotely access the Kubernetes API. Running `kubectl get nodes` should show a list of nodes similar to this:

```sh
NAME      STATUS    AGE       VERSION
kube1     Ready     1h        v1.9.1
kube2     Ready     1h        v1.9.1
kube3     Ready     1h        v1.9.1
```

### Role-Based Access Control

As of version 1.6, kubeadm configures Kubernetes with RBAC enabled. Because our hobby cluster is typically operated by trusted people, we should enable permissive RBAC permissions to be able to deploy any kind of services using any kind of resources. If you're in doubt whether this is secure enough for your use case, please refer to the official [RBAC documentation](https://kubernetes.io/docs/admin/authorization/rbac).

```sh
kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts
```

### Deploying services

Services can now be deployed remotely by calling `kubectl -f apply <FILE>`. It's also possible to apply multiple files by pointing to a folder, for example:

```sh
$ ls dashboard/
deployment.yml  service.yml

$ kubectl apply -f dashboard/
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

This guide will make no further explanations in this regard. Please refer to the official documentation on [kubernetes.io](https://kubernetes.io/).

## Bringing traffic to the cluster

There are downsides to running Kubernetes outside of well integrated platforms such as AWS or GCE. One of those is the lack of external ingress and load balancing solutions. Fortunately, it's fairly easy to get an NGINX powered ingress controller running inside the cluster, which will enable services to register for receiving public traffic.

### Ingress controller setup

Because there's no load balancer available with most cloud providers, we have to make sure the NGINX server is always running on the same host, accessible via an IP address that doesn't change. As our master node is pretty much idle at this point, and no ordinary pods will get scheduled on it, we make kube1 our dedicated host for routing public traffic.

We already opened port 80 and 443 during the initial firewall configuration, now all we have to do is to write a couple of manifests to deploy the NGINX ingress controller on kube1:

- [ingress/00-namespace.yml](https://github.com/hobby-kube/manifests/blob/master/ingress/00-namespace.yml)
- [ingress/deployment.yml](https://github.com/hobby-kube/manifests/blob/master/ingress/deployment.yml)
- [ingress/service.yml](https://github.com/hobby-kube/manifests/blob/master/ingress/service.yml)
- [ingress/configmap.yml](https://github.com/hobby-kube/manifests/blob/master/ingress/configmap.yml)

One part requires special attention. In order to make sure NGINX runs on kube1—which is a tainted master node and no pods will normally be scheduled on it—we need to specify a toleration:

```yaml
# from ingress/deployment.yml
tolerations:
- key: node-role.kubernetes.io/master
  operator: Equal
  effect: NoSchedule
```

Specifying a toleration doesn't make sure that a pod is getting scheduled on any specific node. For this we need to add a node affinity rule. As we have just a single master node, the following specification is enough to schedule a pod on kube1:

```yaml
# from ingress/deployment.yml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-role.kubernetes.io/master
          operator: Exists
```

Running `kubectl apply -f ingress/` will apply all manifests in this folder. First, a namespace called *ingress* is created, followed by the NGINX deployment, plus a default backend to serve 404 pages for undefined domains and routes including the necessary service object. There's no need to define a service object for NGINX itself, because we configure it to use the host network (`hostNetwork: true`), which means that the container is bound to the actual ports on the host, not to some virtual interface within the pod overlay network.

Services are now able to make use of the ingress controller and receive public traffic with a simple manifest:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: service.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: example-service
          servicePort: example-service-http
```

The NGINX ingress controller is quite flexible and supports a whole bunch of [configuration options](https://github.com/kubernetes/ingress/blob/master/controllers/nginx/configuration.md).

### DNS records

![Terraform](assets/terraform.png) [`dns/cloudflare`](https://github.com/hobby-kube/provisioning/tree/master/dns/cloudflare)
![Terraform](assets/terraform.png) [`dns/google`](https://github.com/hobby-kube/provisioning/tree/master/dns/google)

At this point we could use a domain name and put some DNS entries into place. To serve web traffic it's enough to create an A record pointing to the public IP address of kube1 plus a wildcard entry to be able to use subdomains:

| Type  | Name          | Value             |
| ----- | ------------- | ----------------- |
| A     | example.com   | <PUBLIC_IP_KUBE1> |
| CNAME | *.example.com | example.com       |

Once the DNS entries are propagated our example service would be accessible at `http://service.example.com`. If you don't have a domain name at hand, you can always add an entry to your hosts file instead.

Additionally, it might be a good idea to assign a subdomain to each host, e.g. kube1.example.com. It's way more comfortable to ssh into a host using a domain name instead of an IP address.

### Obtaining SSL/TLS certificates

Thanks to [Let’s Encrypt](https://letsencrypt.org/) and a project called [kube-lego](https://github.com/jetstack/kube-lego) it's incredibly easy to obtain free certificates for any domain name pointing at our Kubernetes cluster. Setting this service up takes no time and it plays well with the NGINX ingress controller we deployed earlier. These are the related manifests:

- [ingress/tls/deployment.yml](https://github.com/hobby-kube/manifests/blob/master/ingress/tls/deployment.yml)
- [ingress/tls/configmap.yml](https://github.com/hobby-kube/manifests/blob/master/ingress/tls/configmap.yml)

Before deploying kube-lego using the manifests above, make sure to replace the email address in `ingress/tls/configmap.yml` with your own.

To enable certificates for a service, the ingress manifest needs to be slightly extended:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/tls-acme: "true" # enable certificates
    kubernetes.io/ingress.class: "nginx"
spec:
  tls: # specify domains to fetch certificates for
  - hosts:
    - service.example.com
    secretName: example-service-tls
  rules:
  - host: service.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: example-service
          servicePort: example-service-http
```

After applying this manifest, kube-lego will try to obtain a certificate for service.example.com and reload the NGINX configuration to enable TLS. Make sure to check the logs of the kube-lego pod if something goes wrong.

NGINX will automatically redirect clients to HTTPS whenever TLS is enabled. In case you still want to serve traffic on HTTP, add `ingress.kubernetes.io/ssl-redirect: "false"`  to the list of annotations.

### Deploying the Kubernetes Dashboard

Now that everything is in place, we are able to expose services on specific domains and automatically obtain certificates for them. Let's try this out by deploying the [Kubernetes Dashboard](https://github.com/kubernetes/dashboard) with the following manifests:

- [dashboard/deployment.yml](https://github.com/hobby-kube/manifests/blob/master/dashboard/deployment.yml)
- [dashboard/service.yml](https://github.com/hobby-kube/manifests/blob/master/dashboard/service.yml)
- [dashboard/ingress.yml](https://github.com/hobby-kube/manifests/blob/master/dashboard/ingress.yml)
- [dashboard/secret.yml](https://github.com/hobby-kube/manifests/blob/master/dashboard/secret.yml)

Optionally, the following manifests can be used to get resource utilization graphs within the dashboard using [heapster](https://github.com/kubernetes/heapster):

- [dashboard/heapster/deployment.yml](https://github.com/hobby-kube/manifests/blob/master/dashboard/heapster/deployment.yml)
- [dashboard/heapster/service.yml](https://github.com/hobby-kube/manifests/blob/master/dashboard/heapster/service.yml)

What's new here is that we enable **basic authentication** to restrict access to the dashboard. The following annotations are supported by the NGINX ingress controller, and may or may not work with other solutions:

```yaml
# from dashboard/ingress.yml
annotations:
  # ...
  ingress.kubernetes.io/auth-type: basic
  ingress.kubernetes.io/auth-secret: kubernetes-dashboard-auth
  ingress.kubernetes.io/auth-realm: "Authentication Required"

# dashboard/secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: kubernetes-dashboard-auth
  namespace: kube-system
data:
  auth: YWRtaW46JGFwcjEkV3hBNGpmQmkkTHYubS9PdzV5Y1RFMXMxMWNMYmJpLw==
type: Opaque
```

This example will prompt a visitor to enter their credentials (user: admin / password: test) when accessing the dashboard. Secrets for basic authentication can be created using `htpasswd`, and need to be added to the manifest as a base64 encoded string.

## Distributed block storage

Data in containers is ephemeral, as soon as a pod gets stopped, crashes, or is for some reason rescheduled to run on another node, all its data is gone. While this is fine for static applications such as the Kubernetes Dashboard (which obtains its data from persistent sources running outside of the container), persisting data becomes a non-optional requirement as soon as we deploy databases on our cluster.

Kubernetes supports various [types of volumes](https://kubernetes.io/docs/concepts/storage/volumes/) that can be attached to pods. Only a few of these match our requirements. There's the `hostPath` type which simply maps a path on the host to a path inside the container, but this won't work because we don't know on which node a pod will be scheduled.

### Persistent volumes

There's a concept within Kubernetes of how to separate storage management from cluster management. To provide a layer of abstraction around storage there are two types of resources deeply integrated into Kubernetes, *PersistentVolume* and *PersistentVolumeClaim*. When running on well integrated platforms such as GCE, AWS or Azure, it's really easy to attach a persistent volume to a pod by creating a persistent volume claim. Unfortunately, we don't have access to such solutions.

Our cluster consists of multiple nodes and **we need the ability to attach persistent volumes to any pod running on any node**. There are a couple of projects and companies emerging around the idea of providing hyper-converged storage solutions. Some of their services are running as pods within Kubernetes itself, which is certainly the perfect way of managing storage on a small cluster such as ours.

### Choosing a solution

Currently there are a couple of interesting solutions matching our criteria, but they all have their downsides:

- [Rook.io](https://rook.io/) is an open source project based on Ceph. It looks promising, but is still pretty much in alpha state and lacking some serious documentation.
- [gluster-kubernetes](https://github.com/gluster/gluster-kubernetes) is an open source project built around GlusterFS and Heketi. Setup seems tedious at this point, requiring some kind of schema to be provided in JSON format.
- [Portworx](https://portworx.com/) is a commercial project that offers a [free variant](https://github.com/portworx/px-dev) of their proprietary software, providing great documentation and tooling.

Even though we would definitely prefer using open source software, Portworx offers the best solution currently available. Setup is simple, deployment and operation is transparent. It launches just a single pod per instance where others create a whole bunch of pods and sidecars. Things might change in the future, but for now we're going to settle on Portworx.

### Deploying Portworx

As we run only a three node cluster, we're going to deploy PX on all three of them using a DaemonSet with master toleration. The [official documentation](https://docs.portworx.com/run-with-kubernetes.html) states that PX should be deployed manually on each host using `docker run`. The main reason behind this statement is probably PX's need for mounting volumes in shared mode (e.g. `-v /host/path:/container/path:shared`). Officially, the Kubernetes pod specification doesn't support this flag, but we can work around this by appending `:shared` to the mount path in the pod spec.

Before deploying the Portworx DaemonSet we need to provide a raw, unformatted block device that will be used for storage on each host. These can either be attached volumes or local loopback devices. On Scaleway, the volume on which the operating system is installed is called `/dev/vda`. Attaching another volume will be available as  `/dev/vdb`. On DigitalOcean things work a little differently. Attached volumes are referenced with something like  `/dev/disk/by-id/scsi-0DO_Volume_<VOLUME_NAME>`.

Make sure to edit the daemonset manifest listed below and replace the value of the `PX_STORAGE_DEVICE` env variable with a block device available in your environment. The resulting manifests turn out pretty lean for such a seemingly complex service:

- [storage/daemonset.yml](https://github.com/hobby-kube/manifests/blob/master/storage/daemonset.yml)
- [storage/storageclass.yml](https://github.com/hobby-kube/manifests/blob/master/storage/storageclass.yml)

It's worth mentioning that the storage class manifest contains a few important parameters:

```yaml
# storage/storageclass.yml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: portworx
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: "2" # replication factor
  snap_interval: "0" # turn off automatic snapshots
  io_priority: "high"
```

Further parameters are listed in the [Portworx storage class documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#portworx-volume).

In order to operate on the storage cluster use the `pxctl` control tool ([documentation](https://docs.portworx.com/cli-reference.html)) via one of the Portworx containers. Here are some examples:

```sh
# show status summary
kubectl exec -it portworx-storage-wp797 -- /opt/pwx/bin/pxctl status
# list volumes in the cluster
kubectl exec -it portworx-storage-wp797 -- /opt/pwx/bin/pxctl volume list
# show cluster wide alerts
kubectl exec -it portworx-storage-wp797 -- /opt/pwx/bin/pxctl cluster alerts
```

### Consuming storage

The storage class we created can be consumed with a persistent volume claim:

```yaml
# minio/pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-persistent-storage
spec:
  storageClassName: portworx
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

This will create a volume called *minio-persistent-storage* with 5GB storage capacity. Please note that there's currently a bug in PX related to ReadWriteMany volume claims ([bug report](https://github.com/portworx/px-dev/issues/23)). Volumes can only be claimed in ReadWriteOnce access mode for the time being.

In this example we're deploying [Minio](https://minio.io), an Amazon S3 compatible object storage server, to create and mount a persistent volume:

- [minio/deployment.yml](https://github.com/hobby-kube/manifests/blob/master/minio/deployment.yml)
- [minio/ingress.yml](https://github.com/hobby-kube/manifests/blob/master/minio/ingress.yml)
- [minio/secret.yml](https://github.com/hobby-kube/manifests/blob/master/minio/secret.yml) (MINIO_ACCESS_KEY: admin / MINIO_SECRET_KEY: admin.minio.secret.key)
- [minio/service.yml](https://github.com/hobby-kube/manifests/blob/master/minio/service.yml)
- [minio/pvc.yml](https://github.com/hobby-kube/manifests/blob/master/minio/pvc.yml)

The volume related configuration is buried in the deployment manifest:

```yaml
# from minio/deployment.yml
containers:
- name: minio
  volumeMounts:
  - name: data
    mountPath: /data
# ...
volumes:
- name: data
  persistentVolumeClaim:
    claimName: minio-persistent-storage
```

The *minio-persistent-storage* volume will live as long as the persistent volume claim is not deleted (e.g. `kubectl delete -f minio/pvc.yml `). The Minio pod itself can be deleted, updated or rescheduled without data loss.

## Where to go from here

This was hopefully just the beginning of your journey. There are many more things to explore around Kubernetes. Feel free to leave feedback or raise questions at any time by opening an issue [here](https://github.com/hobby-kube/guide/issues).
