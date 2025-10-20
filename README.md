# Talos cluster configuration

This repo holds configuration files and node manifests used to provision my Talos-based Kubernetes cluster.

## Repository layout
 
- `.sops.yaml` - SOPS configuration for file encryption
- `config/` - Talos cluster config files. These files aren't to be changed directly; create patches instead
    - `controlplane.encrypted.yaml` - encrypted control plane config (SOPS)
    - `worker.encrypted.yaml` - encrypted worker config (SOPS)
    - `talosconfig.encrypted` - encrypted Talosctl config (SOPS)
- `patches/` - patch manifests
    - `nodes/` - node-specific patches
        - `talos-m1.yaml` - node manifest for a master only node
        - `talos-mw1.yaml` - node manifest for a master-worker node
        - `talos-mw2.yaml` - node manifest for a master-worker node
- `secrets.encrypted.yaml` - encrypted secrets file (SOPS)

## Prerequisites

- A machine or VM image that will run Talos OS on each node.
- `talosctl` - Talos control tool. See https://www.talos.dev/ for installation instructions.
- `kubectl` - Kubernetes CLI for interacting with the cluster once control plane is available.
- `sops` - tool for encrypting and decrypting configuration files. See https://github.com/getsops/sops for installation.
- `age` - encryption tool used by SOPS for key management. See https://github.com/FiloSottile/age for installation.
- `~/.config/sops/age/keys.txt` - your age key file, required by SOPS for encryption and decryption
> **Note:** The files in the `config/` folder are prerequisites and are generated using `talosctl gen config`. Ensure you have created these files before proceeding with cluster setup.

## Quick start (high level)

1. Create patches to override `config/controlplane.yaml` and `config/worker.yaml` as needed to match your cluster settings (IPs, network, etc.), do not edit these files directly.
2. Create or update node-specific manifests under `nodes/`.
3. Keep any sensitive values in `secrets.yaml`. Make sure to encrypt the file before comitting
4. Use `talosctl` to apply configuration to machines and bootstrap the control plane.

## Handling Secrets

- **Encryption:**  
    Use the following command to encrypt the files with SOPS: (secrets file as an example)
    ```
    sops --encrypt secrets.yaml > secrets.encrypted.yaml
    ```

- **Decryption:**  
    To decrypt the files and restore the original, use:
    ```
    sops --decrypt secrets.encrypted.yaml > secrets.yaml
    ```

## Bootstraping The Cluster

1. Decrypt the encrypted files
2. Run the following command to apply the control plane configuration (replace the IP with your node's address):

    ```
    talosctl apply-config --insecure --nodes 192.168.10.126 --file config/controlplane.yaml
    talosctl apply-config --insecure --nodes 192.168.10.127 --file config/controlplane.yaml
    talosctl apply-config --insecure --nodes 192.168.10.128 --file config/controlplane.yaml
    ```
3. Wait a few minutes for them to apply and reboot then patch the first node with the apropriate patches

    ```
    talosctl patch mc --talosconfig config/talosconfig -e 192.168.10.126 -n 192.168.10.126 --patch @patches/nodes/talos-m1.yaml --patch @patches/cluster-patches.yaml --patch @patches/nodes/all-nodes.yaml 
    ```

4. Once the first node is up, patch the other two. (note that the endpoint stays the same), use the IP of the above endpoint for -e
    ```
    talosctl patch mc --talosconfig config/talosconfig -e 192.168.10.41 -n 192.168.10.127 --patch @patches/nodes/talos-mw1.yaml --patch @patches/nodes/all-nodes.yaml 
    talosctl patch mc --talosconfig config/talosconfig -e 192.168.10.41 -n 192.168.10.128 --patch @patches/nodes/talos-mw2.yaml  --patch @patches/nodes/all-nodes.yaml 
    ```
5. Add the new node to the talos cluster endpoint (control planes) and nodes list

    ```
    talosctl config endpoint 192.168.10.41 192.168.10.50 192.168.10.51
    talosctl config node 192.168.10.41 192.168.10.50 192.168.10.51
    ```
6. Upgrade nodes with iSCSI, QEMU and disk utils extensions (check version!)
   If unsure, genereate a new schematic ID: https://factory.talos.dev/
    ```
    TALOS_IMAGE="factory.talos.dev/nocloud-installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.11.3"
    talosctl upgrade --nodes 192.168.10.41 --image $TALOS_IMAGE
    talosctl health --nodes 192.168.10.41
    ```
7. Once the node is healthly, repeat the process for the other two nodes
    ```
    TALOS_IMAGE="factory.talos.dev/nocloud-installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.11.3"
    talosctl upgrade --nodes 192.168.10.50 --image $TALOS_IMAGE
    talosctl health --nodes 192.168.10.50
    ```
    Wait for it to become health
        ```
    TALOS_IMAGE="factory.talos.dev/nocloud-installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.11.3"
    talosctl upgrade --nodes 192.168.10.51 --image $TALOS_IMAGE
    talosctl health --nodes 192.168.10.51
    ```
8. Bootstrap the cluster

    ```
    talosctl bootstrap -n 192.168.10.41
    ```
9. Grab the kubeconfig and save it to .kube
    ```
    talosctl kubeconfig -n 192.168.10.41 ~/.kube/config
    ```
10. Wait a few minutes for the k8s cluster to be online, then verify the node
    ```
    kubectl get node -o wide
    ```

## Getting talosctl to work on a new workstation

1. Decrypt the encrypted files
2. Add the node to the talos cluster endpoint (control planes) and nodes list

    ```
    talosctl config endpoint 192.168.10.41 192.168.10.50 192.168.10.51
    talosctl config node 192.168.10.41 192.168.10.50 192.168.10.51
    ```
3. Grab the kube config
    ```
    talosctl kubeconfig -n 192.168.10.41
    ```