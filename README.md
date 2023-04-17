# Dell Wyse Talos Homelab

**Note:** this can all be accomplished through terraform as well but that path generally means you're bootstrapping each node/machine all at once while this approach is a bit more flextable in that it allows for us to bootstrap a single node at a time.


This guide assumes you have:
* 4 wyse machine nodes and that they're all control planes
* you're using this guides outlined IPs/Host Names

but you may use different machines such as raspberry pi's and change the individual configs using the same patches to make a control plane a worker node. You may also change the host names and IPs as you see fit.


## Machines
```shell
# IPs
export WYSE_1_IP=192.168.1.200
export WYSE_2_IP=192.168.1.201
export WYSE_3_IP=192.168.1.202
export WYSE_4_IP=192.168.1.203

# Host Names
export WYSE_1_HN=wyse-1
export WYSE_2_HN=wyse-2
export WYSE_3_HN=wyse-3
export WYSE_4_HN=wyse-4

# Patch Files Dir
export PATCH_FILE_DIR="$(pwd)/patch_files"

# Patch Files
export WYSE_1_PF="${PATCH_FILE_DIR}/${WYSE_1_HN}-patch.yaml"
export WYSE_2_PF="${PATCH_FILE_DIR}/${WYSE_2_HN}-patch.yaml"
export WYSE_3_PF="${PATCH_FILE_DIR}/${WYSE_3_HN}-patch.yaml"
export WYSE_4_PF="${PATCH_FILE_DIR}/${WYSE_4_HN}-patch.yaml"
```

1. Follow the [quickstart](https://www.talos.dev/latest/introduction/getting-started/)
    * Outcomes:
      * `talosctl`
      * `kubectl`
      * `secrets.yaml`
      * Configs (`controlplane.yaml`, `worker.yaml`, `talosconfig`) generated from secrets + endpoint (`https://$WYSE_1_IP` should be the cluster endpoint)
        * Ensure the endpoint above is listed under the `talosconfig` in `contexts.<cluster name>.endpoints`
      * Update your `talosconfig` file (with new endpoints) and merge it: `talosctl config merge ./talosconfig`

2. Patch the base config(s):
   * Base patch: 
```shell
talosctl machineconfig patch controlplane.yaml --patch @"$PATCH_FILE_DIR/base-patch.yaml" -o controlplane-patched.yaml
talosctl machineconfig patch worker.yaml --patch @"$PATCH_FILE_DIR/base-patch.yaml" -o worker-patched.yaml
```
   * Machine Specific(s) patch:
```shell
talosctl machineconfig patch controlplane-patched.yaml --patch @"$WYSE_1_PF" -o "$WYSE_1_HN.yaml"
talosctl machineconfig patch controlplane-patched.yaml --patch @"$WYSE_2_PF" -o "$WYSE_2_HN.yaml"
talosctl machineconfig patch controlplane-patched.yaml --patch @"$WYSE_3_PF" -o "$WYSE_3_HN.yaml"
talosctl machineconfig patch controlplane-patched.yaml --patch @"$WYSE_4_PF" -o "$WYSE_4_HN.yaml"
```

3. Validate the config files:
```shell
talosctl validate --mode metal --config "$WYSE_1_HN.yaml"
talosctl validate --mode metal --config "$WYSE_2_HN.yaml"
talosctl validate --mode metal --config "$WYSE_3_HN.yaml"
talosctl validate --mode metal --config "$WYSE_4_HN.yaml"
```

4. Apply the primary control plane node patched config:
```shell
talosctl apply-config --insecure --nodes $WYSE_1_IP --file $WYSE_1_HN.yaml
```

5. Bootstrap the primary control plane node:
```shell
talosctl bootstrap --nodes $WYSE_1_HN
```

6. Apply the remaining control plane node configs:
```shell
talosctl apply-config --insecure --nodes $WYSE_2_IP --file $WYSE_2_HN.yaml
talosctl apply-config --insecure --nodes $WYSE_3_IP --file $WYSE_3_HN.yaml
talosctl apply-config --insecure --nodes $WYSE_4_IP --file $WYSE_4_HN.yaml
```

7. Initial backup of the etcd database:
```shell
talosctl -n $WYSE_1_HN etcd snapshot db.snapshot
```

8. Get your kubeconfig to access your kubernetes cluster:
```shell
talosctl kubeconfig --merge --nodes $WYSE_1_HN
```

9. Ensure your nodes appear in `kubectl` (may take a few minutes after configuring and the status to be `ready`)
```shell
kubectl get nodes
```

# Hardware (I'm using)
* 4 Dell Wyse 5070 J4105 with 8GB ram
  * These are easier to find, cheaper and more powerful with more CPU arch use-cases than rasperry pis at the time of writing this
  * You can find these for USD $20-$40 each on ebay (I got all 4 of mine for USD $20-$25 each)
* 4 128 GB M.2 SSDs for the wyse machines (you don't NEED this much space, but it doesn't hurt, and it's cheap)
  * USD $10-$15 each from amazon
* 4 Power adapters for the wyse machines
  * Generic ones lower CPU speed but not by much
  * USD $10-$15 each from ebay
* A cheap ethernet
  * [Talos doesn't support WIFI](https://github.com/siderolabs/talos/discussions/6911) - which is a shame.
  * I use a USD $16 TP-Link switch off amazon
* Ethernet cables
  * I built my own, but you can get some cheap cat5/cat6 patch cables from amazon

# Notes:
* Backup the following files (outside of git) in multiple locations (flash drive, ftp server, nas, etc)
  * secrets.yaml
  * talosconfig