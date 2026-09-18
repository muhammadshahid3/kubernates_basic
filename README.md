# Rocket Service — Kubernetes Helm Chart Documentation

This document explains the Kubernetes **Deployment** and **Service** manifests used to deploy the `rocket-service` application. Both files are Helm templates, meaning values like name, namespace, image, and ports are injected from `values.yaml`.

---

## Table of Contents

- [Overview](#overview)
- [Deployment Manifest](#deployment-manifest)
- [Service Manifest](#service-manifest)
- [Request Flow](#request-flow)
- [Known Issue & Fix](#known-issue--fix)

---

## Overview

| Resource   | API Group | Purpose                                                        |
|------------|-----------|------------------------------------------------------------------|
| Deployment | `apps/v1` | Manages pod lifecycle — creation, scaling, restarts, rollouts   |
| Service    | `v1`      | Provides a stable network endpoint to route traffic to pods     |

---

## Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
```
Declares this resource as a **Deployment**, which belongs to the `apps/v1` API group (unlike `Pod` and `Service`, which live in the core `v1` group). A Deployment's job is to manage pods: how many should run, recreate them on crash, and handle updates/rollbacks.

```yaml
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.name }}
```
| Field       | Resolved Value    | Description                                              |
|-------------|--------------------|------------------------------------------------------------|
| `name`      | `rocket-service`   | The Deployment's identity (visible via `kubectl get deployment`) |
| `namespace` | `rocket`           | The namespace this Deployment is created in               |
| `labels`    | `app: rocket-service` | An organizational tag on the Deployment object itself (used for filtering, e.g. `kubectl get deploy -l app=rocket-service`). Does **not** affect the pods. |

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
```
Specifies how many identical pods should run. Currently set to `1`. Increasing this (e.g. to `3`) would make Kubernetes run 3 identical pods for load balancing and high availability.

```yaml
  selector:
    matchLabels:
      app: {{ .Values.name }}
```
Tells the Deployment which pods belong to it: any pod carrying the label `app: rocket-service` is considered "mine" and will be managed by this Deployment. This is the link between the Deployment and its pods.

```yaml
  template:
    metadata:
      labels:
        app: {{ .Values.name }}
```
This is the **pod blueprint**. Whenever the Deployment needs to create a pod (first launch, crash recovery, scale-up), it copies this template. The label applied here (`app: rocket-service`) matches the `selector.matchLabels` above, which is how the Deployment recognizes its own pods.

```yaml
    spec:
      containers:
        - name: {{ .Values.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```
| Field              | Resolved Value                          | Description                                            |
|--------------------|-------------------------------------------|------------------------------------------------------|
| `name`             | `rocket-service`                        | Name of the container inside the pod                  |
| `image`            | `shahiddevops1/rocket-site:latest`      | Docker image pulled and run for this container         |
| `imagePullPolicy`  | `Always`                                | Forces a fresh image pull on every pod creation, ensuring the latest version is used |

```yaml
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
```
Declares the port the application listens on inside the container (nginx listens on `80`). This is purely **informational/documentation** for Kubernetes — it does not open or block the port. Whether the container actually listens on that port depends entirely on the image's own configuration (e.g. its Dockerfile).

---

## Service Manifest

```yaml
apiVersion: v1
kind: Service
```
Services belong to the core `v1` API group. A Service's job is to give pods a **stable network address**. Pods are ephemeral (they get new names/IPs when recreated), but a Service keeps a fixed name/IP that other components can rely on.

```yaml
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.name }}
```
Same pattern as the Deployment: the Service's name, namespace, and its own organizational label.

```yaml
spec:
  type: {{ .Values.service.type }}
```
Defines how the Service is exposed. `ClusterIP` means it is **only reachable from within the cluster** — not from outside. This is why external access requires `kubectl port-forward`.

| Service Type   | Accessibility                                              |
|-----------------|-------------------------------------------------------------|
| `ClusterIP`     | Internal cluster access only (default)                     |
| `NodePort`      | Accessible externally via node IP + a fixed port            |
| `LoadBalancer`  | Provisions an external cloud load balancer                  |

```yaml
  ports:
    - port: {{ .Values.service.port }}
```
The port on which the **Service itself** is available inside the cluster. Any pod sending a request to `rocket-service:<port>` will hit this port.

```yaml
      targetPort: http
```
Defines which port **inside the pod** the request should be forwarded to. Here it references the name `http`, matching the `name: http` given to `containerPort` in the Deployment — Kubernetes resolves the name to the actual port number.

> ⚠️ **Root cause of the original issue:** `service.port` in `values.yaml` was set to `8000`, but the container was actually listening on port `80`. This mismatch caused traffic forwarding to fail with a connection-refused error.

```yaml
      protocol: TCP
      name: http
```
The protocol is `TCP` (HTTP itself runs over TCP), and this port entry is named `http`.

```yaml
  selector:
    app: {{ .Values.name }}
```
**The most important line in the Service.** A Service does not serve traffic itself — it acts as a router. It says: "send traffic to any pod labeled `app: rocket-service`." This is how the Service discovers which pod(s) to route to, regardless of how many times those pods restart or get replaced — as long as the label matches, the Service finds them.

---

## Request Flow

```
Client
  │
  ▼
Service (port 8000/80 → selector: app=rocket-service)
  │  resolves selector to matching pods
  ▼
Pod (label: app=rocket-service, container listening on port 80)
```

---

## Known Issue & Fix

**Symptom:** Connection refused when accessing the service.

**Cause:** `service.port` (`8000`) did not correspond to the actual port the container was listening on (`80`), and/or `targetPort` was not correctly resolving to the container's port.

**Fix:** Ensure `targetPort` in the Service matches the `containerPort` (or its `name`) defined in the Deployment, so traffic is correctly forwarded from the Service to the pod's actual listening port.

---

## Full Annotated Deployment Reference

```yaml
apiVersion: apps/v1                          # apps/v1 API group resource
kind: Deployment                              # Creating a Deployment resource

metadata:                                     # Deployment's own identity
  name: {{ .Values.name }}                    # → rocket-service
  namespace: {{ .Values.namespace }}          # → rocket
  labels:
    app: {{ .Values.name }}                   # Tag/label on the Deployment

spec:                                         # What the Deployment should do
  replicas: {{ .Values.replicaCount }}        # → 1 (number of pod copies)

  selector:                                   # Which pods this Deployment manages
    matchLabels:
      app: {{ .Values.name }}                 # Pods with this label belong to it

  template:                                   # Blueprint for creating pods
    metadata:
      labels:
        app: {{ .Values.name }}               # Label applied to new pods
    spec:
      containers:
        - name: {{ .Values.name }}            # Container name
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          # → shahiddevops1/rocket-site:latest
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          # → Always (pull a fresh image every time)
          ports:
            - name: http                      # Port name (matched by the Service)
              containerPort: {{ .Values.service.port }}
              # → 80 (nginx listens on this port)
              protocol: TCP
```
