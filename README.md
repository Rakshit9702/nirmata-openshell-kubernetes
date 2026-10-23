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
