# kubectl-plugins

A collection of `kubectl` plugins. They are plain shell scripts with no
dependencies beyond `kubectl` and `bash`.

| Plugin                        | Description                                                  |
| ----------------------------- | ------------------------------------------------------------ |
| [`kubectl ssh`](#kubectl-ssh) | Shell access to any Kubernetes node or pod, on any provider. |

---

## kubectl ssh

Open a shell on a Kubernetes **node** without an SSH server, SSH keys or cloud
credentials. It works the same way on kind, minikube, Docker Desktop, GKE, EKS,
AKS and bare metal. The same plugin also opens shells in **pods**, including
pods built from distroless images that have no shell.

```console
$ kubectl ssh node worker-1
Created pod/ssh-node-8hcm8 on node worker-1
Waiting for pod/ssh-node-8hcm8 to start...
root@worker-1:/# systemctl is-active kubelet
active
```

### Features

- **Real host access.** It enters the host's mount, PID, network, IPC and UTS
  namespaces, so `systemctl`, `journalctl`, `ps`, `crictl` and `ip` see the node
  itself, not a container.
- **Interactive or scripted.** Run a shell, run a single command, pipe data in
  or redirect output to a local file. Exit codes are passed through.
- **Fan-out.** Run a command on every node, or on the nodes that match a label
  selector.
- **Interactive picker.** Leave out the node or pod name and choose from a list
  (uses [`fzf`](https://github.com/junegunn/fzf) if it is installed).
- **Pods and workloads.** `kubectl ssh pod/x`, `deploy/x`, `sts/x` and so on.
  It picks bash, ash or sh, whichever the container has.
- **Distroless support.** `--debug` attaches an ephemeral debug container that
  can see the target container's processes and filesystem.
- **Cleans up after itself.** The helper pod is deleted when the session ends.
  It also expires on its own (`--ttl`) if the client is killed.
- **Fast reconnects.** With `--keep`, later sessions reuse the helper pod and
  connect almost instantly.
- **Portable.** Works with bash 3.2 (the macOS default) and Linux.

### Requirements

- `kubectl` 1.27 or later (older versions work, except `--debug`).
- `bash` 3.2 or later.
- Linux nodes, including Bottlerocket (EKS Auto Mode). Windows nodes are not
  supported.
- Permission to create **privileged pods** that use `hostPID`, `hostNetwork`
  and a `hostPath` volume (see [Permissions](#permissions)).
- Optional: `fzf`, for a nicer interactive picker.

---

## Installation

kubectl finds plugins by name: any executable called `kubectl-<name>` on your
`PATH` becomes `kubectl <name>`.

### Download the script

```sh
curl -fsSLo kubectl-ssh https://raw.githubusercontent.com/dmechas/kubectl-plugins/main/kubectl-ssh
chmod +x kubectl-ssh
sudo mv kubectl-ssh /usr/local/bin/
```

### From a clone

```sh
git clone https://github.com/dmechas/kubectl-plugins.git
cd kubectl-plugins
install -m 0755 kubectl-ssh ~/.local/bin/   # any directory on your PATH
```

### Verify

```sh
kubectl plugin list   # should list kubectl-ssh
kubectl ssh --version
```

### Uninstall

```sh
kubectl ssh cleanup               # remove any helper pods that are still around
rm "$(command -v kubectl-ssh)"
```

---

## Usage

```text
kubectl ssh node [NODE] [flags] [-- COMMAND [ARGS...]]
kubectl ssh node/NODE    [flags] [-- COMMAND [ARGS...]]
kubectl ssh pod  [POD]   [flags] [-- COMMAND [ARGS...]]
kubectl ssh TYPE/NAME    [flags] [-- COMMAND [ARGS...]]   (pod/x, deploy/x, sts/x, ...)
kubectl ssh list    [-n NAMESPACE]
kubectl ssh cleanup [-n NAMESPACE]
```

- Leave out `NODE` or `POD` to pick one interactively. If there is only one,
  it is used automatically.
- Everything after the node or pod name is the command. The `--` is optional,
  but it makes the split clear.
- A **single** argument runs through `sh -c`, so pipes, globs and `&&` work:
  `kubectl ssh node n1 'ps aux | grep kubelet'`.
- **Several** arguments run as-is, with no shell in between:
  `kubectl ssh node n1 -- ls -la /etc`.
- Status messages go to stderr, so stdout carries only the command's output.

### Nodes

```sh
# Shell on the only node (kind, minikube, Docker Desktop), or pick one
kubectl ssh node

# Shell on a specific node
kubectl ssh node worker-1
kubectl ssh node/worker-1

# Run a single command
kubectl ssh node worker-1 -- journalctl -u kubelet -n 50 --no-pager
kubectl ssh node worker-1 'crictl ps | grep coredns'

# Copy a file from the node to your machine
kubectl ssh node worker-1 -- cat /var/lib/kubelet/config.yaml > kubelet-config.yaml

# Copy a file to the node
kubectl ssh node worker-1 -- sh -c 'cat > /tmp/script.sh' < script.sh

# Run a command on every node
kubectl ssh node --all -- uptime

# Run a command on the nodes that match a label selector
kubectl ssh node -l node-role.kubernetes.io/worker= --all -- df -h /

# Pick a node interactively from the ones that match a selector
kubectl ssh node -l topology.kubernetes.io/zone=eu-west-1a

# Keep the helper pod so later sessions connect almost instantly
kubectl ssh node worker-1 --keep
kubectl ssh node worker-1          # reuses the kept pod

# Use an image that has more tools (tcpdump, dig, curl, ...)
kubectl ssh node worker-1 --image nicolaka/netshoot:v0.13

# Target another cluster
kubectl ssh node worker-1 --context prod-eu
```

### Pods and workloads

```sh
# Shell in a pod (bash, ash or sh, whichever exists)
kubectl ssh pod my-pod -n my-namespace
kubectl ssh pod/my-pod -n my-namespace -c sidecar

# Pick a running pod interactively, optionally filtered by label
kubectl ssh pod -n my-namespace -l app=api

# Any resource that kubectl exec accepts
kubectl ssh deploy/api -n prod -- env
kubectl ssh sts/postgres -n db -- psql -c 'select 1'

# Pods with no shell (distroless, scratch)
kubectl ssh pod my-distroless-pod --debug
# The target container's filesystem is at /proc/1/root inside the debug shell.
```

### Housekeeping

```sh
kubectl ssh list        # helper pods created by the plugin, in all namespaces
kubectl ssh cleanup     # delete them
```

### Flags

| Flag                 | Applies to               | Description                                                                                                                                     |
| -------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `-n, --namespace NS` | node, pod, list, cleanup | **pod:** namespace of the target pod. **node:** namespace for the helper pod (default `kube-system`). **list/cleanup:** limit to one namespace. |
| `-c, --container C`  | pod                      | Container to enter.                                                                                                                             |
| `-l, --selector SEL` | node, pod                | Label selector that filters the nodes or pods to choose from.                                                                                   |
| `--all`              | node                     | Run `COMMAND` on every node (or every node matching `-l`).                                                                                      |
| `--image IMG`        | node, `--debug`          | Helper image (default `busybox:1.37`).                                                                                                          |
| `--mode MODE`        | node                     | `nsenter` (default) or `chroot`. See [How it works](#how-it-works).                                                                             |
| `--keep`             | node                     | Keep the helper pod after the session so later sessions reuse it.                                                                               |
| `--ttl DURATION`     | node                     | Maximum lifetime of the helper pod: `300`, `30m`, `8h`, `2d` (default `8h`).                                                                    |
| `--timeout SECS`     | node                     | How long to wait for the helper pod to start (default `120`).                                                                                   |
| `--debug`            | pod                      | Use an ephemeral debug container instead of `kubectl exec`.                                                                                     |
| `-q, --quiet`        | all                      | Print errors only.                                                                                                                              |
| `-h, --help`         | all                      | Show help.                                                                                                                                      |
| `--version`          | all                      | Show the version.                                                                                                                               |

These kubectl flags are passed through: `--context`, `--kubeconfig`,
`--cluster`, `--user`, `--as`, `--as-group`, `-s/--server`, `--token`,
`--request-timeout` and `--insecure-skip-tls-verify`.

### Configuration

Environment variables set the defaults. Command-line flags take precedence.

| Variable                | Default        | Matching flag    |
| ----------------------- | -------------- | ---------------- |
| `KUBECTL_SSH_IMAGE`     | `busybox:1.37` | `--image`        |
| `KUBECTL_SSH_NAMESPACE` | `kube-system`  | `-n` (node mode) |
| `KUBECTL_SSH_MODE`      | `nsenter`      | `--mode`         |
| `KUBECTL_SSH_TIMEOUT`   | `120`          | `--timeout`      |
| `KUBECTL_SSH_TTL`       | `8h`           | `--ttl`          |

For example, in an air-gapped cluster that mirrors images internally:

```sh
export KUBECTL_SSH_IMAGE=registry.internal.example.com/library/busybox:1.37
```

---

## How it works

### Nodes

1. The plugin creates a pod in `kube-system` and pins it to the node with
   `spec.nodeName`. Because that skips the scheduler, it works even when the
   node is cordoned or tainted (the pod tolerates every taint).
2. The pod runs `sleep <ttl>` in a container with `privileged: true`,
   `hostPID`, `hostNetwork`, `hostIPC`, and the host's `/` mounted at `/host`.
   It has `activeDeadlineSeconds` set to the TTL, so Kubernetes removes it even
   if nobody else does.
3. The session is a `kubectl exec` into that pod that runs:
   - **`nsenter` mode (default):** `nsenter -t 1 -m -u -i -n -p`. This enters
     the namespaces of the host's PID 1, so you are effectively root on the
     node, with the host's filesystem, processes, network and hostname.
   - **`chroot` mode:** `chroot /host`. This changes the root directory but stays
     in the container's mount namespace. Use it when `nsenter` is unavailable,
     for example in a custom `--image` without it.

   Interactive shells and single-string commands need a POSIX `sh` on the
   host, so the plugin checks for one first. Some hosts have none, for example
   Bottlerocket, which EKS Auto Mode uses. On those, the plugin opens a shell in
   the helper container instead. That shell still sees the host's processes and
   network, has the host's filesystem at `/host`, and defines
   `host <command>` to run a host binary in the host's namespaces:

   ```console
   $ kubectl ssh node i-0828fb437ed0a3cd5
   Node i-0828fb437ed0a3cd5 has no POSIX shell (Bottlerocket?). Opening a shell in the helper
   ...
   [helper@i-0828fb437ed0a3cd5] /host # host journalctl -u kubelet -n 20
   ```

   Commands passed as several arguments (`-- journalctl -u kubelet`) don't need
   a shell, so they always run directly on the host.

4. When the session ends, on Ctrl-C, or when the terminal closes, the pod is
   deleted. Pods created with `--keep` are labelled `kubectl-ssh/keep=true` and
   are reused by later sessions on the same node that use the same image.

All helper pods carry the label `app.kubernetes.io/managed-by=kubectl-ssh`.
That's how `list` and `cleanup` find them.

### Pods

`kubectl ssh pod` is a wrapper around `kubectl exec` that finds a usable shell
and passes `TERM` through. With `--debug`, it calls
`kubectl debug --target=<container> --profile=general`. That adds an ephemeral
container that shares the target container's process namespace, so the
target's processes and its root filesystem (`/proc/1/root`) are visible.

> **Note:** You can't remove ephemeral containers. They stay in the pod spec
> until the pod is deleted.

---

## Permissions

Node access is root access to the node. Only cluster administrators should
have it.

### RBAC

These are the permissions the plugin needs. Bind the role only to people who
are allowed root on the nodes.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubectl-ssh-node
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kubectl-ssh-node
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "get", "list", "delete"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "get"]
```

Pod mode only needs the usual `pods/exec` permission in the target namespace.
`--debug` also needs `patch` on `pods/ephemeralcontainers`.

### Pod Security and policy engines

The helper pod needs the **privileged**
[Pod Security Standard](https://kubernetes.io/docs/concepts/security/pod-security-standards/).
`kube-system` is usually exempt from Pod Security Admission, Kyverno and
Gatekeeper policies. If your cluster restricts it anyway, point the plugin at a
namespace that allows privileged pods:

```sh
kubectl create namespace node-debug
kubectl label namespace node-debug pod-security.kubernetes.io/enforce=privileged
kubectl ssh node worker-1 -n node-debug
# or: export KUBECTL_SSH_NAMESPACE=node-debug
```

Managed platforms that forbid privileged pods (for example GKE Autopilot) can't
be used with node mode.

### Good practice

- Every session is a pod creation and an exec. Both are recorded in the
  Kubernetes audit log, attributed to your user.
- Keep `--ttl` short and run `kubectl ssh cleanup` after using `--keep`.
- Prefer a pinned `--image` from a registry you trust.

## License

Released under the [MIT License](LICENSE).
