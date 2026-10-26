# Highly Available Kubernetes Cluster with kubeadm

**Guide:** DevOps Shack\
**Platform:** Ubuntu Linux VMs, kubeadm, containerd, HAProxy\
**Topology:** 3 control-plane nodes, 3 worker nodes, 1 load-balancer VM

> **Important:** This guide uses three control-plane nodes so the
> stacked etcd cluster can tolerate one control-plane/etcd member
> failure. A single HAProxy VM is still a single point of failure for
> API access. For load-balancer HA, deploy two load balancers with a VIP
> managed by a tool such as Keepalived.

## 1. Architecture

``` text
                  Admin / kubectl
                         |
                  LOAD_BALANCER_IP:6443
                         |
                      HAProxy
                 /       |       \
              CP1       CP2       CP3
             :6443     :6443     :6443
                 \       |       /
                   Kubernetes API

           Worker 1    Worker 2    Worker 3
```

Replace placeholders such as `LOAD_BALANCER_IP` and `<MASTER1_IP>` with
your actual addresses. Ensure the nodes can communicate over the
required Kubernetes ports and that the load balancer can reach each
control-plane node on TCP/6443.

## 2. Prepare the load-balancer node

### Install HAProxy

``` bash
sudo apt-get update
sudo apt-get install -y haproxy
```

### Configure HAProxy

Back up the existing configuration, then edit it:

``` bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo nano /etc/haproxy/haproxy.cfg
```

Add or merge the following configuration with the existing
global/default sections:

``` haproxy
frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    default-server inter 3s fall 3 rise 2
    server master1 <MASTER1_IP>:6443 check
    server master2 <MASTER2_IP>:6443 check
    server master3 <MASTER3_IP>:6443 check
```

Validate and restart HAProxy:

``` bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy
sudo systemctl restart haproxy
sudo systemctl status haproxy --no-pager
```

### Optional: HAProxy statistics page

Add this section to the HAProxy configuration if you need a basic stats
page:

``` haproxy
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /
    stats refresh 10s
```

Restrict port `8404` with a firewall to trusted administrator addresses.
Do not expose the statistics page publicly.

## 3. Prepare all Kubernetes nodes

Run these steps on all three control-plane nodes and all three workers.

### Install a container runtime

Kubernetes needs a CRI-compatible runtime. This guide uses `containerd`;
Docker Engine alone is not a CRI runtime for kubelet unless you also
install and configure `cri-dockerd`.

``` bash
sudo apt-get update
sudo apt-get install -y containerd
sudo install -d /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

Configure the runtime to use the systemd cgroup driver. On containerd
versions using the standard config format, set `SystemdCgroup = true`
under the
`plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options`
section. Check the configuration format for your installed containerd
version.

``` bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl status containerd --no-pager
```

### Install kubeadm, kubelet, and kubectl

This example uses the Kubernetes v1.30 package repository. Kubernetes
v1.30 is an older release; for a new deployment, select a currently
supported Kubernetes minor version and use that same minor version
consistently across the nodes.

``` bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gpg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

> The kubelet may restart repeatedly before `kubeadm init` or
> `kubeadm join`; this is expected during initial setup.

Before proceeding, configure unique hostnames, working DNS or
`/etc/hosts` entries, synchronized time, required firewall rules, and
swap according to the Kubernetes installation requirements. Verify that
every node can reach the load balancer and that the load balancer can
reach all control-plane API endpoints.

## 4. Initialize the first control-plane node

Run on control-plane node 1:

``` bash
sudo kubeadm init \
  --control-plane-endpoint "LOAD_BALANCER_IP:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16
```

Save the generated `kubeadm join` commands and certificate key securely.
The certificate upload key is time-limited; regenerate it if it expires.

Configure kubectl for the current user:

``` bash
mkdir -p "$HOME/.kube"
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

Check the API endpoint:

``` bash
kubectl get nodes
kubectl get pods -n kube-system
```

## 5. Install a CNI network plugin

The pod network CIDR must match the CNI configuration. For this example,
the kubeadm pod CIDR is `10.244.0.0/16`.

Install a currently supported release of your chosen CNI plugin by
following its official installation instructions. Do not mix CNI plugins
in the same cluster.

**Example only:** Flannel commonly uses `10.244.0.0/16`, but confirm the
manifest and compatibility for your chosen version before applying it.

``` bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Wait for the system pods to become ready:

``` bash
kubectl get pods -A
kubectl get nodes
```

## 6. Join control-plane nodes 2 and 3

On control-plane node 1, create a fresh join command:

``` bash
kubeadm token create --print-join-command
```

Generate or retrieve the control-plane certificate key as needed:

``` bash
sudo kubeadm init phase upload-certs --upload-certs
```

Run the join command shown by kubeadm on control-plane nodes 2 and 3,
adding the control-plane flags and the certificate key:

``` bash
sudo kubeadm join LOAD_BALANCER_IP:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_CERT_HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY>
```

Use the actual token, hash, and certificate key generated by your
cluster. Do not publish these values in a public repository.

Configure kubectl on each control-plane node if you intend to administer
the cluster from that node:

``` bash
mkdir -p "$HOME/.kube"
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

## 7. Join the worker nodes

On a control-plane node, generate a fresh worker join command:

``` bash
kubeadm token create --print-join-command
```

Run the resulting command on each worker node. It will look similar to:

``` bash
sudo kubeadm join LOAD_BALANCER_IP:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_CERT_HASH>
```

## 8. Verify cluster health

Run from a machine with a working kubeconfig:

``` bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get --raw='/readyz?verbose'
```

All six nodes should eventually report `Ready`, assuming the CNI and
other prerequisites are configured correctly.

## 9. Verify etcd health

On a control-plane node, use an `etcdctl` version compatible with the
etcd version shipped with your Kubernetes release. Installing an
arbitrary distribution package may give you an incompatible version.

For kubeadm's stacked-etcd layout, the health-check client certificate
is typically available at
`/etc/kubernetes/pki/etcd/healthcheck-client.crt` and its key at
`/etc/kubernetes/pki/etcd/healthcheck-client.key`.

``` bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint health
```

Check membership:

``` bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  member list
```

Repeat endpoint health checks on each control-plane node as appropriate.
Keep certificate and key files private.

## 10. Verify HAProxy

Check the service and configuration:

``` bash
sudo systemctl status haproxy --no-pager
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Test API access through the load balancer:

``` bash
curl -k https://LOAD_BALANCER_IP:6443/version
```

A successful response confirms that the API endpoint is reachable
through HAProxy; it does not by itself prove full cluster high
availability.

## 11. Test failover safely

Perform failure tests only in a maintenance window or a non-production
environment.

1.  Confirm all three control-plane nodes and etcd members are healthy.
2.  Stop or isolate **one** control-plane node at a time.
3.  Test API access through the load-balancer endpoint.
4.  Check HAProxy's backend status and run `kubectl get nodes`.
5.  Restore the node and wait for it to recover before testing another.

**Do not stop Docker containers on a Kubernetes node as a generic
failure test.** This guide uses containerd, and stopping all containers
can disrupt system components. A node failure test should be planned and
observed carefully.

A three-member etcd cluster can tolerate one member being unavailable
while the other two remain healthy. Two unavailable members would lose
etcd quorum.

## 12. Important notes

-   The load balancer endpoint must remain stable and match the
    `--control-plane-endpoint` used by kubeadm.
-   A single HAProxy VM is a single point of failure. For end-to-end API
    access HA, make the load-balancer tier redundant too.
-   Keep the Kubernetes minor version, kubeadm, kubelet, kubectl,
    container runtime, and CNI versions compatible.
-   The old Ingress-NGINX `controller-v0.49.0` manifest from many older
    tutorials should not be copied into a new deployment without
    checking current support and security status. Ingress is optional
    for Kubernetes API high availability.
-   Never commit real tokens, certificate keys, kubeconfig files, or
    private keys to a public Git repository.

## References

-   kubeadm high availability:
    https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
-   kubeadm implementation:
    https://kubernetes.io/docs/reference/setup-tools/kubeadm/
-   Kubernetes package repositories:
    https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
-   HAProxy documentation: https://www.haproxy.org/
-   Flannel: https://github.com/flannel-io/flannel
