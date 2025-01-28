+++
date = '2025-01-27T23:30:44-07:00'
draft = true
title = 'Talos Longhorn'
+++
# Installing Longhorn on Talos Linux: A Step-by-Step Guide

Longhorn provides cloud-native distributed block storage for Kubernetes. This guide walks through setting up Longhorn on a Talos Linux cluster, addressing common pitfalls and requirements.

## Prerequisites
- A running Talos Linux cluster
- `kubectl` configured to access your cluster
- `talosctl` configured for your cluster

### Required Talos Extensions
Your Talos Linux cluster must have the following system extensions installed:
- `iscsi-tools`
- `linux-utils`

Add these to your Talos Linux machine configuration:
```yaml
customization:
  systemExtensions:
    officialExtensions:
      - image: ghcr.io/siderolabs/iscsi-tools
      - image: ghcr.io/siderolabs/util-linux-tools
```

## 1. Configure Talos Worker Nodes

Longhorn requires specific kernel modules to be loaded on worker nodes. Update your Talos worker configuration to include these modules:

```yaml
machine:
  kernel:
    modules:
      - name: nbd
      - name: iscsi_tcp
      - name: iscsi_generic
      - name: configfs
```

Apply this configuration to your worker nodes:

```bash
talosctl apply-config --nodes <worker-ip> --file worker.yaml
```

The nodes will reboot to apply these changes. Verify the modules are loaded:

```bash
talosctl ls /lib/modules/$(talosctl uname -r)/kernel/drivers/target/
```

## 2. Create Namespace with Privileged Access

Longhorn requires privileged access to manage storage. Create a namespace with appropriate Pod Security Admission (PSA) labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: longhorn-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

Apply this configuration:

```bash
kubectl apply -f longhorn-namespace.yaml
```

## 3. Install Longhorn

Add the Longhorn Helm repository and install Longhorn with ArgoCD:

Login to ArgoCD:
```bash
argocd login --core

```
Set K8s context namespace to ArgoCD:
```bash
kubectl config set-context --current --namespace=argocd
```
Create the Longhorn Application custom resource:
```bash
cat > longhorn-application.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn
  namespace: argocd
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
  project: default
  sources:
    - chart: longhorn
      repoURL: https://charts.longhorn.io/
      targetRevision: v1.8.0 # Replace with the Longhorn version you'd like to install or upgrade to
      helm:
        values: |
          preUpgradeChecker:
            jobEnabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: longhorn-system
EOF
kubectl apply -f longhorn-application.yaml
```
Now deploy Longhorn:
```bash
argocd app sync longhorn
```
Verify deployment:
```bash
kubectl -n longhorn-system get pod
```

## 4. Configure Default Backup Target

Longhorn requires a default backup target configuration, even if you're not planning to use backups. Create a backup target:

```yaml
apiVersion: longhorn.io/v1beta2
kind: BackupTarget
metadata:
  name: default
  namespace: longhorn-system
spec:
  backupTargetURL: ""
```

Apply the backup target:

```bash
kubectl apply -f backup-target.yaml
```

## 5. Verify Installation

Check that all Longhorn components are running:

```bash
kubectl get pods -n longhorn-system
```

Verify the storage class is available:

```bash
kubectl get storageclass
```

You should see `longhorn` listed as a storage class.

## Common Issues and Troubleshooting

### Pod Security Standards
If you see errors about privileged containers or hostPath volumes being forbidden, ensure your namespace has the correct PSA labels as shown in step 2.

### Kernel Modules
If Longhorn pods fail to start, verify the required kernel modules are loaded:

```bash
talosctl exec -n <worker-ip> -- lsmod | grep -E 'nbd|iscsi_tcp|iscsi_generic'
```

### Volume Creation Issues
If PVCs remain in pending state with "backuptarget not found" errors, ensure you've created the default backup target as shown in step 4.

## Next Steps

With Longhorn installed, you can:
- Access the Longhorn UI through a port-forward or ingress
- Configure backup settings if needed
- Set Longhorn as your default storage class
- Start creating PersistentVolumeClaims using the Longhorn storage class

## Conclusion

You now have a functioning Longhorn installation on your Talos Linux cluster. Longhorn provides reliable distributed storage for your applications, with features like snapshots, backups, and volume replication available when needed.

Remember to keep your Talos configurations backed up, especially the kernel module settings, as they'll be needed when adding new nodes to your cluster.
