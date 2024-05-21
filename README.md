# Enabling NFS-common package on TKG-m worker nodes for NFS mounts in Kubernetes

## Pre-reqs
* TKGm management cluster is created (tested with 2.5)
* [ytt installed](https://carvel.dev/ytt/docs/v0.48.0/install/)
* kubectl installed and set to management cluster context
* Tanzu CLI
* Git CLI

## Create Custom Cluster Class
This process involves creating a custom cluster class that allows for deploying worker nodes in a workload cluster that have multiple network interfaces. Creating custom clusterclasses is roughly documented [here](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.5/using-tkg/workload-clusters-cclass.html).

1. In Tanzu Kubernetes Grid 2.3.0 and later, after you deploy a management cluster, you can find the default ClusterClass manifest in the ~/.config/tanzu/tkg/clusterclassconfigs folder.

```bash
cp ~/.config/tanzu/tkg/clusterclassconfigs/tkg-vsphere-default-v1.2.0.yaml .
```

2. To customize your ClusterClass manifest, you create ytt overlay files alongside the manifest. Clone this repo
```bash
git clone https://github.com/logankimmel/tkgm-nfs-common.git

```

Use the default ClusterClass manifest from step 1 to generate the base ClusterClass:
```bash
ytt -f tkg-vsphere-default-v1.2.0.yaml -f custom/overlays/filter.yaml > default_cc.yaml
```

Generate the custom ClusterClass:
```bash
ytt -f default_cc.yaml -f custom/ > custom_cc.yaml
```

Install the custom clusterClass in the Management cluster:
```bash
kubectl apply -f custom_cc.yaml
```

You should see the following output when you run `kubectl get clusterclasses`:
```
NAME                                  AGE
tkg-vsphere-default-v1.2.0            21h
tkg-vsphere-default-v1.2.0-extended   20h
```
We have now created an "extended" cluster class that accepts a new variable: `nfsCommon`

## Deploy new workload cluster that uses the custom clusterclass

1. Locate your configuration file you used for creating the management cluster (these are stored in: `~/.config/tanzu/tkg/clusterconfigs`). More info on the config file can be found [here](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.5/using-tkg/workload-clusters-deploy.html#prerequisites-0)

2. Copy the config file to your working director:
```bash
cp ~/.config/tanzu/tkg/clusterconfigs\{config_file}.yaml ./workload-1.yaml
```

3. Generate the custom workload cluster manifest
```bash
tanzu cluster create --file workload-1.yaml --dry-run > default_cluster.yaml
```

Using the overlay, create the custom manifest:
```bash
ytt -f default_cluster.yaml -f cluster_overlay.yaml > custom_cluster.yaml
```

4. Deploy
```bash
tanzu cluster create -f custom_cluster.yaml
```
