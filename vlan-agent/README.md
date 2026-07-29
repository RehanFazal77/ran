# vlan-agent

Automates the node-side VLAN sub-interface creation that the
[Nephio docs](https://docs.nephio.org/docs/guides/user-guides/usecase-user-guides/exercise-1-free5gc/)
still require to be done **manually** on every worker node
(`test-infra/e2e/provision/hacks/vlan-interfaces.sh` — the docs
themselves say *"this will be automated in a future release"*).

## Why this is needed

Nephio's native VLAN-based networking flow:

```
Interface (attachmentType: vlan)
  └── interface-fn creates a VLANClaim
        └── vlan-specializer allocates vlanID from the VLANIndex
              └── nad-fn renders the NAD with master: <masterInterface>.<vlanID>
```

The macvlan CNI can only attach a pod to `<masterInterface>.<vlanID>`
(e.g. `ens4f0.6`) if that VLAN sub-interface **exists on the node**.
Kubernetes does not create it — upstream Nephio leaves this to a manual
script with a hardcoded interface and VLAN range.

## What the agent does

A `kube-system` DaemonSet on every node that, every 30 seconds:

1. Lists all `NetworkAttachmentDefinitions` in the cluster (read-only RBAC).
2. Parses each NAD's `spec.config` JSON and collects every `master`
   value of the form `<nic>.<vlanID>`.
3. Creates any missing VLAN sub-interfaces and brings them up:

   ```sh
   ip link add link ens4f0 name ens4f0.6 type vlan id 6
   ip link set ens4f0.6 up
   ```

Creation is idempotent, and the agent reconciles continuously, so VLAN
links survive node reboots and NADs added later (e.g. a new NF package
deployed via Config Sync).

## Optional: docs-parity static range

To also pre-create a fixed range exactly like the Nephio docs script
(`eth1.2` … `eth1.6`), set the optional setters:

```yaml
# In a PackageVariant pipeline
- image: ghcr.io/kptdev/krm-functions-catalog/apply-setters:v0.2
  configMap:
    cluster-name: ran
    master-interface: eth1
    vlan-range: "2 3 4 5 6"
```

Leave both empty (the default) to rely purely on dynamic NAD discovery.

## Deployment order

Deploy together with `networking/multus-cni`, **before** any NF
workloads, whenever the NF blueprints use `networking-mode: vlan`.
In `networking-mode: nic` (direct physical NIC in the NAD master) the
agent finds no `<nic>.<vlanID>` masters and is a harmless no-op.
