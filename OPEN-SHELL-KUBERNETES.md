# OpenShell on the local Kind cluster

This documents the verified NVIDIA OpenShell 0.1.5 Kubernetes setup for this
Windows/WSL2 development machine. The Kubernetes context is `kind-openshell`.
Do not use this guide to modify the separate `ai-governance` or `dev` clusters.

## Architecture

- A single-node Kind cluster runs Kubernetes v1.36.1 with Calico v3.31.3 as
  its policy-enforcing CNI.
- The Agent Sandbox controller and `sandboxes.agents.x-k8s.io` API run in
  `agent-sandbox-system`.
- The OpenShell 0.1.5 gateway runs in namespace `openshell`. Its Kubernetes
  compute driver creates an Agent Sandbox resource, a supervisor Pod, a
  workload Pod, a boundary Service, a workspace PVC, and NetworkPolicies.
- The Agent Sandbox controller creates the workload Pod. Calico enforces the
  workload NetworkPolicy; the workload policy permits supervisor ingress to
  the boundary port and denies workload-initiated egress.
- The CLI's `k8s` gateway is distinct from its local Docker gateway named
  `openshell`.

The OpenShell gateway uses mTLS for local CLI authentication. The Helm release
has `server.auth.allowUnauthenticatedUsers=false`; its gateway TLS and client
certificate secrets are managed in Kubernetes. Do not copy private keys into
this file or disable TLS verification.

## Connect after a restart

In a Windows PowerShell terminal, find the current WSL-facing Windows IP and
start a port-forward bound only to that interface:

```powershell
kubectl config use-context kind-openshell
$wslHost = Get-NetIPAddress -AddressFamily IPv4 |
  Where-Object { $_.InterfaceAlias -like '*WSL*' -and $_.AddressState -eq 'Preferred' } |
  Select-Object -First 1 -ExpandProperty IPAddress
wsl.exe --user root --distribution Ubuntu -- sh -c "sed -i '/host.docker.internal/d' /etc/hosts; echo '$wslHost host.docker.internal' >> /etc/hosts"
kubectl --context kind-openshell -n openshell port-forward svc/openshell 8080:8080 --address $wslHost
```

Keep this terminal open. The forward is for local evaluation; it is not an
ingress or public gateway. The first command refreshes the WSL host alias for
the current WSL session; rerun it after WSL regenerates `/etc/hosts`. In WSL
Ubuntu, check the separate Kubernetes gateway:

```bash
openshell --gateway k8s status
openshell --gateway k8s whoami
```

Expected status is connected/authenticated with version `0.1.5`, and the user
identity is provided by the `openshell-client` mTLS certificate. The Docker
gateway registration remains separate; specify `--gateway k8s` for Kubernetes
operations.

If the context or gateway is unavailable, inspect it before making changes:

```powershell
kind get clusters
kubectl config current-context
kubectl --context kind-openshell get nodes
kubectl --context kind-openshell get pods -A
kubectl --context kind-openshell -n openshell get pods,svc,pvc
kubectl --context kind-openshell -n agent-sandbox-system get pods
```

The port-forward must be restarted after it exits. If the WSL-facing IP
changed, rerun the PowerShell snippet to discover and bind the current address.

## Create and use a sandbox

In WSL Ubuntu, create a small sandbox with a long-running harmless process:

```bash
openshell --gateway k8s sandbox create \
  --name k8s-demo \
  --from nvcr.io/nvidia/base/ubuntu:24.04 \
  --cpu 500m \
  --memory 512Mi \
  --detach \
  -- sh -c 'sleep 1800'
```

The demo process sleeps for 30 minutes. After it exits, the sandbox may show as
`Stopped`; resume it with `openshell --gateway k8s sandbox start k8s-demo`.

Inspect and execute a harmless command:

```bash
openshell --gateway k8s sandbox list
openshell --gateway k8s sandbox get k8s-demo
openshell --gateway k8s sandbox exec k8s-demo -- sh -lc 'id; echo OpenShell_Kubernetes_sandbox_works'
```

The sandbox name is limited by the CLI; use a short name. Confirm the
Kubernetes resources in PowerShell:

```powershell
kubectl --context kind-openshell -n openshell get sandboxes.agents.x-k8s.io
kubectl --context kind-openshell -n openshell get pods,svc,pvc,networkpolicy -o wide
```

The workload should be Ready alongside an `os-supervisor-*` Pod, and the
gateway should report the sandbox phase as `Ready`. The demonstrated command
prints the sandbox UID and `OpenShell_Kubernetes_sandbox_works`.

## Three short project demos

### 1. Process sample text in the sandbox

With the gateway connected and `k8s-demo` Ready, run this in WSL Ubuntu:

```bash
openshell --gateway k8s sandbox exec k8s-demo -- sh -c "printf '%s\n' 'OpenShell processes sample text.' 'This Kubernetes sandbox is isolated.' 'OpenShell sandbox processes text safely.' | tr '[:upper:]' '[:lower:]' | tr -cs '[:alnum:]' '\n' | sort | uniq -c | sort -nr"
```

This counts words inside the sandbox. The output should include `2 openshell`,
`2 processes`, `2 sandbox`, and `2 text`. The command is executed in the
Kubernetes workload Pod, not on the Windows host.

### 2. Show an allowed path and a blocked sandbox boundary

The `np-test/client` Pod is a connectivity probe. In PowerShell, look up the
current service addresses and test:

```powershell
$gatewayIP = kubectl --context kind-openshell -n openshell get svc openshell -o jsonpath='{.spec.clusterIP}'
$boundaryIP = kubectl --context kind-openshell -n openshell get svc -l openshell.ai/component=boundary-service -o jsonpath='{.items[0].spec.clusterIP}'
kubectl --context kind-openshell -n np-test exec client -- nc -z -v -w 3 $gatewayIP 8080
kubectl --context kind-openshell -n np-test exec client -- nc -z -v -w 3 $boundaryIP 5500
```

The gateway connection should report `open`. The boundary connection should
time out/fail: the OpenShell workload NetworkPolicy permits boundary ingress
from its supervisor, not this unrelated probe Pod. The latter command exits
nonzero on the expected block. Confirm that the service endpoint exists and
the policy selects workload Pods before interpreting a timeout:

```powershell
kubectl --context kind-openshell -n openshell get svc -l openshell.ai/component=boundary-service -o wide
kubectl --context kind-openshell -n openshell get networkpolicy openshell-sandbox-workloads -o yaml
```

This is a focused demonstration of one enforced path, not proof that every
possible egress or network path is blocked.

### 3. Allow one outbound destination and deny another

This is an OpenShell policy test, separate from the Calico NetworkPolicy test
above. It uses a dedicated sandbox so the `k8s-demo` policy is untouched. The
policy is sandbox-scoped, not gateway-global, and allows only `/usr/bin/curl`
to make read-only HTTPS requests to `example.com`. This demonstrates
OpenShell governing an outbound request from a sandbox process.

Create the test sandbox in WSL Ubuntu:

```bash
openshell --gateway k8s sandbox create \
  --name net-policy \
  --from curlimages/curl:8.12.1 \
  --cpu 500m \
  --memory 512Mi \
  --detach \
  -- sh -c 'sleep 1800'
```

Before adding a rule, outbound access is denied by default. This request should
fail:

```bash
openshell --gateway k8s sandbox exec net-policy -- \
  /usr/bin/curl --connect-timeout 5 --max-time 10 --silent --show-error \
  https://example.com/
```

Add a narrow read-only rule for one binary and one destination:

```bash
openshell --gateway k8s policy update net-policy \
  --rule-name example_site \
  --binary /usr/bin/curl \
  --add-endpoint example.com:443:read-only:rest:enforce \
  --wait
```

The policy update should report that the new version is loaded. The allowed
request returns the Example Domain page:

```bash
openshell --gateway k8s sandbox exec net-policy -- \
  /usr/bin/curl --connect-timeout 8 --max-time 20 --silent --show-error --fail \
  https://example.com/
```

This request to an unlisted destination should still fail:

```bash
openshell --gateway k8s sandbox exec net-policy -- \
  /usr/bin/curl --connect-timeout 5 --max-time 10 --silent --show-error \
  https://api.github.com/zen
```

Inspect the policy and OpenShell's enforcement events:

```bash
openshell --gateway k8s policy get net-policy --base --output json
openshell --gateway k8s policy list net-policy
openshell --gateway k8s logs net-policy --since 5m --source sandbox
```

Expected evidence is an active `example_site` rule listing `/usr/bin/curl`
and `example.com:443`, an `ALLOWED`/`HTTP:GET ALLOWED` event for
`https://example.com/`, and a `DENIED` event for `api.github.com:443`.
The denied `curl` exits nonzero; that is the expected outcome. The test
sandbox and its policy are left in the cluster so the result can be inspected.

Optional cleanup: deleting a sandbox also deletes its workspace data. Run only
the command for the sandbox you intend to remove:

```bash
openshell --gateway k8s sandbox delete k8s-demo
openshell --gateway k8s sandbox delete net-policy
```

This deletes the sandbox workspace data. It does not uninstall the gateway or
delete the gateway's PVC, certificates, or other Kind clusters. Stop the
port-forward with Ctrl+C when it is no longer needed.

## Verified checks and limits

Initial cluster and gateway checks were verified on 2026-10-22. The following
short demos were verified on 2026-10-23:

- The `openshell` node, Calico node/controller, Agent Sandbox controller, and
  OpenShell gateway were Ready/Running. The Helm release remained deployed at
  chart/app version 0.1.5 with unauthenticated access disabled.
- The WSL-only port-forward and authenticated mTLS CLI connection succeeded.
- The existing stopped sandbox `k8s-demo` was resumed. It became Ready and
  ran the sample text-processing command, printing the expected counts
  (`openshell`, `processes`, `sandbox`, and `text` each appeared twice).
- Kubernetes showed the Sandbox resource, workload Pod, supervisor Pod,
  boundary Service, workspace PVC, and sandbox NetworkPolicies.
- From `np-test/client`, a TCP connection to the OpenShell service succeeded;
  a connection to the sandbox boundary service timed out as expected. The
  boundary Service endpoint existed, and the workload policy restricted its
  ingress to the supervisor.
- The supervisor log reported “Isolation boundary enforcement confirmed.”
  These checks demonstrate policy enforcement for the tested paths, not every
  possible network path or policy.
- A separate `net-policy` sandbox used the `curlimages/curl:8.12.1` image.
  With no network rule, its request to `api.github.com` was denied. After a
  sandbox-only policy update, a read-only GET to `example.com` returned the
  Example Domain page, while `api.github.com` remained denied. Policy history
  showed revision 4 loaded, and supervisor logs recorded the allowed request
  and denied destination. The rule is scoped to `/usr/bin/curl` and
  `example.com:443`; it is not a global gateway policy.

The `k8s-demo` and `net-policy` sandboxes were left in the cluster after these
demos. They use 30-minute sleep processes and can be restarted with the command
above if either has stopped. Sandbox status may change after the verification
date; inspect it before running a demo.

This is a single-node local development cluster, not a production security
boundary. Network tests covered selected paths, not a full egress audit.
OpenShell's supervisor isolation and Calico NetworkPolicies are demonstrated
controls; hardware/VM isolation, production identity management, multi-node
behavior, and all possible escape/network paths are not certified here.

## Upstream documentation

- [OpenShell 0.1.5 Kubernetes setup](https://docs.nvidia.com/openshell/v0.1.5/kubernetes/setup)
- [OpenShell 0.1.5 access control](https://docs.nvidia.com/openshell/v0.1.5/kubernetes/access-control)
- [Kubernetes SIG Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox)
