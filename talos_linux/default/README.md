# Vagrant

Testumgebung für Vagrant (mit vagrant-libvirt) für Talos bestehend aus einer Controlplane und einem Worker

## Vorbereitung

* Talosctl installieren <https://www.talos.dev/v1.9/talos-guides/install/talosctl/>
* Talos ISO herunterladen

```shell
# Download Talos Iso
if [[ ! -f /tmp/metal-amd64.iso ]]
then
  wget -q https://github.com/siderolabs/talos/releases/download/v1.9.2/metal-amd64.iso -O /tmp/metal-amd64.iso
fi
```

## Start der VMs

```shell
vagrant up
```

Es werden 2 NICs erstellt, die erste NIC hat 192.168.121.0/24 aber keine IPv6. Habe keine Möglichkeit bisher gefunden, der ersten NIC das mitzugeben.
Daher wird nur NIC2 benötigt, die sowohl IPv4 als auch IPv6 aktiviert hat. Mittels `virsh` kann die erste NIC entfernt werden.

```shell
# IP und MAC rausfinden
MAC_CP=$(virsh -c qemu:///system domifaddr --domain default_cp-01 | grep 121 | awk '{ print $2}'); echo $MAC_CP
MAC_WORKER_01=$(virsh -c qemu:///system domifaddr --domain default_worker-01 | grep 121 | awk '{ print $2}'); echo $MAC_WORKER_01
MAC_WORKER_02=$(virsh -c qemu:///system domifaddr --domain default_worker-02 | grep 121 | awk '{ print $2}'); echo $MAC_WORKER_02
MAC_WORKER_03=$(virsh -c qemu:///system domifaddr --domain default_worker-03 | grep 121 | awk '{ print $2}'); echo $MAC_WORKER_03
IP_CP=$(virsh -c qemu:///system domifaddr --domain default_cp-01 | grep 100 | awk '{ print $4 }' | cut -f1 -d'/'); echo $IP_CP
IP_WORKER_01=$(virsh -c qemu:///system domifaddr --domain default_worker-01 | grep 100 | awk '{ print $4 }' | cut -f1 -d'/'); echo $IP_WORKER_01
IP_WORKER_02=$(virsh -c qemu:///system domifaddr --domain default_worker-02 | grep 100 | awk '{ print $4 }' | cut -f1 -d'/'); echo $IP_WORKER_02
IP_WORKER_03=$(virsh -c qemu:///system domifaddr --domain default_worker-03 | grep 100 | awk '{ print $4 }' | cut -f1 -d'/'); echo $IP_WORKER_03
# NIC1 entfernen
virsh -c qemu:///system detach-interface --domain default_cp-01 --mac ${MAC_CP} --type network
virsh -c qemu:///system detach-interface --domain default_worker-01 --mac ${MAC_WORKER_01} --type network
virsh -c qemu:///system detach-interface --domain default_worker-02 --mac ${MAC_WORKER_02} --type network
virsh -c qemu:///system detach-interface --domain default_worker-03 --mac ${MAC_WORKER_03} --type network
```

## Talos

Jetzt kann die Talos Konfiguration erstellt und angewendet werden

```shell
# Generate talos config
talosctl gen config dev https://192.168.100.10:6443 \
  --output talos \
  --install-disk /dev/vda \
  --config-patch-control-plane @patches/controlplane.yaml

# Apply controlplane config to cp-01
talosctl -n ${IP_CP} apply-config \
  --insecure \
  --file talos/controlplane.yaml \
  --config-patch @patches/nodes/cp-01.yaml

# Apply config to workers
talosctl -n ${IP_WORKER_01} apply-config \
  --insecure \
  --file talos/worker.yaml \
  --config-patch @patches/nodes/worker-01.yaml
talosctl -n ${IP_WORKER_02} apply-config \
  --insecure \
  --file talos/worker.yaml \
  --config-patch @patches/nodes/worker-02.yaml
talosctl -n ${IP_WORKER_03} apply-config \
  --insecure \
  --file talos/worker.yaml \
  --config-patch @patches/nodes/worker-03.yaml

export TALOSCONFIG=$(realpath ./talos/talosconfig)
talosctl config endpoint ${IP_CP}
talosctl -n ${IP_CP} bootstrap
talosctl -n ${IP_CP} kubeconfig ./kubeconfig
export KUBECONFIG=./kubeconfig
```

### Quick export

```shell
IP_CP=$(virsh -c qemu:///system domifaddr --domain default_cp-01 | grep 100 | awk '{ print $4 }' | cut -f1 -d'/'); echo $IP_CP
export TALOSCONFIG=$(realpath ./talos/talosconfig)
talosctl config endpoint ${IP_CP}
export KUBECONFIG=./kubeconfig
```

## Cilium

```shell
helm upgrade --install \
    cilium \
    cilium/cilium \
    --namespace kube-system \
    --values cilium/values.yaml
# Wait a bit for the cilium CRDs to be deployed
kubectl apply -f cilium/cilium-lb-l2.yaml
```

## Local path provisioner

```shell
kubectl apply -k local-path-provisioner/
```

## Test Deployment

Nginx Pod mit LoadBalancer Service

```shell
kubectl apply -f test-deployment.yaml
```

## Notizen

Cilium L2 Advertisements für IPv6 funnktionieren nicht
<https://docs.cilium.io/en/latest/network/l2-announcements/#limitations>

```text
The feature currently does not support IPv6/NDP.
```

## Upgrade von Talos

* Nicht alle Nodes auf einmal upgraden
* Image muss mitangegeben werden

```shell
talosctl --nodes ${IP_CP} upgrade --image ghcr.io/siderolabs/installer:v1.8.3
```
