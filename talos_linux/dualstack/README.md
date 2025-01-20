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
MAC_CP=$(virsh -c qemu:///system domifaddr --domain dualstack_cp-01 | grep 121 | awk '{ print $2}'); echo $MAC_CP
MAC_WORKER_01=$(virsh -c qemu:///system domifaddr --domain dualstack_worker-01 | grep 121 | awk '{ print $2}'); echo $MAC_WORKER_01
IP_CP=$(virsh -c qemu:///system domifaddr --domain dualstack_cp-01 | grep 100 | awk '{ print $4 }' | cut -f1 -d'/'); echo $IP_CP
IP_WORKER_01=$(virsh -c qemu:///system domifaddr --domain dualstack_worker-01 | grep 100 | awk '{ print $4 }' | cut -f1 -d'/'); echo $IP_WORKER_01
# NIC1 entfernen
virsh -c qemu:///system detach-interface --domain dualstack_cp-01 --mac ${MAC_CP} --type network
virsh -c qemu:///system detach-interface --domain dualstack_worker-01 --mac ${MAC_WORKER_01} --type network
```

## Talos

Jetzt kann die Talos Konfiguration erstellt und angewendet werden

```shell
# Generate talos config
talosctl gen config dev https://[fd00::10]:6443 \
  --output talos \
  --install-disk /dev/vda \
  --config-patch-control-plane @patches/controlplane/allow-scheduling.yaml \
  --config-patch-control-plane @patches/controlplane/disable-proxy-and-cni.yaml \
  --config-patch @patches/all/pod-service-subnet.yaml

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

export TALOSCONFIG=$(realpath ./talos/talosconfig)
talosctl config endpoint fd00::81
talosctl -n fd00::81 bootstrap
talosctl -n fd00::81 kubeconfig ./kubeconfig
export KUBECONFIG=./kubeconfig
```

### Quick export

```shell
export TALOSCONFIG=$(realpath ./talos/talosconfig)
talosctl config endpoint fd00::81
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

## Local path provisioner

```shell
kubectl apply -k local-path-provisioner/
```

## Metallb

```shell
kubectl apply -f metallb/namespace.yaml
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb --namespace metallb-system
kubectl apply -f metallb/pool.yaml
```

## Adguard

```shell
kubectl apply -f adguard.yaml
```
