# Vagrant

Testumgebung für Vagrant (mit vagrant-libvirt) für Talos bestehend aus einer Controlplane und einem Worker

## Vorbereitung

* Talosctl installieren <https://www.talos.dev/v1.8/talos-guides/install/talosctl/>
* Talos ISO herunterladen

```shell
# Download Talos Iso
mkdir -p talos
if [[ ! -f talos/metal-amd64.iso ]]
then
  wget -q https://github.com/siderolabs/talos/releases/download/v1.9.1/metal-amd64.iso -O talos/metal-amd64.iso
fi
```

## Start der VMs

```shell
vagrant up
```

Es werden 2 NICs erstellt, die erste NIC hat 192.168.121.0/24 aber keine IPv6. Habe keine Möglichkeit bisher gefunden, der ersten NIC das mitzugeben
Daher wird nur NIC2 benötigt, die sowohl IPv4 als auch IPv6 aktiviert hat. Mittels `virsh` kann die erste NIC entfernt werden.

```shell
# IP und MAC rausfinden
MAC_CP=$(virsh -c qemu:///system domifaddr --domain dev-home.tamay.cloud_k8s-node-01 | grep 121 | awk '{ print $2}'); echo $MAC_CP
IP_CP=$(virsh -c qemu:///system domifaddr --domain dev-home.tamay.cloud_k8s-node-01 | grep 111 | awk '{ print $4 }' | cut -f1 -d'/'); echo $IP_CP
# NIC1 entfernen
virsh -c qemu:///system detach-interface --domain dev-home.tamay.cloud_k8s-node-01 --mac ${MAC_CP} --type network
```

## Talos

Jetzt kann die Talos Konfiguration erstellt und angewendet werden

```shell
# Generate talos config
talosctl gen config dev https://[fd00::10]:6443 \
  --output talos \
  --install-disk /dev/vda \
  --config-patch @patches/patch.yaml \
  --config-patch-control-plane @patches/controlplane.yaml

# Apply Config to first Controlplane, including patch for hostname
talosctl apply-config -n ${IP_CP} \
  --insecure \
  --file talos/controlplane.yaml \
  --config-patch @patches/node-cp-1.yaml
export TALOSCONFIG=$(realpath ./talos/talosconfig)
talosctl config endpoint ${IP_CP}
talosctl -n ${IP_CP} bootstrap
talosctl -n ${IP_CP} kubeconfig ./kubeconfig
export KUBECONFIG=./kubeconfig
```

### Quick export

```shell
IP_CP=$(virsh -c qemu:///system domifaddr --domain dev-home.tamay.cloud_k8s-node-01 | grep 111 | awk '{ print $4 }' | cut -f1 -d'/')
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
```

## MetalLB

```shell
kubectl apply -f metallb/namespace.yaml
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb --namespace metallb-system
kubectl apply -f metallb/pool.yaml
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

## Cilium Notes

https://github.com/cilium/cilium/issues/34997#issuecomment-2380284075

```text
Hi! I am running Cilium 1.16.1 in a pure IPv6 only environment and it's been holding up just fine. If your nodes are IPv6 only you will need to install Cilium explicitly with IPv4 disabled. Please note that by default Cilium uses routingMode=tunnel but if you are running in a pure IPv6 only environment as of now due to #17240 the only way to make Cilium work with pure IPv6 is by using routingMode=native for which you will need to provide a few more config bits all of which I am listing below. All the best!

Documentation about native routing https://docs.cilium.io/en/stable/network/concepts/routing/#routing

ipv4.enabled=false
ipv6.enabled=true 
routingMode=native
set autodirectnoderoutes=true
ipv6NativeRoutingCIDR= 
ipam.operator.clusterPoolIPv6PodCIDRList=
k8s.requireIPv6PodCIDR=
```
