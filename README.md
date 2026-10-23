# NVIDIA OpenShell on Kubernetes

This project documents a local NVIDIA OpenShell 0.1.5 deployment on a Kind
Kubernetes cluster, including Calico network policy enforcement and short
sandbox demonstrations.

## Project guide

See [OPEN-SHELL-KUBERNETES.md](./OPEN-SHELL-KUBERNETES.md) for the architecture,
Windows/WSL2 setup and recovery commands, demonstrations, verification evidence,
limitations, and cleanup instructions.

The `kind-config.yaml` file is the Kind cluster configuration used for this
setup. The guide describes the Kubernetes resources and commands; credentials,
certificates, kubeconfig files, and live cluster state are not included in this
repository.

## Important

This is a local development demonstration, not a production security
certification. Review the guide's verified checks and limitations before
reproducing it.

## Screenshots

These screenshots show the successful demonstrations documented in the guide.
The unapproved-destination terminal screenshot is intentionally omitted: by
itself, a curl connection error does not prove that OpenShell policy caused the
request to be denied.

### Authenticated Kubernetes gateway

![OpenShell Kubernetes gateway connected and authenticated over mTLS](./screenshots/gateway-mtls-authenticated.png)

### Command executed inside the sandbox

![Word-frequency command executed in the Kubernetes sandbox](./screenshots/sandbox-text-processing.png)

### Sandbox network boundary

![Gateway connection succeeds while access to the sandbox boundary service times out](./screenshots/network-boundary-test.png)

### OpenShell egress rule loaded

![Sandbox policy allowing read-only curl access to example.com is loaded](./screenshots/openshell-egress-policy.png)

### Allowed outbound request

![HTTPS request to example.com succeeds inside the sandbox](./screenshots/allowed-example-request.png)
