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

## Encryption

All files in the `config/` directory and the `secrets.yaml` file are encrypted using [SOPS](https://github.com/getsops/sops) to ensure sensitive data is protected. Use SOPS to decrypt and edit these files as needed.

## Prerequisites

- A machine or VM image that will run Talos OS on each node.
- `talosctl` - Talos control tool. See https://www.talos.dev/ for installation instructions.
- `kubectl` - Kubernetes CLI for interacting with the cluster once control plane is available.
- `sops` - tool for encrypting and decrypting configuration files. See https://github.com/getsops/sops for installation.
- `age` - encryption tool used by SOPS for key management. See https://github.com/FiloSottile/age for installation.
- `~/.config/sops/age/keys.txt` - your age key file, required by SOPS for encryption and decryption

## Quick start (high level)

1. Create patches to override `config/controlplane.yaml` and `config/worker.yaml` as needed to match your cluster settings (IPs, network, etc.), do not edit these files directly.
2. Create or update node-specific manifests under `nodes/`.
3. Keep any sensitive values in `secrets.yaml`. Make sure to encrypt the file before comitting
4. Use `talosctl` to apply configuration to machines and bootstrap the control plane.

## Handling Secrets

Sensitive files are encrypted with a combination of [SOPS](https://github.com/getsops/sops) and [age](https://github.com/FiloSottile/age).

Make sure the packages are installed before proceeding.

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
    talosctl apply-config --insecure --nodes 192.168.10.121 --file config/controlplane.yaml
    ```
3. Patch the talos-mw1 node with it's network settings, and some custome cluster settings

    ```
    talosctl patch mc --talosconfig _out/talosconfig -e 192.168.10.121 -n 192.168.10.121 --patch @nodes/talos-mw1.yaml
    ```
4. Add the new node to the talos cluster endpoint (control planes) and nodes list

    ```
    talosctl config endpoint 192.168.10.40
    talosctl config node 192.168.10.40
    ```
5. Bootstrap the cluster

    ```
    talosctl bootstrap -n 10.0.50.161
    ```
6. Grab the kubeconfig and save it to .kube
    ```
    talosctl kubeconfig -n 10.0.50.161 ~/.kube/config
    ```
7. Wait a few minutes for the k8s cluster to be online, then verify the node
    ```
    kubectl get node -o wide
    ```