# px-csi

## 1. Title and Purpose

This repo is a collection of field notes and step-by-step guides for deploying **Portworx CSI (px-csi)** with Pure Storage FlashArray backends across different Kubernetes distributions and environments — including air-gapped installs, Harvester HCI, and RKE2. The goal is to have a working, repeatable reference for standing up PX-CSI, configuring multipathd, and wiring up storage classes/StorageCluster objects for real deployments.

## 2. Upstream Portworx CSI (px-csi) Docs

- Portworx CSI install (airgapped): https://docs.portworx.com/portworx-csi/install/airgapped-install#configure-portworx-version-manifest
- Portworx + FlashArray install guide: https://docs.portworx.com/portworx-enterprise/platform/kubernetes/flasharray/install/install-flasharray/install-flasharray-cd-da
- Storage class reference: https://docs.portworx.com/portworx-csi/reference/storage-class#storageclass
- FlashArray dynamic provisioning (NFS/file services): https://docs.portworx.com/portworx-csi/provision-storage/dynamic-provisioning/flasharray-file-services
- Version lookup helper: https://dzver.rfed.io

## 3. Readmes in this Repo

- [airgap_readme.md](airgap_readme.md) — Guide for air-gapping the Pure/Portworx bits using [Hauler](https://docs.hauler.dev/docs/intro). Covers generating a Hauler manifest, syncing an offline store, transferring the tarball across the air gap, serving files/images from the isolated side, and installing PX-CSI on Harvester from that local registry.
- [harvester_readme.md](harvester_readme.md) — Quick guide for adding Portworx CSI to a Harvester cluster: installing Harvester, running `harvester_setup.sh`, configuring multipathd, installing the PX operator/StorageCluster pointed at a FlashArray, updating Harvester's CSI settings, and adding a test image/PVC.
- [rke2_readme.md](rke2_readme.md) — Same Portworx CSI install flow as the Harvester guide, but targeted at a bare RKE2 cluster: installing multipath/iscsi packages, installing RKE2 itself (with the Calico Prometheus port moved off the default), then installing the PX operator, StorageCluster, and storage classes.
