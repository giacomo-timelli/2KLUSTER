# 2KLUSTER

2KLUSTER is a Proof-of-Concept platform for integrating cloud-native services with High Performance Computing resources in the context of distributed scientific workflows.

The project was developed to support the submission, execution, storage and visualization of molecular dynamics workloads by combining Kubernetes-based services with an external Slurm-based HPC backend. The platform uses MinIO as S3-compatible object storage, OIDC-based authentication through INDIGO-IAM, a web application for job submission, and Mol* for browser-based molecular visualization.

## Overview

The main goal of 2KLUSTER is to provide a simplified user-facing interface for submitting molecular dynamics jobs while keeping the computational execution on an HPC system.

The general workflow is the following:

1. A user accesses the platform through the web interface.
2. Authentication is handled through an OIDC-based login flow.
3. Input files and workflow data are organized in MinIO object storage.
4. The web application submits the computational workload to an external Slurm backend through a bridge virtual machine.
5. The job is executed on the HPC infrastructure.
6. Output files are uploaded back to MinIO.
7. Results can be accessed and visualized through Mol*.

This repository contains the configuration files and deployment resources used to reproduce the main components of the proof-of-concept architecture.

## Components

### MinIO

The `minio` directory contains the Kubernetes and Helm configuration files required to deploy MinIO as the S3-compatible object storage layer of the platform.

MinIO is used to store input datasets, configuration files, simulation outputs and logs. The provided manifests include:

* namespace and service configuration;
* ingress configuration for API and console access;
* persistent volume and persistent volume claim configuration;
* Helm values for a standalone MinIO deployment;
* an example S3 access policy;
* an NGINX reverse proxy configuration for MinIO API access.

### Mol*

The `molstar` directory contains the resources used to deploy a Mol* instance in Kubernetes.

Mol* is used as the browser-based molecular visualization component of the platform. It allows users to inspect molecular structures and simulation outputs directly from the web interface.

The directory includes:

* a multi-stage Dockerfile for building and serving the Mol* viewer through NGINX;
* Kubernetes manifests for deployment and service exposure;
* oauth2-proxy configuration for OIDC-based access control;
* an NGINX reverse proxy configuration for path-based routing.

### Web application

The `web-app` directory contains the Kubernetes deployment configuration for the 2KLUSTER web interface.

The web application represents the user-facing layer of the platform. It is responsible for collecting job information, interacting with the object storage layer, and coordinating the submission of molecular dynamics jobs to the HPC backend.

The configuration includes:

* namespace and deployment resources;
* environment variables for the HPC login node, bridge VM and MinIO endpoint;
* service and ingress configuration;
* oauth2-proxy configuration for authenticated access;
* NGINX reverse proxy configuration for exposing the application under the `/2kluster` path.

## Architecture

The platform follows a hybrid cloud-HPC architecture.

```text
User
 │
 ▼
Ingress / HTTPS
 │
 ▼
oauth2-proxy + OIDC authentication
 │
 ├── 2KLUSTER web application
 │       │
 │       ├── MinIO object storage
 │       │
 │       └── Bridge VM
 │               │
 │               └── Slurm-based HPC backend
 │
 └── Mol* visualization service
```

The Kubernetes environment hosts the user-facing and data management services, while the actual computational workload is executed on the external HPC infrastructure. This preserves a separation between the cloud and HPC environments while still allowing them to cooperate within the same workflow.

## Requirements

The deployment assumes the availability of:

* a Kubernetes cluster;
* `kubectl`;
* Helm;
* NGINX Ingress Controller;
* cert-manager;
* a valid domain name;
* an OIDC identity provider, such as INDIGO-IAM;
* a MinIO-compatible S3 endpoint;
* a bridge VM able to reach the HPC login node;
* an HPC backend managed by Slurm;
* Docker, only if the Mol* image needs to be rebuilt.

## Configuration

Most manifests contain placeholder values that must be replaced before deployment.

Common placeholders include:

```text
<HOST>
<TLS_CERTIFICATE>
<CLUSTER-ISSUER>
<OIDC-ISSUER>
<CLIENT_ID>
<CLIENT_SECRET>
<COOKIE_SECRET>
<MINIO-ENDPOINT>
<MINIO_CLIENT_ID>
<BUCKET-NAME>
<BRIDGE-PRIVATE-IP>
<BRIDGE-USER>
<HPC_PORTAL_HOSTNAME>
<WORKER_NODE>
```

Before applying the manifests, replace these values according to the target infrastructure.

Sensitive values, such as client secrets, cookie secrets and access credentials, should not be committed in plain text in a public repository. In a real deployment, they should be managed through Kubernetes Secrets or an external secret management system.

## Deployment notes

The manifests are intended as configuration templates for the proof-of-concept environment. A typical deployment order is:

```bash
# Clone the repository
git clone https://github.com/giacomo-timelli/2KLUSTER.git
cd 2KLUSTER
```

### MinIO

```bash
kubectl apply -f minio/manifests/clusterissuer.yaml
kubectl apply -f minio/manifests/minio-configuration-manifest.yaml
kubectl apply -f minio/revproxy/revproxy-minio-configuration-manifest.yaml
```

If MinIO is deployed through Helm, adapt the values file and run:

```bash
helm upgrade --install minio minio/minio \
  --namespace minio-system \
  --values minio/manifests/minio-helm-values.yaml
```

### Mol*

```bash
kubectl apply -f molstar/manifests/molstar-configuration-mainfest.yaml
kubectl apply -f molstar/manifests/molstar-aai-configuration.yaml
kubectl apply -f molstar/revproxy/revproxy-molstar-configuration-manifest.yaml
```

To rebuild the Mol* image:

```bash
cd molstar
docker build -t molstar-viewer .
```

### Web application

```bash
kubectl apply -f web-app/manifests/webapp-configuration-manifest.yaml
kubectl apply -f web-app/manifests/webapp-aai-configuration.yaml
kubectl apply -f web-app/revproxy/revproxy-webapp-configuration-manifest.yaml
```

The deployment order may need to be adapted depending on the specific Kubernetes cluster, ingress configuration, namespace availability and secret management strategy.

## Status

This repository represents an academic proof-of-concept and is not intended to be used as a production-ready deployment without further hardening.

Possible future improvements include:

* stronger secret management;
* private bucket policies and user-aware storage authorization;
* token exchange for delegated access to MinIO and HPC resources;
* improved user-level job submission;
* support for more complex multi-node Slurm workloads;
* more complete automation of the deployment procedure.

## License

This project is licensed under the Apache License 2.0. See the `LICENSE` file for details.
