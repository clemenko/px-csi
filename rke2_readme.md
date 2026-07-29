# PX + Harvester

This is a quick guide for adding Portworx CSI ( PX-CSI 26.1.0 ) to RKE2.

## add multipathd

to all the harvester nodes

```bash
dnf install -y wget nfs-utils cryptsetup iscsi-initiator-utils epel-release iptables-services iptables-utils device-mapper-multipath
systemctl enable --now iscsid

cat << EOF >> /etc/multipath.conf
devices {
    device {
        vendor                      "NVME"
        product                     "Pure Storage FlashArray"
        path_selector               "queue-length 0"
        path_grouping_policy        group_by_prio
        prio                        ana
        failback                    immediate
        fast_io_fail_tmo            10
        user_friendly_names         no
        no_path_retry               0
        features                    0
        dev_loss_tmo                60
    }
    device {
        vendor                   "PURE"
        product                  "FlashArray"
        path_selector            "service-time 0"
        hardware_handler         "1 alua"
        path_grouping_policy     group_by_prio
        prio                     alua
        failback                 immediate
        path_checker             tur
        fast_io_fail_tmo         10
        user_friendly_names      no
        no_path_retry            0
        features                 0
        dev_loss_tmo             600
    }
}
blacklist_exceptions {
    property "(SCSI_IDENT_|ID_WWN)"
}
blacklist {
    devnode "^pxd[0-9]*"
    devnode "^pxd*"
    device {
        vendor "VMware"
        product "Virtual disk"
    }
    device {
        vendor "IET"
        product "VIRTUAL-DISK"
    }
}
EOF
```

## add rke2

add rke2 and move the calico prometheus port from 9091 to 9891

```bash
dnf install kernel-modules-extra-$(uname -r) -y ; modprobe ip_tables && curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=stable sh - && systemctl enable --now rke2-server.service && echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml PATH=$PATH:/usr/local/bin/:/var/lib/rancher/rke2/bin/" >> ~/.bashrc && source ~/.bashrc && curl -s https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

kubectl patch felixconfiguration default --type=merge -p '{"spec":{"prometheusMetricsPort":9891}}'

# validate 
ss -tln | grep 91
```

## add portworx stuff

Here we are going to install the CSI. Note the API token for a "storage admin" user. Here are the docs : https://docs.portworx.com/portworx-enterprise/platform/kubernetes/flasharray/install/install-flasharray/install-flasharray-cd-da

```bash
# get latest version of PX-CSI
PX_CSI_VER=$(curl -sL https://dzver.rfed.io/json | jq -r .portworx)

# create namespace
kubectl create ns portworx

# create and add secret
cat << EOF > pure.json 
{
    "FlashArrays": [
        {
            "MgmtEndPoint": "192.168.1.11",
            "APIToken": "934f95b6-6d1d-ee91-d210-6ed9bce13ad1",
            "NFSEndPoint": "192.168.1.8"
        }
    ]
}
EOF
kubectl create secret generic px-pure-secret -n portworx --from-file=pure.json=pure.json

# apply operator yaml
kubectl apply -f 'https://install.portworx.com/'$PX_CSI_VER'?comp=pxoperator&oem=px-csi&kbver=1.35.6&ns=portworx'

# add annotation of "portworx.io/health-check: "skip" " for running on a single node

#  If you want nvme-tcp change the value: "NVMEOF-TCP" "FC"

# or FA NFS : https://docs.portworx.com/portworx-csi/provision-storage/dynamic-provisioning/flasharray-file-services

kubectl apply -n portworx  -f - << EOF 
kind: StorageCluster
apiVersion: core.libopenstorage.org/v1
metadata:
  name: px-cluster
  namespace: portworx
  annotations:
    portworx.io/misc-args: "--oem px-csi"
    #portworx.io/health-check: "skip"
spec:
  image: portworx/px-pure-csi-driver:$PX_CSI_VER
  imagePullPolicy: IfNotPresent
  csi:
    enabled: true
  monitoring:
    telemetry:
      enabled: false
    prometheus:
      enabled: false
      exportMetrics: true
  env:
  - name: PURE_FLASHARRAY_SAN_TYPE
    value: "ISCSI"
EOF
```

## update csi settings

Update the Harvester CSI settings - https://docs.harvesterhci.io/v1.8/advanced/csidriver#configure-harvester-cluster.

## add image

```bash
kubectl apply -f - << EOF 
apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineImage
metadata:
  name: fa-rocky
  namespace: default
  annotations:
    harvesterhci.io/storageClassName: px-fa-direct-access
spec:
  backend: cdi
  displayName: fa-rocky
  retry: 3
  sourceType: download
  targetStorageClassName: px-fa-direct-access
  url: https://dl.rockylinux.org/pub/rocky/10/images/x86_64/Rocky-10-GenericCloud-Base.latest.x86_64.qcow2
EOF
```

## or a simple pvc test

```bash
kubectl apply -n portworx  -f - << EOF 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: demopvc2
spec:
  storageClassName: px-fa-direct-access
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10234Mi
EOF
```

Success.

## optional storage class

from https://docs.portworx.com/portworx-csi/reference/storage-class#storageclass

```bash
kubectl apply -n portworx  -f - << EOF 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: px-fb-direct-access-nfsv4-hardquota
mountOptions:
- nfsvers=4.1
- tcp
allowVolumeExpansion: false
parameters:
  backend: pure_file
  pure_export_rules: '*(rw)'
provisioner: pxd.portworx.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF

# or 

kubectl apply -n portworx  -f - << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: px-fa-direct-access
allowVolumeExpansion: true
parameters:
  backend: pure_block #pure_fa_file
provisioner: pxd.portworx.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
```
