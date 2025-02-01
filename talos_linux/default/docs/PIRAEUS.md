# Piraeus Datastore

<https://piraeus.io/docs/stable/tutorial/get-started/>

## Vorbereitung

* Talos benötigt das DRBD Modul

```bash
 factory.talos.dev/installer/e048aaf4461ff9f9576c9a42f760f2fef566559bd4933f322853ac291e46f238:v1.9.3
 ```

## Talos

Abweichende Erzeugung der Konfiguration

```shell
# Generate talos config
talosctl gen config dev https://192.168.100.10:6443 \
  --output talos \
  --install-disk /dev/vda \
  --install-image factory.talos.dev/installer/e048aaf4461ff9f9576c9a42f760f2fef566559bd4933f322853ac291e46f238:v1.9.3 \
  --config-patch-control-plane @patches/controlplane.yaml \
  --config-patch @patches/piraeus/drbd.yaml
```

## Piraeus Operator installieren

```shell
kubectl apply --server-side -k "https://github.com/piraeusdatastore/piraeus-operator//config/default?ref=v2.7.1"
```

Abweichende Konfiguration für Talos

```shell
kubectl apply -f - <<'EOF'
apiVersion: piraeus.io/v1
kind: LinstorSatelliteConfiguration
metadata:
  name: talos-loader-override
spec:
  podTemplate:
    spec:
      initContainers:
        - name: drbd-shutdown-guard
          $patch: delete
        - name: drbd-module-loader
          $patch: delete
      volumes:
        - name: run-systemd-system
          $patch: delete
        - name: run-drbd-shutdown-guard
          $patch: delete
        - name: systemd-bus-socket
          $patch: delete
        - name: lib-modules
          $patch: delete
        - name: usr-src
          $patch: delete
        - name: etc-lvm-backup
          hostPath:
            path: /var/etc/lvm/backup
            type: DirectoryOrCreate
        - name: etc-lvm-archive
          hostPath:
            path: /var/etc/lvm/archive
            type: DirectoryOrCreate
EOF
```

Linstor Cluster erzeugen

```shell
kubectl apply -f - <<EOF
apiVersion: piraeus.io/v1
kind: LinstorCluster
metadata:
  name: linstorcluster
spec: {}
EOF
```

Überprüfen

```shell
tamaymueller@terra:~/Dokumente/projects/kubernetes/dev-environments/talos_linux/default$ kubectl -n piraeus-datastore exec deploy/linstor-controller -- linstor node list
+------------------------------------------------------------+
| Node      | NodeType  | Addresses                 | State  |
|============================================================|
| cp-01     | SATELLITE | 10.244.3.136:3366 (PLAIN) | Online |
| worker-01 | SATELLITE | 10.244.0.70:3366 (PLAIN)  | Online |
| worker-02 | SATELLITE | 10.244.2.121:3366 (PLAIN) | Online |
| worker-03 | SATELLITE | 10.244.1.4:3366 (PLAIN)   | Online |
+------------------------------------------------------------+
```

Storage Pool auf Dateisystem anlegen

```shell
kubectl apply -f - <<EOF
apiVersion: piraeus.io/v1
kind: LinstorSatelliteConfiguration
metadata:
  name: storage-pool
spec:
  storagePools:
    - name: pool1
      fileThinPool:
        directory: /var/lib/piraeus-datastore/pool1
EOF
```

Storage Pool in Linstor anzeigen

```shell
tamaymueller@terra:~/Dokumente/projects/kubernetes/dev-environments/talos_linux/default$ kubectl -n piraeus-datastore exec deploy/linstor-controller -- linstor storage-pool list
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| StoragePool          | Node      | Driver    | PoolName                         | FreeCapacity | TotalCapacity | CanSnapshots | State | SharedName                     |
|========================================================================================================================================================================|
| DfltDisklessStorPool | cp-01     | DISKLESS  |                                  |              |               | False        | Ok    | cp-01;DfltDisklessStorPool     |
| DfltDisklessStorPool | worker-01 | DISKLESS  |                                  |              |               | False        | Ok    | worker-01;DfltDisklessStorPool |
| DfltDisklessStorPool | worker-02 | DISKLESS  |                                  |              |               | False        | Ok    | worker-02;DfltDisklessStorPool |
| DfltDisklessStorPool | worker-03 | DISKLESS  |                                  |              |               | False        | Ok    | worker-03;DfltDisklessStorPool |
| pool1                | cp-01     | FILE_THIN | /var/lib/piraeus-datastore/pool1 |     5.84 GiB |      8.76 GiB | True         | Ok    | cp-01;pool1                    |
| pool1                | worker-01 | FILE_THIN | /var/lib/piraeus-datastore/pool1 |     6.20 GiB |      8.76 GiB | True         | Ok    | worker-01;pool1                |
| pool1                | worker-02 | FILE_THIN | /var/lib/piraeus-datastore/pool1 |     6.07 GiB |      8.76 GiB | True         | Ok    | worker-02;pool1                |
| pool1                | worker-03 | FILE_THIN | /var/lib/piraeus-datastore/pool1 |     5.89 GiB |      8.76 GiB | True         | Ok    | worker-03;pool1                |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Storageclass anlegen und mit PVC und Deployment testen

```shell
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: piraeus-storage
provisioner: linstor.csi.linbit.com
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  linstor.csi.linbit.com/storagePool: pool1
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-volume
spec:
  storageClassName: piraeus-storage
  resources:
    requests:
      storage: 1Gi
  accessModes:
    - ReadWriteOnce
EOF

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: web-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: web-server
    spec:
      containers:
        - name: web-server
          image: nginx
          volumeMounts:
            - mountPath: /usr/share/nginx/html
              name: data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: data-volume
EOF
```

PVC und Volume in Linstor anzeigen

```shell
tamaymueller@terra:~/Dokumente/projects/kubernetes/dev-environments/talos_linux/default$ kubectl get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                 VOLUMEATTRIBUTESCLASS   AGE
data-volume         Bound    pvc-ff181f80-5163-4c9f-9679-384e4b91784a   1Gi        RWO            piraeus-storage              <unset>                 1m

tamaymueller@terra:~/Dokumente/projects/kubernetes/dev-environments/talos_linux/default$ kubectl -n piraeus-datastore exec deploy/linstor-controller -- linstor resource list-volumes
+-------------------------------------------------------------------------------------------------------------------------------------+
| Node      | Resource                                 | StoragePool | VolNr | MinorNr | DeviceName    | Allocated | InUse |    State |
|=====================================================================================================================================|
| worker-01 | pvc-ff181f80-5163-4c9f-9679-384e4b91784a | pool1       |     0 |    1000 | /dev/drbd1000 |           | InUse | UpToDate |
+-------------------------------------------------------------------------------------------------------------------------------------+
```

## Replicated Storage

Dateien via DRBD auf mehreren Hosts replizieren.

* neue Storageclass anlegen, `linstor.csi.linbit.com/placementCount: "2"` legt fest, wieviele Replicas es sein sollen

```shell
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: piraeus-storage-replicated
provisioner: linstor.csi.linbit.com
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  linstor.csi.linbit.com/storagePool: pool1
  linstor.csi.linbit.com/placementCount: "2"
EOF
```

Wieder testen

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: replicated-volume
spec:
  storageClassName: piraeus-storage-replicated
  resources:
    requests:
      storage: 1Gi
  accessModes:
    - ReadWriteOnce
EOF

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: volume-logger
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: volume-logger
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: volume-logger
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: volume-logger
          image: busybox
          args:
            - sh
            - -c
            - |
              echo "Hello from \$HOSTNAME, running on \$NODENAME, started at \$(date)" >> /volume/hello
              # We use this to keep the Pod running
              tail -f /dev/null
          env:
            - name: NODENAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - mountPath: /volume
              name: replicated-volume
      volumes:
        - name: replicated-volume
          persistentVolumeClaim:
            claimName: replicated-volume
EOF

```shell
tamaymueller@terra:~/Dokumente/projects/kubernetes/dev-environments/talos_linux/default$ kubectl get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                 VOLUMEATTRIBUTESCLASS   AGE
replicated-volume   Bound    pvc-7f95f393-8fb3-4772-9dbc-a4598345957d   1Gi        RWO            piraeus-storage-replicated   <unset>                 47s
```

```shell
tamaymueller@terra:~/Dokumente/projects/kubernetes/dev-environments/talos_linux/default$ kubectl -n piraeus-datastore exec deploy/linstor-controller -- linstor resource list-volumes
+-------------------------------------------------------------------------------------------------------------------------------------------------+
| Node      | Resource                                 | StoragePool          | VolNr | MinorNr | DeviceName    | Allocated | InUse  |      State |
|=================================================================================================================================================|
| cp-01     | pvc-7f95f393-8fb3-4772-9dbc-a4598345957d | DfltDisklessStorPool |     0 |    1001 | /dev/drbd1001 |           | Unused | TieBreaker |
| worker-01 | pvc-7f95f393-8fb3-4772-9dbc-a4598345957d | pool1                |     0 |    1001 | /dev/drbd1001 |           | InUse  |   UpToDate |
| worker-02 | pvc-7f95f393-8fb3-4772-9dbc-a4598345957d | pool1                |     0 |    1001 | /dev/drbd1001 |           | Unused |   UpToDate |
+-------------------------------------------------------------------------------------------------------------------------------------------------+
```

## weitere Festplatten verwenden

Dokumentation zu Storagepool: <https://piraeus.io/docs/stable/reference/linstorsatelliteconfiguration/#specstoragepools>

* Disks zu Nodes hinzufügen
* 20GB /dev/vdb

```shell
tamaymueller@terra:~/Dokumente/projects/kubernetes/dev-environments/talos_linux/default$ talosctl -n 192.168.100.84 get disks
NODE             NAMESPACE   TYPE   ID      VERSION   SIZE     READ ONLY   TRANSPORT   ROTATIONAL   WWID   MODEL          SERIAL
192.168.100.84   runtime     Disk   loop0   1         143 kB   true                                                       
192.168.100.84   runtime     Disk   loop1   1         4.1 kB   true                                                       
192.168.100.84   runtime     Disk   loop2   1         573 kB   true                                                       
192.168.100.84   runtime     Disk   loop3   1         74 MB    true                                                       
192.168.100.84   runtime     Disk   loop4   1         1.1 GB   false                   true                               
192.168.100.84   runtime     Disk   sr0     1         105 MB   false       ata                             QEMU DVD-ROM   
192.168.100.84   runtime     Disk   vda     1         11 GB    false       virtio      true                               
192.168.100.84   runtime     Disk   vdb     1         22 GB    false       virtio      true                               
```

* Storagepool von linstorsatelliteconfigurations.piraeus.io erweitern

```yaml
apiVersion: piraeus.io/v1
kind: LinstorSatelliteConfiguration
metadata:
  name: storage-pool
spec:
  storagePools:
  - fileThinPool:
      directory: /var/lib/piraeus-datastore/pool1
    name: pool1
  - lvmPool:
      volumeGroup: vg2
    name: vg2
    properties:
    - name: StorDriver/LvcreateOptions
      value: -i 2
    source:
      hostDevices:
      - /dev/vdb
```

* Fertig

```shell
tamaymueller@terra:~/Dokumente/projects/kubernetes/dev-environments/talos_linux/default$ kubectl -n piraeus-datastore exec deploy/linstor-controller -- linstor storage-pool list
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| StoragePool          | Node      | Driver    | PoolName                         | FreeCapacity | TotalCapacity | CanSnapshots | State | SharedName                     |
|========================================================================================================================================================================|
| DfltDisklessStorPool | cp-01     | DISKLESS  |                                  |              |               | False        | Ok    | cp-01;DfltDisklessStorPool     |
| DfltDisklessStorPool | worker-01 | DISKLESS  |                                  |              |               | False        | Ok    | worker-01;DfltDisklessStorPool |
| DfltDisklessStorPool | worker-02 | DISKLESS  |                                  |              |               | False        | Ok    | worker-02;DfltDisklessStorPool |
| DfltDisklessStorPool | worker-03 | DISKLESS  |                                  |              |               | False        | Ok    | worker-03;DfltDisklessStorPool |
| pool1                | cp-01     | FILE_THIN | /var/lib/piraeus-datastore/pool1 |     5.58 GiB |      8.76 GiB | True         | Ok    | cp-01;pool1                    |
| pool1                | worker-01 | FILE_THIN | /var/lib/piraeus-datastore/pool1 |     5.94 GiB |      8.76 GiB | True         | Ok    | worker-01;pool1                |
| pool1                | worker-02 | FILE_THIN | /var/lib/piraeus-datastore/pool1 |     6.07 GiB |      8.76 GiB | True         | Ok    | worker-02;pool1                |
| pool1                | worker-03 | FILE_THIN | /var/lib/piraeus-datastore/pool1 |     5.88 GiB |      8.76 GiB | True         | Ok    | worker-03;pool1                |
| vg2                  | cp-01     | LVM       | vg2                              |              |               | False        | Ok    | cp-01;vg2                      |
| vg2                  | worker-01 | LVM       | vg2                              |    20.00 GiB |     20.00 GiB | False        | Ok    | worker-01;vg2                  |
| vg2                  | worker-02 | LVM       | vg2                              |    20.00 GiB |     20.00 GiB | False        | Ok    | worker-02;vg2                  |
| vg2                  | worker-03 | LVM       | vg2                              |    20.00 GiB |     20.00 GiB | False        | Ok    | worker-03;vg2                  |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
