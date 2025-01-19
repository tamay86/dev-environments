# Piraeus Datastore

<https://piraeus.io/docs/stable/tutorial/get-started/>

## Vorbereitung

* Talos benötigt das DRBD Modul

```bash
 factory.talos.dev/installer/e048aaf4461ff9f9576c9a42f760f2fef566559bd4933f322853ac291e46f238:v1.9.2 
 ```

## Talos

Abweichende Erzeugung der Konfiguration

```shell
# Generate talos config
talosctl gen config dev https://192.168.100.10:6443 \
  --output talos \
  --install-disk /dev/vda \
  --install-image factory.talos.dev/installer/e048aaf4461ff9f9576c9a42f760f2fef566559bd4933f322853ac291e46f238:v1.9.2 \
  --config-patch-control-plane @patches/controlplane.yaml \
  --config-patch @patches/piraeus/drbd.yaml
```
