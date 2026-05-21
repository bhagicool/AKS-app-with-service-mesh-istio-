# AKS-app-with-service-mesh-istio-
AKS Cross-Namespace Frontend/API Isolation with FQDN-Based Egress Restriction
# AKS Cross-Namespace Frontend/API Isolation with FQDN-Based Egress Restriction

## Overview

This Proof of Concept (POC) demonstrates how to implement:

* Cross-namespace application communication in AKS
* Strict outbound FQDN-based egress governance
* Namespace isolation
* Zero-trust outbound connectivity model
* Inbound publishing using AGIC
* Workload-specific outbound access using Istio

The implementation ensures that even if a frontend pod is compromised, it cannot:

* Access arbitrary internet destinations
* Call unauthorized internal APIs
* Download malicious tooling from the internet
* Pivot laterally across namespaces

---

# Architecture Summary

## Namespaces

| Namespace | Frontend    | Backend API     |
| --------- | ----------- | --------------- |
| example   | example.com | api.example.com |
| contoso   | contoso.com | api.contoso.com |

---

# Communication Rules

## Allowed Traffic

| Source Pod       | Allowed Destination |
| ---------------- | ------------------- |
| example frontend | api.contoso.com     |
| contoso frontend | api.example.com     |

---

## Blocked Traffic

| Source Pod       | Blocked Destination             |
| ---------------- | ------------------------------- |
| example frontend | api.example.com                 |
| example frontend | google.com / arbitrary internet |
| contoso frontend | api.contoso.com                 |
| contoso frontend | arbitrary internet              |

---

# High-Level Architecture

```text
                         INTERNET
                              |
               --------------------------------
               |                              |
        Azure Application Gateway      Azure Application Gateway
              (AGIC-1)                       (AGIC-2)
               |                              |
       -------------------           -------------------
       |                 |           |                 |
   example.com      api.example.com  contoso.com  api.contoso.com
       |                 |           |                 |
       -------------------           -------------------
               |                              |
               -------------------------------------------------
                                   AKS CLUSTER
               -------------------------------------------------

        Namespace: example                     Namespace: contoso

   --------------------------         --------------------------
   | example-frontend      |         | contoso-frontend      |
   | + Envoy Sidecar       |         | + Envoy Sidecar       |
   --------------------------         --------------------------
             |                                      |
             | Allowed only                         | Allowed only
             | api.contoso.com                     | api.example.com
             |                                      |
             v                                      v
   --------------------------         --------------------------
   | contoso-api           |         | example-api           |
   | + Envoy Sidecar       |         | + Envoy Sidecar       |
   --------------------------         --------------------------
```

---

# Security Model

The solution uses:

| Component          | Purpose                                |
| ------------------ | -------------------------------------- |
| Istio Service Mesh | Traffic governance                     |
| Envoy Sidecars     | Traffic interception and enforcement   |
| ServiceEntry       | External FQDN registration             |
| Sidecar CRD        | Workload-specific outbound restriction |
| REGISTRY_ONLY      | Default deny outbound posture          |
| AGIC               | Inbound publishing                     |

---

# Why Istio Was Used

Traditional Kubernetes Network Policies operate at:

* Layer 3 / Layer 4
* IP / Port level

This POC required:

* FQDN-based restrictions
* Hostname-aware outbound governance
* Workload-specific egress permissions

Istio was selected because it provides:

* Layer 7 outbound governance
* Hostname-aware policies
* Sidecar-based enforcement
* Service mesh traffic control

---

# Core Istio Concepts Used

## 1. Envoy Sidecar

Each application pod contains:

```text
Application Container
+
Envoy Proxy Sidecar
```

The Envoy sidecar intercepts all traffic using iptables redirection.

Traffic flow:

```text
Application
   ↓
Envoy Sidecar
   ↓
Destination
```

The sidecar enforces:

* Outbound policies
* Routing
* Service discovery
* FQDN restrictions
* mTLS
* Observability

---

## 2. ServiceEntry

ServiceEntry registers external or FQDN-based services into the Istio mesh.

Example:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
spec:
  hosts:
  - api.contoso.com
```

Purpose:

```text
"This hostname exists and is allowed in the mesh registry"
```

Without ServiceEntry:

* Envoy treats destination as unknown
* REGISTRY_ONLY blocks it

---

## 3. Sidecar Resource

The Istio Sidecar resource restricts which workloads may access which hosts.

Example:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
spec:
  workloadSelector:
    labels:
      app: example-frontend

  egress:
  - hosts:
    - "./api.contoso.com"
```

Purpose:

```text
"Only this workload may access this host"
```

---

## 4. REGISTRY_ONLY Outbound Mode

Global mesh outbound policy:

```yaml
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
```

Behavior:

| Mode          | Behavior                                   |
| ------------- | ------------------------------------------ |
| ALLOW_ANY     | Any outbound allowed                       |
| REGISTRY_ONLY | Only ServiceEntry registered hosts allowed |

This creates a:

```text
Default Deny Outbound Model
```

---

# Final Security Flow

```text
Frontend Pod
    ↓
iptables redirect
    ↓
Envoy Sidecar
    ↓
Check ServiceEntry
    ↓
Check Sidecar egress policy
    ↓
Allow or deny traffic
```

---

# Deployment Flow

## Recommended Build Sequence

1. Create AKS Cluster
2. Configure networking and outbound model
3. Deploy AGIC instances
4. Create namespaces
5. Deploy applications and services
6. Configure DNS / hosts resolution
7. Validate normal app flow
8. Enable Istio restrictions
9. Validate positive and negative scenarios

---

# AKS Environment Details

## Cluster Components

| Component          | Purpose             |
| ------------------ | ------------------- |
| AKS                | Kubernetes platform |
| Azure CNI          | Cluster networking  |
| Istio Add-on (ASM) | Service mesh        |
| AGIC               | Ingress integration |
| Envoy              | Traffic proxy       |

---

# Namespace Layout

```text
example namespace
 ├── example-frontend
 ├── example-api
 ├── services
 └── ingress objects

contoso namespace
 ├── contoso-frontend
 ├── contoso-api
 ├── services
 └── ingress objects
```

---

# Inbound Connectivity

## AGIC-Based Publishing

| Hostname        | Published Through     |
| --------------- | --------------------- |
| example.com     | AGIC                  |
| api.example.com | AGIC                  |
| contoso.com     | NGINX Ingress / AGIC  |
| api.contoso.com | NGINX Internal / AGIC |

---

# Validation Results

## Positive Tests

| Test                               | Expected Result | Status |
| ---------------------------------- | --------------- | ------ |
| Access example.com                 | Success         | PASS   |
| Access contoso.com                 | Success         | PASS   |
| example frontend → api.contoso.com | Success         | PASS   |
| contoso frontend → api.example.com | Success         | PASS   |

---

## Negative Tests

| Test                                  | Expected Result | Status |
| ------------------------------------- | --------------- | ------ |
| example frontend → api.example.com    | Blocked         | PASS   |
| contoso frontend → api.contoso.com    | Blocked         | PASS   |
| example frontend → google.com         | Blocked         | PASS   |
| contoso frontend → arbitrary internet | Blocked         | PASS   |

---

# Example Validation Commands

## Allowed Test

```bash
curl http://api.contoso.com
```

Expected:

```text
Hello from Contoso API
```

---

## Blocked Test

```bash
curl http://google.com
```

Expected:

```text
HTTP/1.1 502 Bad Gateway
server: envoy
```

---

# Important Observations

## ServiceEntry vs Sidecar

| Object       | Responsibility                                |
| ------------ | --------------------------------------------- |
| ServiceEntry | Registers allowed FQDNs                       |
| Sidecar      | Restricts which workloads may use those FQDNs |
| Envoy        | Enforces traffic policy                       |

---

# Security Benefits Achieved

The implementation successfully demonstrates:

* Namespace isolation
* Workload-specific outbound governance
* Default deny outbound posture
* Prevention of arbitrary internet access
* Controlled cross-namespace communication
* Zero-trust service mesh architecture

---

# Key Learnings

This POC demonstrates practical enterprise concepts:

| Concept                 | Description                       |
| ----------------------- | --------------------------------- |
| Service Mesh            | Centralized traffic governance    |
| Sidecar Proxy           | Per-pod security enforcement      |
| FQDN Egress Restriction | Hostname-aware outbound filtering |
| Zero Trust Networking   | Least privilege communication     |
| Namespace Isolation     | Reduced lateral movement          |
| Envoy Enforcement       | Runtime traffic control           |

---

# Conclusion

This AKS POC successfully validates a secure cross-namespace communication architecture using Istio service mesh.

The implementation enforces:

* workload-level outbound restrictions
* FQDN-aware egress control
* prevention of unrestricted internet access
* secure frontend-to-API communication

while maintaining:

* inbound accessibility through AGIC
* namespace separation
* repeatable Kubernetes deployment architecture

The final solution demonstrates a practical enterprise-grade zero-trust outbound governance model in Azure Kubernetes Service.
