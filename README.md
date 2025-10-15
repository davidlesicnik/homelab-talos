# Talos cluster configuration

This repo holds configuration files and node manifests used to provision my Talos-based Kubernetes cluster.

## Repository layout
 
- `.sops.yaml` - SOPS configuration for file encryption
- `config/` - Talos cluster config files. These files aren't to be changed directly; create patches instead
    - `controlplane.encrypted.yaml` - encrypted control plane config (SOPS)
    - `worker.encrypted.yaml` - encrypted worker config (SOPS)
    - `talosconfig.encrypted` - encrypted Talosctl config (SOPS)
- `nodes/` - node-specific manifests
    - `talos-mw1.yaml` - node manifest for a master-worker node
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
    ```
3. Patch the nodes with the apropriate patches (note that the endpoint stays the same), use the IP of the above endpoint for -e

    ```
    talosctl patch mc --talosconfig config/talosconfig -e 192.168.10.126 -n 192.168.10.126 --patch @patches/nodes/talos-m1.yaml --patch @patches/cluster-patches.yaml --patch @patches/nodes/all-nodes.yaml 
    talosctl patch mc --talosconfig config/talosconfig -e 192.168.10.126 -n 192.168.10.127 --patch @patches/nodes/talos-mw1.yaml --patch @patches/cluster-patches.yaml --patch @patches/nodes/all-nodes.yaml 
    talosctl patch mc --talosconfig config/talosconfig -e 192.168.10.126 -n 192.168.10.128 --patch @patches/nodes/talos-mw2.yaml --patch @patches/cluster-patches.yaml --patch @patches/nodes/all-nodes.yaml 
    ```
4. Add the new node to the talos cluster endpoint (control planes) and nodes list

    ```
    talosctl config endpoint 192.168.10.41 192.168.10.50 192.168.10.51
    talosctl config node 192.168.10.41 192.168.10.50 192.168.10.51
    ```
5. Bootstrap the cluster

    ```
    talosctl bootstrap -n 192.168.10.41
    ```
6. Grab the kubeconfig and save it to .kube
    ```
    talosctl kubeconfig -n 192.168.10.41 ~/.kube/config
    ```
7. Wait a few minutes for the k8s cluster to be online, then verify the node
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