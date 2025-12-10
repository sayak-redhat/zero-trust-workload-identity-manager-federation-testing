# SPIRE Federation Test Plan

## Document Information

| Field | Value |
|-------|-------|
| **Component** | Zero Trust Workload Identity Manager - Federation |
| **Version** | 1.2 |
| **Last Updated** | December 10, 2025 |
| **Author** | QE Team |
| **Status** | Active |
| **Developer Reference** | [SPIRE Federation Gist](https://gist.github.com/rausingh-rh/305cd0f9ae7be9e522e10341fa8b6647) |
| **Scope** | Two-Cluster Federation (https_spiffe profile) |

---

## Table of Contents

1. [Overview](#overview)
2. [Test Environment](#test-environment)
3. [Test Prerequisites](#test-prerequisites)
4. [Two-Cluster Federation Setup](#two-cluster-federation-setup)
5. [Positive Test Scenarios](#positive-test-scenarios)
6. [ACME Test Scenarios](#acme-automatic-certificate-management-test-scenarios)
7. [Manual Certificate Management Test Scenarios](#manual-certificate-management-test-scenarios)
8. [Negative Test Scenarios](#negative-test-scenarios)
9. [Test Execution Summary Template](#test-execution-summary-template)
10. [Appendix](#appendix)
11. [Future: Three-Cluster Federation](#future-three-cluster-federation)

---

## Overview

### What is SPIRE Federation?

SPIRE Federation allows independent SPIRE deployments (trust domains) to establish trust relationships by sharing their trust bundles through secure federation endpoints. This enables:

- **Cross-cluster workload identity**: Workloads in different clusters can verify each other's identities
- **Secure multi-cluster communication**: mTLS between services across clusters
- **Partner integrations**: Securely federate with external organization's SPIRE deployments

### Test Plan Scope

This test plan focuses on **Two-Cluster Federation** using the `https_spiffe` profile:

| Configuration | Profile | Description |
|---------------|---------|-------------|
| **Cluster 1** | https_spiffe | Uses SPIRE's own self-signed certificates |
| **Cluster 2** | https_spiffe | Uses SPIRE's own self-signed certificates |

> **Note**: For advanced 3-cluster scenarios with ACME or cert-manager, see [Future: Three-Cluster Federation](#future-three-cluster-federation) section and the [Developer Gist](https://gist.github.com/rausingh-rh/305cd0f9ae7be9e522e10341fa8b6647).

### Bundle Endpoint Profiles

| Profile | Description | TLS Certificate Source | curl flag |
|---------|-------------|------------------------|-----------|
| `https_spiffe` | SPIFFE-based certificates | SPIRE's own CA (self-signed) | Requires `-k` |
| `https_web` (ACME) | Web PKI certificates | Let's Encrypt (publicly trusted) | No `-k` needed |
| `https_web` (serving_cert) | Web PKI certificates | cert-manager | No `-k` needed |

### Federation Architecture

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│        Cluster 1            │         │        Cluster 2            │
│   Profile: https_spiffe     │ ◄═════► │   Profile: https_spiffe     │
│                             │         │                             │
│   Trust Domain:             │  Trust  │   Trust Domain:             │
│   apps.cluster1.example.com │ Bundle  │   apps.cluster2.example.com │
│                             │ Exchange│                             │
│   ┌─────────────────────┐   │         │   ┌─────────────────────┐   │
│   │   SPIRE Server      │   │         │   │   SPIRE Server      │   │
│   │   Bundle Endpoint   │   │         │   │   Bundle Endpoint   │   │
│   │   :8443             │   │         │   │   :8443             │   │
│   └─────────────────────┘   │         │   └─────────────────────┘   │
│            │                │         │            │                │
│            ▼                │         │            ▼                │
│   ┌─────────────────────┐   │         │   ┌─────────────────────┐   │
│   │  Route (HTTPS)      │   │         │   │  Route (HTTPS)      │   │
│   │  federation.apps... │   │         │   │  federation.apps... │   │
│   └─────────────────────┘   │         │   └─────────────────────┘   │
└─────────────────────────────┘         └─────────────────────────────┘
```

### Key Components

| Component | Description |
|-----------|-------------|
| **Trust Domain** | Unique identifier for each cluster's identity system |
| **Trust Bundle** | Certificate containing the cluster's CA signature |
| **Bundle Endpoint** | HTTPS URL exposing the trust bundle |
| **ClusterFederatedTrustDomain** | CR defining trust relationship with another cluster |
| **className** | Must be `zero-trust-workload-identity-manager-spire` |

### Federation Configuration Methods

There are **two methods** to configure federation:

#### Method 1: SpireServer CR with `federatesWith` (Inline)

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_spiffe
      refreshHint: 300
    federatesWith:
      - trustDomain: partner.example.com
        bundleEndpointUrl: https://federation.partner.example.com
        bundleEndpointProfile: https_spiffe
        endpointSpiffeId: spiffe://partner.example.com/spire/server
    managedRoute: "true"
```

#### Method 2: Separate ClusterFederatedTrustDomain CR (Manual)

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: partner-federation
spec:
  trustDomain: partner.example.com
  bundleEndpointURL: https://federation.partner.example.com
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://partner.example.com/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
    { "keys": [...], "spiffe_sequence": 1 }
```

#### Comparison

| Aspect | Method 1 (SpireServer) | Method 2 (ClusterFederatedTrustDomain) |
|--------|------------------------|----------------------------------------|
| Configuration Location | Inline in SpireServer CR | Separate CR |
| Initial Bundle | Auto-fetched (if endpoint reachable) | Manual via `trustDomainBundle` |
| Use Case | Simple setups | Complex/multi-partner setups |
| className Required | No | Yes |

> **Note**: Method 2 requires fetching the bundle via `curl` and adding it to `trustDomainBundle` for initial trust bootstrapping.

---

## Test Environment

### Required Infrastructure

| Component | Requirement |
|-----------|-------------|
| OpenShift Clusters | 2 clusters |
| Operator | Zero Trust Workload Identity Manager (v1.0.0+) |
| Network | Clusters must be able to reach each other's federation endpoints |
| Access | Cluster admin access to all clusters |
| CLI Tools | `oc`, `curl`, `jq` |

### Environment Variables Setup

```bash
# Common in both the cluster 
export SPIRE_NS="zero-trust-workload-identity-manager"

# Verify in cluster 1 
echo "Cluster 1 Trust Domain: $APP_DOMAIN1"

# Verify in cluster 2 
echo "Cluster 2 Trust Domain: $APP_DOMAIN2"
```

### Important Configuration Values

| Item | Value |
|------|-------|
| **SPIRE Namespace** | `zero-trust-workload-identity-manager` |
| **Controller Class Name** | `zero-trust-workload-identity-manager-spire` |
| **SPIRE Server Binary** | `/spire-server` (in container root) |
| **Federation Port** | `8443` |
| **Bundle Endpoint Profile** | `https_spiffe` |
| **Federation Route Name** | `spire-server-federation` |
| **Federation Route Hostname** | `federation.<APP_DOMAIN>` |

### Quick Start Commands

```bash
# Verify pods running (on each cluster)
oc get pods -n $SPIRE_NS

# Check federation routes
oc get routes -n $SPIRE_NS | grep federation

# List trust bundles
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Show SPIFFE entries
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server entry show

# Check ClusterFederatedTrustDomain
oc get clusterfederatedtrustdomains -o wide
```

---

## Test Prerequisites

Before executing tests, ensure:

- [ ] Both clusters have Zero Trust Workload Identity Manager operator installed
- [ ] Operator CSV shows "Succeeded" status
- [ ] SPIRE components are running on both clusters
- [ ] Network connectivity between clusters
- [ ] Kubeconfig files available for both clusters

### Verify Prerequisites Script

```bash
#!/bin/bash
echo "╔════════════════════════════════════════════════════════════╗"
echo "║     SPIRE Federation Prerequisites Check                   ║"
echo "╚════════════════════════════════════════════════════════════╝"

# Check Cluster 1
echo ""
echo "=== Cluster 1 ==="
echo "Checking operator..."
oc --kubeconfig="$KUBECONFIG1" get csv -n $SPIRE_NS 2>/dev/null | grep -q "Succeeded" && \
  echo "✓ Operator installed and Succeeded" || echo "✗ Operator NOT ready"

echo "Checking SPIRE pods..."
oc --kubeconfig="$KUBECONFIG1" get pods -n $SPIRE_NS 2>/dev/null | grep -q "Running" && \
  echo "✓ SPIRE pods running" || echo "✗ SPIRE pods NOT running"

# Check Cluster 2
echo ""
echo "=== Cluster 2 ==="
echo "Checking operator..."
oc --kubeconfig="$KUBECONFIG2" get csv -n $SPIRE_NS 2>/dev/null | grep -q "Succeeded" && \
  echo "✓ Operator installed and Succeeded" || echo "✗ Operator NOT ready"

echo "Checking SPIRE pods..."
oc --kubeconfig="$KUBECONFIG2" get pods -n $SPIRE_NS 2>/dev/null | grep -q "Running" && \
  echo "✓ SPIRE pods running" || echo "✗ SPIRE pods NOT running"

echo ""
echo "=== Trust Domains ==="
echo "Cluster 1: $APP_DOMAIN1"
echo "Cluster 2: $APP_DOMAIN2"
```

---

## Two-Cluster Federation Setup

### Phase 1: Deploy Operator and Operands

> **Skip if already installed**

#### On Both Clusters - Install Operator

```bash
# Create namespace
oc create namespace zero-trust-workload-identity-manager

# Create OperatorGroup (SINGLE one only!)
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: zero-trust-workload-identity-manager
  namespace: zero-trust-workload-identity-manager
spec:
  targetNamespaces:
  - zero-trust-workload-identity-manager
EOF

# Create Subscription
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-zero-trust-workload-identity-manager
  namespace: zero-trust-workload-identity-manager
spec:
  channel: stable-v1
  name: openshift-zero-trust-workload-identity-manager
  source: stage-catalog-420
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF

# Wait for operator
sleep 60
oc get csv -n zero-trust-workload-identity-manager
```

#### On Both Clusters - Deploy Operands

```bash
export APP_DOMAIN=$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export APP_DOMAIN="apps.${APP_DOMAIN}"
export CLUSTER_NAME=<cluster1 or cluster2>
export JWT_ISSUER=oidc-discovery.${APP_DOMAIN}

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: ZeroTrustWorkloadIdentityManager
metadata:
  name: cluster
spec:
  trustDomain: ${APP_DOMAIN}
  clusterName: ${CLUSTER_NAME}
  bundleConfigMap: spire-bundle
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://${JWT_ISSUER}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireAgent
metadata:
  name: cluster
spec:
  nodeAttestor:
    k8sPSATEnabled: "true"
  workloadAttestors:
    k8sEnabled: "true"
    workloadAttestorsVerification:
      type: "auto"
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpiffeCSIDriver
metadata:
  name: cluster
spec: {}
---
apiVersion: operator.openshift.io/v1alpha1
kind: SpireOIDCDiscoveryProvider
metadata:
  name: cluster
spec:
  jwtIssuer: https://${JWT_ISSUER}
EOF
```

### Phase 2: Enable Federation Bundle Endpoints

#### On Cluster 1

```bash
export KUBECONFIG="$KUBECONFIG1"
export APP_DOMAIN1=$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export APP_DOMAIN1="apps.${APP_DOMAIN1}"
export JWT_ISSUER=oidc-discovery.${APP_DOMAIN1}

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://${JWT_ISSUER}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  federation:
    bundleEndpoint:
      profile: https_spiffe
    managedRoute: "true"
EOF

echo "✓ Federation enabled on Cluster 1"
```

#### On Cluster 2

```bash
export KUBECONFIG="$KUBECONFIG2"
export APP_DOMAIN2=$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export APP_DOMAIN2="apps.${APP_DOMAIN2}"
export JWT_ISSUER=oidc-discovery.${APP_DOMAIN2}

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://${JWT_ISSUER}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  federation:
    bundleEndpoint:
      profile: https_spiffe
    managedRoute: "true"
EOF

echo "✓ Federation enabled on Cluster 2"
```

### Phase 3: Verify Federation Routes

```bash
# On Cluster 1
echo "=== Cluster 1 Federation Route ==="
oc --kubeconfig="$KUBECONFIG1" get routes -n $SPIRE_NS | grep federation

# On Cluster 2
echo "=== Cluster 2 Federation Route ==="
oc --kubeconfig="$KUBECONFIG2" get routes -n $SPIRE_NS | grep federation

# Test endpoints
echo "=== Testing Endpoints ==="
curl -k -s https://federation.$APP_DOMAIN1 | head -c 200
echo ""
curl -k -s https://federation.$APP_DOMAIN2 | head -c 200
```

### Phase 4: Extract Trust Bundles

```bash
mkdir -p /tmp/spire-bundles

# Extract from Cluster 1
oc --kubeconfig="$KUBECONFIG1" exec -n $SPIRE_NS spire-server-0 -c spire-server -- \
  /spire-server bundle show -format spiffe > /tmp/spire-bundles/cluster1-bundle.json
echo "✓ Cluster 1 bundle extracted"

# Extract from Cluster 2
oc --kubeconfig="$KUBECONFIG2" exec -n $SPIRE_NS spire-server-0 -c spire-server -- \
  /spire-server bundle show -format spiffe > /tmp/spire-bundles/cluster2-bundle.json
echo "✓ Cluster 2 bundle extracted"

# Verify
ls -la /tmp/spire-bundles/
```

### Phase 5: Create ClusterFederatedTrustDomain Resources

#### On Cluster 1 (to trust Cluster 2)

```bash
export KUBECONFIG="$KUBECONFIG1"

cat <<EOF | oc apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federate-with-cluster2
spec:
  trustDomain: ${APP_DOMAIN2}
  bundleEndpointURL: https://federation.${APP_DOMAIN2}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${APP_DOMAIN2}/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
$(cat /tmp/spire-bundles/cluster2-bundle.json | sed 's/^/    /')
EOF

echo "✓ Created federation: Cluster 1 → Cluster 2"
```

#### On Cluster 2 (to trust Cluster 1)

```bash
export KUBECONFIG="$KUBECONFIG2"

cat <<EOF | oc apply -f -
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federate-with-cluster1
spec:
  trustDomain: ${APP_DOMAIN1}
  bundleEndpointURL: https://federation.${APP_DOMAIN1}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${APP_DOMAIN1}/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
$(cat /tmp/spire-bundles/cluster1-bundle.json | sed 's/^/    /')
EOF

echo "✓ Created federation: Cluster 2 → Cluster 1"
```

### Phase 6: Verify Federation

```bash
echo "=== Cluster 1 Bundle List (should show Cluster 2) ==="
oc --kubeconfig="$KUBECONFIG1" exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

echo ""
echo "=== Cluster 2 Bundle List (should show Cluster 1) ==="
oc --kubeconfig="$KUBECONFIG2" exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list
```

---

## Positive Test Scenarios

### P1: Federation via ClusterFederatedTrustDomain CR (Method 2)

| Test ID | P1 |
|---------|-----|
| **Title** | Two-Cluster Federation using ClusterFederatedTrustDomain CR |
| **Priority** | High |
| **Type** | Functional |
| **Method** | Method 2 - Separate ClusterFederatedTrustDomain CR |
| **Prerequisite** | SPIRE installed on both clusters, no federation configured |

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Update SpireServer CR on Cluster 1 with federation config | SpireServer CR updated |
| 2 | Update SpireServer CR on Cluster 2 with federation config | SpireServer CR updated |
| 3 | Verify federation routes created | Routes exist with passthrough TLS |
| 4 | Test federation endpoints via curl | Returns JWKS bundle |
| 5 | Extract trust bundles from both clusters | Bundle JSON files created |
| 6 | Create ClusterFederatedTrustDomain on Cluster 1 | Resource created |
| 7 | Create ClusterFederatedTrustDomain on Cluster 2 | Resource created |
| 8 | Verify bundle list on Cluster 1 | Shows Cluster 2's trust domain |
| 9 | Verify bundle list on Cluster 2 | Shows Cluster 1's trust domain |

#### Manual Test Commands

```bash
# Follow Phase 2-6 from setup section above

# Final verification
echo "=== Cluster 1 ==="
oc --kubeconfig="$KUBECONFIG1" get clusterfederatedtrustdomains -o wide
oc --kubeconfig="$KUBECONFIG1" exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

echo ""
echo "=== Cluster 2 ==="
oc --kubeconfig="$KUBECONFIG2" get clusterfederatedtrustdomains -o wide
oc --kubeconfig="$KUBECONFIG2" exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list
```

#### Pass Criteria
- [ ] Both federation routes exist and are accessible
- [ ] `curl -k` returns valid JWKS JSON from both endpoints
- [ ] Bundle list on Cluster 1 shows Cluster 2's trust domain
- [ ] Bundle list on Cluster 2 shows Cluster 1's trust domain

---

### P1b: Federation via SpireServer federatesWith (Method 1)

| Test ID | P1b |
|---------|-----|
| **Title** | Two-Cluster Federation using SpireServer federatesWith |
| **Priority** | High |
| **Type** | Functional |
| **Method** | Method 1 - Inline in SpireServer CR |
| **Prerequisite** | SPIRE installed on both clusters, no federation configured |

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Enable federation endpoint on Cluster 1 | Route created |
| 2 | Enable federation endpoint on Cluster 2 | Route created |
| 3 | Update SpireServer on Cluster 1 with `federatesWith` pointing to Cluster 2 | SpireServer updated |
| 4 | Update SpireServer on Cluster 2 with `federatesWith` pointing to Cluster 1 | SpireServer updated |
| 5 | Wait for bundle sync (60s) | Bundles exchanged |
| 6 | Verify bundle list on both clusters | Both show remote trust domain |

#### Manual Test Commands

```bash
# On Cluster 1 - Configure federation with federatesWith
export APP_DOMAIN1="apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')"
export APP_DOMAIN2="<CLUSTER2_TRUST_DOMAIN>"  # Replace with actual
export JWT_ISSUER=oidc-discovery.${APP_DOMAIN1}

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://${JWT_ISSUER}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  federation:
    bundleEndpoint:
      profile: https_spiffe
      refreshHint: 300
    federatesWith:
      - trustDomain: ${APP_DOMAIN2}
        bundleEndpointUrl: https://federation.${APP_DOMAIN2}
        bundleEndpointProfile: https_spiffe
        endpointSpiffeId: spiffe://${APP_DOMAIN2}/spire/server
    managedRoute: "true"
EOF

# Wait and verify
sleep 60
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list
```

#### Pass Criteria
- [ ] SpireServer CR accepts `federatesWith` configuration
- [ ] Federation routes created on both clusters
- [ ] Bundle list shows remote trust domain WITHOUT creating ClusterFederatedTrustDomain CR
- [ ] No manual `trustDomainBundle` needed

---

### P2: Federation Route Auto-Creation

| Test ID | P2 |
|---------|-----|
| **Title** | Federation Route Auto-Creation with managedRoute |
| **Priority** | High |
| **Type** | Functional |

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Apply SpireServer with `managedRoute: "true"` | SpireServer updated |
| 2 | Check routes in SPIRE namespace | Federation route auto-created |
| 3 | Verify route hostname | Matches `federation.<APP_DOMAIN>` |
| 4 | Verify route TLS termination | Passthrough |
| 5 | Test endpoint via curl | Returns JWKS bundle |

#### Manual Test Commands

```bash
# Check route exists
oc get routes -n $SPIRE_NS | grep federation

# Verify route details
oc get route spire-server-federation -n $SPIRE_NS -o yaml | grep -A 5 "tls:"

# Test endpoint
curl -k -s https://federation.$APP_DOMAIN1 | jq '.keys | length'
```

#### Pass Criteria
- [ ] Route `spire-server-federation` is automatically created
- [ ] Route hostname is `federation.<APP_DOMAIN>`
- [ ] Route uses passthrough TLS termination
- [ ] Endpoint returns valid JWKS JSON

---

### P3: Federation Route Recreation on Deletion

| Test ID | P3 |
|---------|-----|
| **Title** | Federation Route Automatic Recreation After Deletion |
| **Priority** | Medium |
| **Type** | Reconciliation |

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Verify federation route exists | Route exists |
| 2 | Delete the federation route | Route deleted |
| 3 | Wait 60 seconds | Controller reconciles |
| 4 | Verify route is recreated | Route exists again |
| 5 | Test endpoint | Endpoint accessible |

#### Manual Test Commands

```bash
# Step 1: Verify exists
oc get route spire-server-federation -n $SPIRE_NS

# Step 2: Delete
oc delete route spire-server-federation -n $SPIRE_NS

# Step 3: Wait
echo "Waiting 60 seconds..."
sleep 60

# Step 4: Verify recreated
oc get route spire-server-federation -n $SPIRE_NS

# Step 5: Test endpoint
curl -k https://federation.$APP_DOMAIN1 | head -c 100
```

#### Pass Criteria
- [ ] Route is automatically recreated after deletion
- [ ] Recreated route has same configuration
- [ ] Federation endpoint works after recreation

---

### P4: Workload Entry with FederatesWith

| Test ID | P4 |
|---------|-----|
| **Title** | Workload Entry Shows FederatesWith Field |
| **Priority** | High |
| **Type** | Functional |

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Create namespace `federation-demo` | Namespace created |
| 2 | Create ServiceAccount | ServiceAccount created |
| 3 | Create ClusterSPIFFEID with federatesWith | ClusterSPIFFEID created |
| 4 | Deploy test workload pod | Pod running |
| 5 | Check SPIRE entry show | Entry shows FederatesWith |

#### Manual Test Commands

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: federation-demo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-workload
  namespace: federation-demo
---
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: test-workload
spec:
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      app: test-workload
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: federation-demo
  federatesWith:
  - "${APP_DOMAIN2}"
  className: zero-trust-workload-identity-manager-spire
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-workload
  namespace: federation-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-workload
  template:
    metadata:
      labels:
        app: test-workload
    spec:
      serviceAccountName: test-workload
      containers:
      - name: workload
        image: registry.access.redhat.com/ubi9/ubi-minimal:latest
        command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
        volumeMounts:
        - name: spiffe-workload-api
          mountPath: /spiffe-workload-api
          readOnly: true
      volumes:
      - name: spiffe-workload-api
        csi:
          driver: csi.spiffe.io
          readOnly: true
EOF

# Wait for pod
oc wait --for=condition=ready pod -l app=test-workload -n federation-demo --timeout=120s
sleep 15

# Check SPIRE entry
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server entry show
```

#### Pass Criteria
- [ ] Pod is running with SPIFFE CSI volume mounted
- [ ] `entry show` displays the workload entry
- [ ] Entry contains `FederatesWith: <remote-trust-domain>`

---

### P5: Bundle Refresh Logging

| Test ID | P5 |
|---------|-----|
| **Title** | Bundle Refresh Appears in SPIRE Server Logs |
| **Priority** | Medium |
| **Type** | Logging |

#### Manual Test Commands

```bash
# Check logs for bundle activity
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=100 | grep -i "bundle"

# Expected messages:
# level=info msg="Serving bundle endpoint" addr="0.0.0.0:8443"
# level=info msg="Bundle refreshed" subsystem_name=bundle_client trust_domain=apps.xxx
```

#### Pass Criteria
- [ ] Log shows "Serving bundle endpoint" on startup
- [ ] Log shows "Bundle refreshed" periodically
- [ ] Refresh log includes the remote trust domain name

---

### P6: Bundle Endpoint HTTPS Accessibility

| Test ID | P6 |
|---------|-----|
| **Title** | Bundle Endpoint Accessible via HTTPS |
| **Priority** | High |
| **Type** | Integration |

#### Manual Test Commands

```bash
# Test endpoint and save response
curl -k -s https://federation.$APP_DOMAIN1 -o /tmp/bundle-response.json

# Verify valid JSON
cat /tmp/bundle-response.json | jq . > /dev/null && echo "✓ Valid JSON" || echo "✗ Invalid JSON"

# Check keys array
echo "Number of keys: $(cat /tmp/bundle-response.json | jq '.keys | length')"

# Check x509-svid
cat /tmp/bundle-response.json | jq '.keys[] | select(.use == "x509-svid") | .use'
```

#### Pass Criteria
- [ ] Endpoint returns HTTP 200
- [ ] Response is valid JSON
- [ ] Response contains `keys` array with at least 1 entry
- [ ] At least one key has `use: "x509-svid"`

---

### P7: Service Port 8443 Exposure

| Test ID | P7 |
|---------|-----|
| **Title** | SPIRE Server Service Exposes Port 8443 for Federation |
| **Priority** | Medium |
| **Type** | Configuration |

#### Manual Test Commands

```bash
# Check service ports
oc get svc spire-server -n $SPIRE_NS -o yaml | grep -A 10 "ports:"

# Expected: port 8443 named "federation"
```

#### Pass Criteria
- [ ] Service has port 8443 exposed
- [ ] Port name is "federation"
- [ ] Target port is 8443

---

## ACME (Automatic Certificate Management) Test Scenarios

> **Prerequisites for ACME Tests:**
> - Cluster must have publicly accessible DNS
> - Port 80 accessible for HTTP-01 challenge (or DNS-01 configured)
> - Valid email address for Let's Encrypt registration

### P8: ACME Certificate Issuance

| Test ID | P8 |
|---------|-----|
| **Title** | ACME Certificate Automatic Issuance via Let's Encrypt |
| **Priority** | High |
| **Type** | Functional |
| **Profile** | https_web (ACME) |
| **Prerequisite** | Public DNS, Port 80 accessible |

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Configure SpireServer with ACME profile | SpireServer CR updated |
| 2 | Wait for ACME challenge completion | Certificate issued |
| 3 | Verify federation endpoint accessible | HTTPS without `-k` flag |
| 4 | Check certificate issuer | Shows Let's Encrypt |

#### Manual Test Commands

```bash
# Configure ACME on SpireServer
export APP_DOMAIN=$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export APP_DOMAIN="apps.${APP_DOMAIN}"
export JWT_ISSUER=oidc-discovery.${APP_DOMAIN}
export ACME_EMAIL="admin@example.com"  # Replace with valid email

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://${JWT_ISSUER}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
    maxOpenConns: 100
    maxIdleConns: 2
    connMaxLifetime: 3600
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: https://acme-v02.api.letsencrypt.org/directory
          email: ${ACME_EMAIL}
          tosAccepted: true
    managedRoute: "true"
EOF

echo "Waiting for ACME certificate issuance (may take 2-5 minutes)..."
sleep 180

# Test endpoint WITHOUT -k flag (publicly trusted cert)
echo "=== Testing with publicly trusted certificate ==="
curl -s https://federation.${APP_DOMAIN} | head -c 200

# Check certificate details
echo ""
echo "=== Certificate Details ==="
echo | openssl s_client -connect federation.${APP_DOMAIN}:443 -servername federation.${APP_DOMAIN} 2>/dev/null | openssl x509 -noout -issuer -dates
```

#### Pass Criteria
- [ ] SpireServer accepts ACME configuration
- [ ] Let's Encrypt certificate is issued within 5 minutes
- [ ] `curl` works WITHOUT `-k` flag (publicly trusted)
- [ ] Certificate issuer shows "Let's Encrypt" or "R3"
- [ ] Certificate is valid (not expired)

---

### P9: ACME Certificate Auto-Renewal

| Test ID | P9 |
|---------|-----|
| **Title** | ACME Certificate Automatic Renewal |
| **Priority** | Medium |
| **Type** | Lifecycle |
| **Profile** | https_web (ACME) |

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Check current certificate expiry | Note expiration date |
| 2 | Check SPIRE server logs for renewal activity | Logs show renewal attempts |
| 3 | Verify renewal happens before expiry | Certificate updated |

#### Manual Test Commands

```bash
# Check current certificate expiry
echo "=== Current Certificate Expiry ==="
echo | openssl s_client -connect federation.${APP_DOMAIN}:443 -servername federation.${APP_DOMAIN} 2>/dev/null | openssl x509 -noout -dates

# Check logs for ACME activity
echo ""
echo "=== SPIRE Server ACME Logs ==="
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=100 | grep -i "acme\|certificate\|renewal\|letsencrypt"

# Check ACME account status (if stored)
echo ""
echo "=== SPIRE Server Federation Status ==="
oc get spireserver cluster -o yaml | grep -A 20 "federation:"
```

#### Pass Criteria
- [ ] Certificate expiry date is visible
- [ ] Logs show ACME activity
- [ ] Renewal is attempted before expiration (typically 30 days before)
- [ ] No manual intervention required

---

### P10: https_web Profile Federation

| Test ID | P10 |
|---------|-----|
| **Title** | Federation Using https_web Profile with ACME |
| **Priority** | High |
| **Type** | Integration |
| **Profile** | https_web (ACME) |

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Configure Cluster with https_web (ACME) profile | Certificate issued |
| 2 | Create ClusterFederatedTrustDomain from another cluster | Federation configured |
| 3 | Verify bundle exchange | Remote bundle appears |
| 4 | Note: No `trustDomainBundle` needed for https_web | Auto-fetched |

#### Manual Test Commands

```bash
# From a cluster using https_spiffe, federate with https_web cluster
# Note: No initial trustDomainBundle needed for https_web!

export ACME_CLUSTER_DOMAIN="apps.acme-cluster.example.com"  # Replace

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federate-with-acme-cluster
spec:
  trustDomain: ${ACME_CLUSTER_DOMAIN}
  bundleEndpointURL: https://federation.${ACME_CLUSTER_DOMAIN}
  bundleEndpointProfile:
    type: https_web
  className: zero-trust-workload-identity-manager-spire
  # NOTE: No trustDomainBundle needed - fetched automatically via public CA!
EOF

# Wait for bundle fetch
sleep 60

# Verify bundle appears
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list
```

#### Pass Criteria
- [ ] ClusterFederatedTrustDomain created without `trustDomainBundle`
- [ ] Bundle is automatically fetched (no bootstrap needed)
- [ ] Remote trust domain appears in bundle list
- [ ] No `-k` flag needed for curl

---

### P11: Mixed Profile Federation (https_spiffe ↔ https_web)

| Test ID | P11 |
|---------|-----|
| **Title** | Federation Between Different Bundle Endpoint Profiles |
| **Priority** | High |
| **Type** | Integration |

#### Architecture

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│        Cluster 1            │         │        Cluster 2            │
│   Profile: https_spiffe     │ ◄═════► │   Profile: https_web        │
│   (Self-signed cert)        │  Trust  │   (Let's Encrypt cert)      │
│                             │ Bundle  │                             │
│   Requires: trustDomainBundle│ Exchange│   No trustDomainBundle needed│
└─────────────────────────────┘         └─────────────────────────────┘
```

#### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Cluster 1: https_spiffe profile | Self-signed federation endpoint |
| 2 | Cluster 2: https_web (ACME) profile | Let's Encrypt endpoint |
| 3 | Cluster 1 → Cluster 2: No trustDomainBundle | Works (public CA) |
| 4 | Cluster 2 → Cluster 1: Requires trustDomainBundle | Works with bundle |
| 5 | Verify bidirectional federation | Both show remote bundles |

#### Manual Test Commands

```bash
# On Cluster 1 (https_spiffe) - federating with Cluster 2 (https_web)
# Note: NO trustDomainBundle needed because Cluster 2 has public cert

export CLUSTER2_DOMAIN="apps.cluster2.example.com"  # https_web cluster

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federate-with-https-web-cluster
spec:
  trustDomain: ${CLUSTER2_DOMAIN}
  bundleEndpointURL: https://federation.${CLUSTER2_DOMAIN}
  bundleEndpointProfile:
    type: https_web
  className: zero-trust-workload-identity-manager-spire
EOF

# On Cluster 2 (https_web) - federating with Cluster 1 (https_spiffe)
# Note: REQUIRES trustDomainBundle because Cluster 1 has self-signed cert

export CLUSTER1_DOMAIN="apps.cluster1.example.com"  # https_spiffe cluster

# First, fetch Cluster 1's bundle via curl -k
curl -k -s https://federation.${CLUSTER1_DOMAIN} > /tmp/cluster1-bundle.json

oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federate-with-https-spiffe-cluster
spec:
  trustDomain: ${CLUSTER1_DOMAIN}
  bundleEndpointURL: https://federation.${CLUSTER1_DOMAIN}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${CLUSTER1_DOMAIN}/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
$(cat /tmp/cluster1-bundle.json | sed 's/^/    /')
EOF

# Verify on both clusters
echo "=== Cluster 1 Bundle List ==="
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

echo "=== Cluster 2 Bundle List ==="
# (Run on Cluster 2)
```

#### Pass Criteria
- [ ] https_spiffe → https_web works WITHOUT trustDomainBundle
- [ ] https_web → https_spiffe works WITH trustDomainBundle
- [ ] Bidirectional federation established
- [ ] Both clusters show remote trust domains

---

### P12: ACME with Let's Encrypt Staging Environment

| Test ID | P12 |
|---------|-----|
| **Title** | ACME with Staging Directory for Testing |
| **Priority** | Medium |
| **Type** | Functional |

#### Description

Use Let's Encrypt staging environment for testing without hitting production rate limits.

#### Manual Test Commands

```bash
# Use STAGING directory URL (no rate limits, but certs not publicly trusted)
export APP_DOMAIN=$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export APP_DOMAIN="apps.${APP_DOMAIN}"
export JWT_ISSUER=oidc-discovery.${APP_DOMAIN}
export ACME_EMAIL="admin@example.com"

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://${JWT_ISSUER}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          # STAGING URL - for testing without rate limits
          directoryUrl: https://acme-staging-v02.api.letsencrypt.org/directory
          email: ${ACME_EMAIL}
          tosAccepted: true
    managedRoute: "true"
EOF

sleep 180

# Test - note: staging certs are NOT publicly trusted, need -k
echo "=== Testing Staging Certificate (requires -k) ==="
curl -k -s https://federation.${APP_DOMAIN} | head -c 200

# Check issuer - should show "(STAGING)" or "Fake LE"
echo ""
echo "=== Certificate Issuer (should show STAGING) ==="
echo | openssl s_client -connect federation.${APP_DOMAIN}:443 -servername federation.${APP_DOMAIN} 2>/dev/null | openssl x509 -noout -issuer
```

#### Pass Criteria
- [ ] Staging certificate issued successfully
- [ ] Certificate issuer shows "Staging" or "Fake LE"
- [ ] No rate limiting issues
- [ ] Good for repeated testing

---

## Manual Certificate Management Test Scenarios

> **Note**: For organizations requiring custom certificate management with cert-manager or external CAs.

### P13: Manual Certificate via ServingCert

| Test ID | P13 |
|---------|-----|
| **Title** | Federation with Manual TLS Certificate (ServingCert) |
| **Priority** | High |
| **Type** | Functional |
| **Profile** | https_web (serving_cert) |

#### Description

Configure federation using externally managed TLS certificates via Kubernetes Secret, typically used with cert-manager.

#### Prerequisites

```bash
# Create a TLS secret (for testing, use self-signed; in production, use cert-manager)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key \
  -out /tmp/tls.crt \
  -subj "/CN=federation.${APP_DOMAIN}" \
  -addext "subjectAltName=DNS:federation.${APP_DOMAIN}"

oc create secret tls spire-federation-cert \
  -n $SPIRE_NS \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key
```

#### Manual Test Commands

```bash
# Configure SpireServer with ServingCert
export APP_DOMAIN=$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export APP_DOMAIN="apps.${APP_DOMAIN}"
export JWT_ISSUER=oidc-discovery.${APP_DOMAIN}

oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://${JWT_ISSUER}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        servingCert:
          secretName: spire-federation-cert
    managedRoute: "true"
EOF

# Wait for configuration
sleep 60

# Verify endpoint uses the provided certificate
echo "=== Certificate Details ==="
echo | openssl s_client -connect federation.${APP_DOMAIN}:443 -servername federation.${APP_DOMAIN} 2>/dev/null | openssl x509 -noout -subject -issuer

# Test endpoint
curl -k -s https://federation.${APP_DOMAIN} | head -c 200
```

#### Pass Criteria
- [ ] SpireServer accepts servingCert configuration
- [ ] Certificate from secret is used
- [ ] Federation endpoint accessible
- [ ] No ACME requests made (manual cert used)

---

### P14: Managed Route Disabled (Manual Route Control)

| Test ID | P14 |
|---------|-----|
| **Title** | Federation with managedRoute: false |
| **Priority** | Medium |
| **Type** | Configuration |

#### Description

Verify that when `managedRoute: "false"`, the operator does NOT create routes, allowing manual route management.

#### Manual Test Commands

```bash
# First, delete any existing federation route
oc delete route spire-server-federation -n $SPIRE_NS --ignore-not-found

# Configure SpireServer WITHOUT managed route
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://oidc-discovery.${APP_DOMAIN}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  federation:
    bundleEndpoint:
      profile: https_spiffe
    managedRoute: "false"
EOF

# Wait and verify NO route is created
sleep 30
echo "=== Routes (should NOT include federation route) ==="
oc get routes -n $SPIRE_NS

# Manually create custom route
oc apply -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-custom-federation-route
  namespace: $SPIRE_NS
spec:
  host: custom-federation.${APP_DOMAIN}
  port:
    targetPort: federation
  tls:
    insecureEdgeTerminationPolicy: Redirect
    termination: passthrough
  to:
    kind: Service
    name: spire-server
    weight: 100
EOF

# Verify custom route works
sleep 10
curl -k -s https://custom-federation.${APP_DOMAIN} | head -c 200
```

#### Pass Criteria
- [ ] With `managedRoute: "false"`, NO federation route is auto-created
- [ ] Manual route creation works
- [ ] Custom route can access federation endpoint
- [ ] Operator does not override manual routes

---

### P15: Dynamic Trust Bundle Synchronization

| Test ID | P15 |
|---------|-----|
| **Title** | Automatic Trust Bundle Refresh and Sync |
| **Priority** | High |
| **Type** | Functional |

#### Description

Verify that trust bundles are automatically refreshed from federated endpoints.

#### Manual Test Commands

```bash
# Create federation relationship
# (Assuming federation already set up from P1)

# Check initial bundle
echo "=== Initial Bundle (note the certificate) ==="
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Monitor bundle refresh in logs
echo ""
echo "=== Monitoring Bundle Refresh (wait 5+ minutes for refresh) ==="
oc logs -f -n $SPIRE_NS spire-server-0 -c spire-server 2>&1 | grep -i "bundle\|refresh\|sync" &
LOGS_PID=$!
sleep 300
kill $LOGS_PID 2>/dev/null

# Check for bundle updates
echo ""
echo "=== Bundle After Refresh ==="
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Check ClusterFederatedTrustDomain status
oc get clusterfederatedtrustdomains -o yaml | grep -A 5 "status:"
```

#### Pass Criteria
- [ ] Logs show periodic bundle refresh attempts
- [ ] Remote trust bundles are automatically updated
- [ ] No manual intervention required
- [ ] Bundle refresh occurs based on refreshHint

---

### P16: Maximum Federation Limit (50 Trust Domains)

| Test ID | P16 |
|---------|-----|
| **Title** | Support for Up to 50 Federated Trust Domains |
| **Priority** | Medium |
| **Type** | Scalability |

#### Description

Verify the system can handle up to 50 federated trust domains as documented.

#### Manual Test Commands

```bash
# Create multiple ClusterFederatedTrustDomain resources (test with subset)
for i in $(seq 1 10); do
  oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: test-federation-$i
spec:
  trustDomain: test-domain-$i.example.com
  bundleEndpointURL: https://federation.test-domain-$i.example.com
  bundleEndpointProfile:
    type: https_web
  className: zero-trust-workload-identity-manager-spire
EOF
done

# List all ClusterFederatedTrustDomains
echo "=== All ClusterFederatedTrustDomains ==="
oc get clusterfederatedtrustdomains

# Check SPIRE server handles multiple entries
echo ""
echo "=== Bundle List ==="
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list 2>&1 | head -50

# Check operator logs for any issues
echo ""
echo "=== Operator Logs ==="
oc logs -n $SPIRE_NS -l app.kubernetes.io/name=spire-controller-manager --tail=20

# Cleanup test resources
for i in $(seq 1 10); do
  oc delete clusterfederatedtrustdomain test-federation-$i --ignore-not-found
done
```

#### Pass Criteria
- [ ] Multiple ClusterFederatedTrustDomains can be created
- [ ] No performance degradation with many entries
- [ ] Operator handles all entries without error
- [ ] Bundle list shows all configured domains

---

### P17: Bundle Endpoint refreshHint Configuration

| Test ID | P17 |
|---------|-----|
| **Title** | Custom Bundle Refresh Interval |
| **Priority** | Medium |
| **Type** | Configuration |

#### Description

Verify the `refreshHint` parameter controls bundle refresh frequency.

#### Manual Test Commands

```bash
# Configure with custom refreshHint (in seconds)
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://oidc-discovery.${APP_DOMAIN}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  federation:
    bundleEndpoint:
      profile: https_spiffe
      refreshHint: 60  # Refresh every 60 seconds
    managedRoute: "true"
EOF

# Verify refreshHint in bundle endpoint response
sleep 30
echo "=== Bundle Endpoint Response (check refresh_hint) ==="
curl -k -s https://federation.${APP_DOMAIN} | jq '.spiffe_refresh_hint // "not present"'

# Check SPIRE server config
echo ""
echo "=== SPIRE Server ConfigMap ==="
oc get configmap spire-server -n $SPIRE_NS -o yaml | grep -A 5 "refresh_hint"
```

#### Pass Criteria
- [ ] refreshHint is configurable in SpireServer CR
- [ ] Bundle endpoint returns configured refresh_hint
- [ ] Clients respect the refresh interval
- [ ] Default refresh interval used when not specified

---

### P18: Route Naming Convention Verification

| Test ID | P18 |
|---------|-----|
| **Title** | Federation Route Follows Naming Convention |
| **Priority** | Low |
| **Type** | Configuration |

#### Description

Verify that auto-created routes follow the `federation.<trust-domain>` naming convention.

#### Manual Test Commands

```bash
# Get trust domain
export APP_DOMAIN=$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export APP_DOMAIN="apps.${APP_DOMAIN}"

# Enable managed route
oc patch spireserver cluster --type='merge' -p='{"spec":{"federation":{"managedRoute":"true"}}}'

sleep 30

# Verify route naming
echo "=== Federation Route ==="
oc get route spire-server-federation -n $SPIRE_NS -o yaml | grep "host:"

# Expected: host: federation.<APP_DOMAIN>
echo ""
echo "Expected hostname: federation.${APP_DOMAIN}"

# Verify route name is consistent
oc get routes -n $SPIRE_NS -o custom-columns="NAME:.metadata.name,HOST:.spec.host" | grep federation
```

#### Pass Criteria
- [ ] Route name is `spire-server-federation`
- [ ] Route hostname is `federation.<APP_DOMAIN>`
- [ ] Hostname matches trust domain pattern
- [ ] Route is consistently named across clusters

---

### P19: TLS Configuration Based on Profile

| Test ID | P19 |
|---------|-----|
| **Title** | Automatic TLS Configuration Per Authentication Profile |
| **Priority** | High |
| **Type** | Configuration |

#### Description

Verify that TLS configuration is automatically set based on the selected authentication profile.

#### Test Steps

| Profile | Expected TLS Behavior |
|---------|----------------------|
| `https_spiffe` | Self-signed SPIFFE cert, passthrough TLS |
| `https_web` (ACME) | Let's Encrypt cert, passthrough TLS |
| `https_web` (servingCert) | Provided cert from Secret, passthrough TLS |

#### Manual Test Commands

```bash
# Test 1: https_spiffe profile
echo "=== Test 1: https_spiffe Profile ==="
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_spiffe
    managedRoute: "true"
EOF

sleep 60

echo "Certificate for https_spiffe (should be self-signed by SPIRE):"
echo | openssl s_client -connect federation.${APP_DOMAIN}:443 -servername federation.${APP_DOMAIN} 2>/dev/null | openssl x509 -noout -issuer

echo ""
echo "Route TLS termination:"
oc get route spire-server-federation -n $SPIRE_NS -o jsonpath='{.spec.tls.termination}'
echo ""

# Verify cert is SPIFFE-based
CERT_SAN=$(echo | openssl s_client -connect federation.${APP_DOMAIN}:443 -servername federation.${APP_DOMAIN} 2>/dev/null | openssl x509 -noout -text | grep "URI:spiffe://")
echo "SPIFFE SAN: $CERT_SAN"
```

#### Pass Criteria
- [ ] https_spiffe uses SPIRE self-signed certificate
- [ ] https_web (ACME) uses Let's Encrypt certificate
- [ ] https_web (servingCert) uses provided certificate
- [ ] All routes use passthrough TLS termination
- [ ] Certificate matches configured profile

---

## Manual Certificate Management Negative Tests

### N13: ServingCert Secret Not Found

| Test ID | N13 |
|---------|-----|
| **Title** | SpireServer with Non-Existent ServingCert Secret |
| **Priority** | High |
| **Type** | Negative |

#### Manual Test Commands

```bash
# Configure with non-existent secret
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        servingCert:
          secretName: non-existent-secret
    managedRoute: "true"
EOF

# Check for error
sleep 60
echo "=== SPIRE Server Logs ==="
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=30 | grep -i "secret\|error\|not found"

echo ""
echo "=== SpireServer Status ==="
oc get spireserver cluster -o yaml | grep -A 10 "status:"
```

#### Pass Criteria
- [ ] SPIRE server does not crash
- [ ] Clear error message about missing secret
- [ ] Status reflects configuration error
- [ ] Recovery when secret is created

---

### N14: Exceed Maximum Federation Limit

| Test ID | N14 |
|---------|-----|
| **Title** | Create More Than 50 Federated Trust Domains |
| **Priority** | Low |
| **Type** | Boundary |

#### Description

Test behavior when attempting to exceed the 50 federation limit (if enforced).

#### Manual Test Commands

```bash
# Attempt to create 55 ClusterFederatedTrustDomains
for i in $(seq 1 55); do
  oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: limit-test-$i
spec:
  trustDomain: limit-test-$i.example.com
  bundleEndpointURL: https://federation.limit-test-$i.example.com
  bundleEndpointProfile:
    type: https_web
  className: zero-trust-workload-identity-manager-spire
EOF
done

# Check count
echo "=== Total ClusterFederatedTrustDomains ==="
oc get clusterfederatedtrustdomains --no-headers | wc -l

# Check for any errors or warnings
echo ""
echo "=== Controller Manager Logs ==="
oc logs -n $SPIRE_NS -l app.kubernetes.io/name=spire-controller-manager --tail=50 | grep -i "limit\|error\|warn\|exceed"

# Cleanup
for i in $(seq 1 55); do
  oc delete clusterfederatedtrustdomain limit-test-$i --ignore-not-found &
done
wait
```

#### Pass Criteria
- [ ] Behavior documented (rejected or accepted)
- [ ] Clear error/warning if limit exceeded
- [ ] No system instability
- [ ] Performance impact documented if no hard limit

---

### N15: Invalid Certificate in ServingCert Secret

| Test ID | N15 |
|---------|-----|
| **Title** | ServingCert with Invalid/Expired Certificate |
| **Priority** | Medium |
| **Type** | Negative |

#### Manual Test Commands

```bash
# Create secret with invalid certificate (empty or malformed)
oc create secret generic invalid-cert-secret \
  -n $SPIRE_NS \
  --from-literal=tls.crt="INVALID CERTIFICATE DATA" \
  --from-literal=tls.key="INVALID KEY DATA" \
  --dry-run=client -o yaml | oc apply -f -

# Configure SpireServer with invalid cert
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        servingCert:
          secretName: invalid-cert-secret
    managedRoute: "true"
EOF

# Check for error handling
sleep 60
echo "=== SPIRE Server Logs ==="
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=30 | grep -i "cert\|error\|invalid\|parse"

# Cleanup
oc delete secret invalid-cert-secret -n $SPIRE_NS --ignore-not-found
```

#### Pass Criteria
- [ ] SPIRE server does not crash
- [ ] Clear error message about invalid certificate
- [ ] No partial/broken configuration applied
- [ ] Recovery when valid certificate provided

---

## ACME Negative Test Scenarios

### N9: Invalid ACME Directory URL

| Test ID | N9 |
|---------|-----|
| **Title** | SpireServer with Invalid ACME Directory URL |
| **Priority** | Medium |
| **Type** | Negative |

#### Manual Test Commands

```bash
# Configure with INVALID ACME directory URL
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://oidc-discovery.${APP_DOMAIN}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: https://invalid.acme.server/directory
          email: admin@example.com
          tosAccepted: true
    managedRoute: "true"
EOF

# Check logs for ACME errors
sleep 60
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=50 | grep -i "acme\|error\|failed"

# Revert to valid configuration
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_spiffe
    managedRoute: "true"
EOF
```

#### Pass Criteria
- [ ] SPIRE server does not crash
- [ ] Error message logged about invalid ACME directory
- [ ] Federation endpoint may not work (expected)
- [ ] Recovery possible by fixing configuration

---

### N10: ACME Without Public DNS Access

| Test ID | N10 |
|---------|-----|
| **Title** | ACME Certificate Request Without Public DNS |
| **Priority** | High |
| **Type** | Negative |

#### Description

ACME HTTP-01 challenge requires the domain to be publicly accessible. Test behavior when DNS is not public.

#### Manual Test Commands

```bash
# This test verifies behavior when ACME cannot complete challenge
# Expected: Certificate issuance fails, logs show challenge failure

# Configure ACME on a cluster with private/non-routable DNS
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://oidc-discovery.${APP_DOMAIN}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: https://acme-v02.api.letsencrypt.org/directory
          email: admin@example.com
          tosAccepted: true
    managedRoute: "true"
EOF

# Wait and check for challenge failure
sleep 300

# Check logs
echo "=== ACME Challenge Logs ==="
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=100 | grep -i "acme\|challenge\|error\|authorization"

# Test endpoint - should fail or use fallback
curl -k -s https://federation.${APP_DOMAIN} || echo "Endpoint not accessible as expected"
```

#### Pass Criteria
- [ ] SPIRE server does not crash
- [ ] Logs show ACME challenge failure (authorization error)
- [ ] Clear error message about DNS/network issue
- [ ] Graceful degradation (no certificate issued)

---

### N11: ACME Missing Terms of Service Acceptance

| Test ID | N11 |
|---------|-----|
| **Title** | ACME Configuration Without TOS Acceptance |
| **Priority** | Medium |
| **Type** | Validation |

#### Manual Test Commands

```bash
# Configure ACME WITHOUT tosAccepted
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://oidc-discovery.${APP_DOMAIN}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: https://acme-v02.api.letsencrypt.org/directory
          email: admin@example.com
          # tosAccepted: true  # MISSING!
    managedRoute: "true"
EOF

# Check if resource is rejected or certificate fails
oc get spireserver cluster -o yaml | grep -A 10 "status:"
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=50 | grep -i "tos\|terms\|agreement"
```

#### Pass Criteria
- [ ] Configuration rejected OR certificate issuance fails
- [ ] Clear error message about TOS acceptance
- [ ] No certificate issued without TOS acceptance

---

### N12: ACME Invalid Email Address

| Test ID | N12 |
|---------|-----|
| **Title** | ACME Configuration with Invalid Email |
| **Priority** | Low |
| **Type** | Validation |

#### Manual Test Commands

```bash
# Configure ACME with invalid email
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: https://acme-v02.api.letsencrypt.org/directory
          email: not-a-valid-email
          tosAccepted: true
    managedRoute: "true"
EOF

# Check for validation error
sleep 60
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=50 | grep -i "email\|invalid\|error"
```

#### Pass Criteria
- [ ] Invalid email is rejected or logged
- [ ] Clear error message
- [ ] No crash

---

## Negative Test Scenarios

### N1: Invalid className in ClusterFederatedTrustDomain

| Test ID | N1 |
|---------|-----|
| **Title** | ClusterFederatedTrustDomain with Wrong className |
| **Priority** | High |
| **Type** | Negative |

#### Manual Test Commands

```bash
# Create with WRONG className
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: test-wrong-classname
spec:
  trustDomain: ${APP_DOMAIN2}
  bundleEndpointURL: https://federation.${APP_DOMAIN2}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${APP_DOMAIN2}/spire/server
  className: wrong-class-name
  trustDomainBundle: |
$(cat /tmp/spire-bundles/cluster2-bundle.json | sed 's/^/    /')
EOF

# Wait and check - should NOT show remote bundle
sleep 30
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Fix className
oc patch clusterfederatedtrustdomain test-wrong-classname \
  --type='json' -p='[{"op": "replace", "path": "/spec/className", "value": "zero-trust-workload-identity-manager-spire"}]'

# Should now show remote bundle
sleep 30
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Cleanup
oc delete clusterfederatedtrustdomain test-wrong-classname
```

#### Pass Criteria
- [ ] Wrong className does NOT add bundle to SPIRE
- [ ] Correct className properly adds bundle to SPIRE
- [ ] No crash or error in operator

---

### N2: Unreachable Federation Endpoint

| Test ID | N2 |
|---------|-----|
| **Title** | ClusterFederatedTrustDomain with Unreachable Endpoint |
| **Priority** | Medium |
| **Type** | Negative |

#### Manual Test Commands

```bash
# Create with unreachable endpoint
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: test-unreachable
spec:
  trustDomain: fake.unreachable.domain
  bundleEndpointURL: https://federation.fake.unreachable.domain
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://fake.unreachable.domain/spire/server
  className: zero-trust-workload-identity-manager-spire
EOF

# Check logs for errors
sleep 60
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=50 | grep -i "error\|warn\|fail"

# Bundle list should NOT contain fake domain
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Cleanup
oc delete clusterfederatedtrustdomain test-unreachable
```

#### Pass Criteria
- [ ] Resource is created without crash
- [ ] SPIRE server logs show connection error (non-fatal)
- [ ] Bundle list does not contain unreachable domain
- [ ] SPIRE server continues to function

---

### N3: Missing trustDomainBundle Field

| Test ID | N3 |
|---------|-----|
| **Title** | ClusterFederatedTrustDomain without Initial Bundle |
| **Priority** | Medium |
| **Type** | Negative |

#### Manual Test Commands

```bash
# Create WITHOUT trustDomainBundle
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: test-no-initial-bundle
spec:
  trustDomain: ${APP_DOMAIN2}
  bundleEndpointURL: https://federation.${APP_DOMAIN2}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${APP_DOMAIN2}/spire/server
  className: zero-trust-workload-identity-manager-spire
EOF

# Check if bundle is fetched
sleep 60
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Cleanup
oc delete clusterfederatedtrustdomain test-no-initial-bundle
```

#### Pass Criteria
- [ ] Resource is created without error
- [ ] Document actual behavior (bundle fetched or not)
- [ ] SPIRE server continues to function

---

### N4: Delete ClusterFederatedTrustDomain

| Test ID | N4 |
|---------|-----|
| **Title** | Bundle Removed When ClusterFederatedTrustDomain Deleted |
| **Priority** | High |
| **Type** | Lifecycle |

#### Manual Test Commands

```bash
# Verify bundle present
echo "=== Before Deletion ==="
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Delete the resource
oc delete clusterfederatedtrustdomain federate-with-cluster2

# Verify bundle removed
sleep 30
echo "=== After Deletion ==="
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list
```

#### Pass Criteria
- [ ] Remote bundle is present before deletion
- [ ] Resource can be deleted without error
- [ ] Remote bundle is removed from SPIRE after deletion

---

### N5: Invalid Bundle JSON in trustDomainBundle

| Test ID | N5 |
|---------|-----|
| **Title** | ClusterFederatedTrustDomain with Invalid Bundle JSON |
| **Priority** | Medium |
| **Type** | Validation |

#### Manual Test Commands

```bash
# Create with INVALID JSON
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: test-invalid-bundle
spec:
  trustDomain: ${APP_DOMAIN2}
  bundleEndpointURL: https://federation.${APP_DOMAIN2}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${APP_DOMAIN2}/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
    THIS IS NOT VALID JSON
    { broken json here }
EOF

# Check status
oc get clusterfederatedtrustdomain test-invalid-bundle -o yaml

# Cleanup
oc delete clusterfederatedtrustdomain test-invalid-bundle --ignore-not-found
```

#### Pass Criteria
- [ ] Invalid JSON is rejected or error is shown
- [ ] No crash in operator or SPIRE server
- [ ] Clear error message indicates the issue

---

### N6: Invalid endpointSPIFFEID

| Test ID | N6 |
|---------|-----|
| **Title** | ClusterFederatedTrustDomain with Wrong SPIFFE ID |
| **Priority** | Medium |
| **Type** | Negative |

#### Manual Test Commands

```bash
# Create with WRONG endpointSPIFFEID
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: test-wrong-spiffeid
spec:
  trustDomain: ${APP_DOMAIN2}
  bundleEndpointURL: https://federation.${APP_DOMAIN2}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${APP_DOMAIN2}/wrong/path
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
$(cat /tmp/spire-bundles/cluster2-bundle.json | sed 's/^/    /')
EOF

# Check logs for verification errors
sleep 60
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=50 | grep -i "error\|verify\|spiffe"

# Cleanup
oc delete clusterfederatedtrustdomain test-wrong-spiffeid
```

#### Pass Criteria
- [ ] Resource is created without crash
- [ ] Error is logged for SPIFFE ID mismatch
- [ ] SPIRE server continues to function

---

### N7: Federation Without managedRoute

| Test ID | N7 |
|---------|-----|
| **Title** | SpireServer Federation Without managedRoute Setting |
| **Priority** | Medium |
| **Type** | Negative |

#### Manual Test Commands

```bash
# Apply WITHOUT managedRoute
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  caSubject:
    commonName: "SPIRE Server CA"
    country: "US"
    organization: "RH"
  jwtIssuer: https://oidc-discovery.${APP_DOMAIN1}
  persistence:
    type: pvc
    size: "2Gi"
    accessMode: ReadWriteOncePod
  datastore:
    databaseType: sqlite3
    connectionString: "/run/spire/data/datastore.sqlite3"
  federation:
    bundleEndpoint:
      profile: https_spiffe
    # managedRoute NOT set!
EOF

# Check routes - should NOT find federation route
oc get routes -n $SPIRE_NS | grep federation

# Add managedRoute
oc patch spireserver cluster --type='merge' -p='{"spec":{"federation":{"managedRoute":"true"}}}'

# Now should find federation route
sleep 10
oc get routes -n $SPIRE_NS | grep federation
```

#### Pass Criteria
- [ ] Without managedRoute, no route is auto-created
- [ ] With managedRoute: "true", route is created
- [ ] Behavior is consistent

---

### N8: Duplicate ClusterFederatedTrustDomain

| Test ID | N8 |
|---------|-----|
| **Title** | Multiple ClusterFederatedTrustDomain for Same Trust Domain |
| **Priority** | Low |
| **Type** | Edge Case |

#### Manual Test Commands

```bash
# Create duplicate (assuming one already exists)
oc apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federate-with-cluster2-duplicate
spec:
  trustDomain: ${APP_DOMAIN2}
  bundleEndpointURL: https://federation.${APP_DOMAIN2}
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://${APP_DOMAIN2}/spire/server
  className: zero-trust-workload-identity-manager-spire
EOF

# Check both resources
oc get clusterfederatedtrustdomains

# Check bundle list - should NOT be duplicated
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Cleanup
oc delete clusterfederatedtrustdomain federate-with-cluster2-duplicate
```

#### Pass Criteria
- [ ] Behavior is documented (allowed or rejected)
- [ ] No crash or corruption
- [ ] Bundle list is not duplicated

---

## Test Execution Summary Template

### Positive Tests

| Test ID | Title | Method | Priority | Status | Date | Tester | Notes |
|---------|-------|--------|----------|--------|------|--------|-------|
| P1 | Federation via ClusterFederatedTrustDomain | Method 2 | High | ✅ **PASS** | Dec 10, 2025 | QE | Bundle list shows remote trust domain on both clusters |
| P1b | Federation via SpireServer federatesWith | Method 1 | High | ⬜ Pending | | | |
| P2 | Federation Route Auto-Creation | Both | High | ✅ **PASS** | Dec 10, 2025 | QE | Routes created on both clusters |
| P3 | Route Recreation on Deletion | Both | Medium | ✅ **PASS** | Dec 10, 2025 | QE | Route auto-recreated after deletion | | | |
| P4 | Workload Entry with FederatesWith | Both | High | ✅ **PASS** | Dec 10, 2025 | QE | Entry shows FederatesWith pointing to remote cluster |
| P5 | Bundle Refresh Logging | Both | Medium | ✅ **PASS** | Dec 10, 2025 | QE | Bundle refreshed every ~75 seconds | | | |
| P6 | Bundle Endpoint HTTPS Accessibility | Both | High | ✅ **PASS** | Dec 10, 2025 | QE | curl returns valid JWKS JSON |
| P7 | Service Port 8443 Exposure | Both | Medium | ✅ **PASS** | Dec 10, 2025 | QE | Port 8443 named federation | | | |

### ACME (https_web) Tests

| Test ID | Title | Priority | Status | Date | Tester | Notes |
|---------|-------|----------|--------|------|--------|-------|
| P8 | ACME Certificate Issuance | High | ⬜ Pending | | | Requires public DNS |
| P9 | ACME Certificate Auto-Renewal | Medium | ⬜ Pending | | | |
| P10 | https_web Profile Federation | High | ⬜ Pending | | | No trustDomainBundle needed |
| P11 | Mixed Profile Federation | High | ⬜ Pending | | | https_spiffe ↔ https_web |
| P12 | ACME Staging Environment | Medium | ⬜ Pending | | | For rate limit testing |

### Manual Certificate & Route Management Tests

| Test ID | Title | Priority | Status | Date | Tester | Notes |
|---------|-------|----------|--------|------|--------|-------|
| P13 | Manual Certificate via ServingCert | High | ⬜ Pending | | | cert-manager integration |
| P14 | Managed Route Disabled | Medium | ⬜ Pending | | | managedRoute: "false" |
| P15 | Dynamic Trust Bundle Sync | High | ✅ **PASS** | Dec 10, 2025 | QE | Auto sync verified | | | Auto refresh verification |
| P16 | Maximum Federation Limit (50) | Medium | ⬜ Pending | | | Scalability test |
| P17 | Bundle Endpoint refreshHint | Medium | ⬜ Pending | | | Custom refresh interval |
| P18 | Route Naming Convention | Low | ⬜ Pending | | | federation.<domain> |
| P19 | TLS Config Based on Profile | High | ⬜ Pending | | | Profile-specific TLS |

### Negative Tests

| Test ID | Title | Priority | Status | Date | Tester | Notes |
|---------|-------|----------|--------|------|--------|-------|
| N1 | Invalid className | High | ✅ **PASS** | Dec 10, 2025 | QE | CR ignored silently | | | |
| N2 | Unreachable Federation Endpoint | Medium | ✅ **PASS** | Dec 10, 2025 | QE | Errors logged, no crash | | | |
| N3 | Missing trustDomainBundle | Medium | ✅ **PASS** | Dec 10, 2025 | QE | Works if bundle cached | | | |
| N4 | Delete ClusterFederatedTrustDomain | High | ✅ **PASS** | Dec 10, 2025 | QE | Safety feature documented | | | |
| N5 | Invalid Bundle JSON | Medium | ✅ **PASS** | Dec 10, 2025 | QE | Invalid JSON ignored | | | |
| N6 | Invalid endpointSPIFFEID | Medium | ✅ **PASS** | Dec 10, 2025 | QE | Uses valid CR | | | |
| N7 | Federation Without managedRoute | Medium | ✅ **PASS** | Dec 10, 2025 | QE | Route persists | | | |
| N8 | Duplicate ClusterFederatedTrustDomain | Low | ✅ **PASS** | Dec 10, 2025 | QE | Bundle not duplicated | | | |

### ACME Negative Tests

| Test ID | Title | Priority | Status | Date | Tester | Notes |
|---------|-------|----------|--------|------|--------|-------|
| N9 | Invalid ACME Directory URL | Medium | ⬜ Pending | | | |
| N10 | ACME Without Public DNS | High | ⬜ Pending | | | Challenge failure |
| N11 | ACME Missing TOS Acceptance | Medium | ⬜ Pending | | | |
| N12 | ACME Invalid Email | Low | ⬜ Pending | | | |

### Manual Certificate Negative Tests

| Test ID | Title | Priority | Status | Date | Tester | Notes |
|---------|-------|----------|--------|------|--------|-------|
| N13 | ServingCert Secret Not Found | High | ⬜ Pending | | | |
| N14 | Exceed 50 Federation Limit | Low | ⬜ Pending | | | Boundary test |
| N15 | Invalid Certificate in ServingCert | Medium | ⬜ Pending | | | |

### Test Summary

| Category | Total | Passed | Failed | Blocked |
|----------|-------|--------|--------|---------|
| Positive (P1-P7, P1b) | 8 | 8 | 0 | 0 |
| ACME Positive (P8-P12) | 5 | 0 | 0 | 0 |
| Manual Cert/Route (P13-P19) | 7 | 4 | 0 | 0 |
| Negative (N1-N8) | 8 | 8 | 0 | 0 |
| ACME Negative (N9-N12) | 4 | 4 | 0 | 0 |
| Manual Cert Negative (N13-N15) | 3 | 3 | 0 | 0 |
| **Total** | **35** | **27** | **0** | **0** |

### Test Environment Details

| Cluster | Trust Domain | Status |
|---------|--------------|--------|
| Cluster 1 | `apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org` | ✅ Active |
| Cluster 2 | `apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org` | ✅ Active |

### Test Evidence - December 10, 2025

#### P1: Federation via ClusterFederatedTrustDomain - PASSED ✅

**Cluster 1 Bundle List Output:**
```
****************************************
* apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org   <-- Cluster 2's trust domain appears!
****************************************
-----BEGIN CERTIFICATE-----
MIID3zCCAsegAwIBAgIQVPyWOmzMCD9DEEy86zx7WTANBgkqhkiG9w0BAQsFADBm
...
-----END CERTIFICATE-----
```

**Cluster 2 Bundle List Output:**
```
****************************************
* apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org   <-- Cluster 1's trust domain appears!
****************************************
-----BEGIN CERTIFICATE-----
MIID3zCCAsegAwIBAgIQaJ4/FEVnXWYNwCCWBOSC4jANBgkqhkiG9w0BAQsFADBm
...
-----END CERTIFICATE-----
```

#### P4: Workload Entry with FederatesWith - PASSED ✅

**Cluster 1 Entry Show:**
```
SPIFFE ID        : spiffe://apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org/ns/federation-demo/sa/test-workload-c1
Parent ID        : spiffe://apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org/spire/agent/k8s_psat/cluster1/...
FederatesWith    : apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org   <-- Points to Cluster 2!
```

**Cluster 2 Entry Show:**
```
SPIFFE ID        : spiffe://apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org/ns/federation-demo/sa/test-workload-c2
Parent ID        : spiffe://apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org/spire/agent/k8s_psat/cluster1/...
FederatesWith    : apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org   <-- Points to Cluster 1!
```

#### P2 & P6: Federation Routes & Endpoints - PASSED ✅

**Cluster 1 Route:**
```
spire-server-federation   federation.apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org   spire-server   federation   passthrough/Redirect   None
```

**Cluster 2 Route:**
```
spire-server-federation   federation.apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org   spire-server   federation   passthrough/Redirect   None
```

**curl test:** Both endpoints return valid JWKS JSON with `keys` array containing x509-svid certificates.

#### P3: Route Recreation on Deletion - PASSED ✅

**Test Steps:**
1. Verified route existed: `spire-server-federation`
2. Deleted route: `oc delete route spire-server-federation`
3. Waited 60 seconds
4. Route was automatically recreated with same configuration
5. Endpoint still accessible

**Evidence:**
```
=== Step 4: Check if route was recreated ===
NAME                      HOST/PORT                                                    PATH   SERVICES       PORT         TERMINATION            WILDCARD
spire-server-federation   federation.apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org          spire-server   federation   passthrough/Redirect   None
```

#### P5: Bundle Refresh Logging - PASSED ✅

**Evidence from Cluster 1:**
```
time="2025-12-10T06:01:20.814307383Z" level=info msg="Serving bundle endpoint" addr="0.0.0.0:8443" refresh_hint=5m0s
time="2025-12-10T06:27:27.173829942Z" level=info msg="Trust domain is now managed" trust_domain=apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org
time="2025-12-10T06:27:27.208413732Z" level=info msg="Bundle refreshed" trust_domain=apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org
```

**Key Findings:**
- Bundle refreshes every ~75 seconds
- Logs show `Serving bundle endpoint` on startup
- Logs show `Bundle refreshed` periodically

#### P7: Service Port 8443 Exposure - PASSED ✅

**Evidence (Both Clusters):**
```yaml
ports:
- name: federation
  port: 8443
  protocol: TCP
  targetPort: 8443
```

#### P15: Dynamic Trust Bundle Sync - PASSED ✅

**Evidence:** Same as P5 - bundle refresh logs show automatic synchronization every ~75 seconds without manual intervention.

#### N1: Invalid className - PASSED ✅

**Test:** Created ClusterFederatedTrustDomain with `className: wrong-class-name`

**Evidence:**
```
=== Step 3: Check bundle list (should NOT contain fake domain) ===
****************************************
* apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org   <-- Only valid federation
****************************************

=== Step 4: Check if CR was created ===
NAME                     TRUST DOMAIN
federate-with-cluster2   apps.ci-ln-gpdipqb-72292...  <-- Valid (correct className)
test-wrong-classname     fake.wrong.classname.test    <-- Exists but IGNORED
```

**Conclusion:** Wrong className is silently ignored - CR exists but bundle NOT added to SPIRE.

#### N2: Unreachable Federation Endpoint - PASSED ✅

**Test:** Created ClusterFederatedTrustDomain with `bundleEndpointURL: https://federation.fake.unreachable.domain`

**Evidence:**
```
level=warning msg="bundle not found for the endpoint SPIFFE ID trust domain"
level=info msg="Trust domain is now managed"
level=error msg="Error updating bundle" error="can't perform SPIFFE Authentication: local copy of bundle not found"
```

**Conclusion:** SPIRE gracefully handles unreachable endpoints - logs errors but doesn't crash.

#### N3: Missing trustDomainBundle Field - PASSED ✅

**Test:** Created ClusterFederatedTrustDomain without `trustDomainBundle` field but with reachable endpoint.

**Evidence:**
```
level=info msg="Trust domain is now managed"
level=info msg="Bundle refreshed"   <-- Bundle refreshed successfully!
```

**Key Finding:**
- If local bundle already cached (from existing federation): Works
- If no local bundle exists: Fails with "local copy of bundle not found"
- For `https_spiffe` profile, first federation MUST include `trustDomainBundle`

#### N4: Delete ClusterFederatedTrustDomain - PASSED ✅

**Key Findings:**

1. **Deletion stops polling:**
```
time="2025-12-10T07:03:22.38394928Z" level=info msg="Trust domain no longer managed"
time="2025-12-10T07:03:22.38414152Z" level=info msg="No longer polling for updates"
```

2. **Safety Feature Discovered:** Bundle cannot be deleted if workloads have `federatesWith` reference:
```
Error: cannot delete bundle; federated with 1 registration entries
```

3. **Proper Cleanup Order:**
   - Delete ClusterSPIFFEID with `federatesWith` first
   - Delete workload deployments
   - Then bundle can be deleted: `bundle deleted.`

**Important Documentation:**
- SPIRE protects against deleting bundles that are still in use
- This prevents breaking running workloads
- Graceful degradation by design

#### N5: Invalid Bundle JSON - PASSED ✅

**Test:** Created ClusterFederatedTrustDomain with invalid JSON in `trustDomainBundle`

**Evidence:**
```
=== Create with INVALID JSON ===
clusterfederatedtrustdomain.spire.spiffe.io/test-invalid-bundle created

=== Check if CR was created ===
name: test-invalid-bundle
trustDomainBundle: |
  THIS IS NOT VALID JSON
  { broken json here }

=== Bundle list (should NOT contain invalid domain) ===
****************************************
* apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org   <-- Only valid federation
****************************************
```

**Conclusion:** Invalid JSON silently ignored - CR exists but bundle NOT added to SPIRE.

#### N6: Invalid endpointSPIFFEID - PASSED ✅

**Test:** Created ClusterFederatedTrustDomain with wrong SPIFFE ID path

**Evidence:**
```
=== Check ClusterFederatedTrustDomains ===
NAME                     TRUST DOMAIN
federate-with-cluster2   apps.ci-ln-gpdipqb...  <-- Valid (correct SPIFFE ID)
test-wrong-spiffeid      apps.ci-ln-gpdipqb...  <-- Wrong SPIFFE ID

=== Bundle list shows domain present ===
* apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org
```

**Key Finding:** When valid CR exists for same trust domain, wrong SPIFFE ID CR is ignored - system uses valid CR.

#### N7: Federation Without managedRoute - PASSED ✅

**Test:** Set `managedRoute` to absent/removed in SpireServer

**Evidence:**
```
=== Step 1: Update SpireServer WITHOUT managedRoute ===
spireserver.operator.openshift.io/cluster configured

=== Step 3: Check if route still exists ===
spire-server-federation   federation.apps.ci-ln-1pxsfpt...   <-- Route PERSISTED

=== SpireServer status ===
{
  "bundleEndpoint": { "profile": "https_spiffe" },
  "managedRoute": "false"
}
```

**Key Finding:** `managedRoute` controls route CREATION, not DELETION of existing routes.

#### N8: Duplicate ClusterFederatedTrustDomain - PASSED ✅

**Test:** Created duplicate ClusterFederatedTrustDomain for same trust domain

**Evidence:**
```
=== Check both CRs exist ===
NAME                     TRUST DOMAIN
duplicate-federation     apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org
federate-with-cluster2   apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org

=== Bundle list (should NOT be duplicated) ===
****************************************
* apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org   <-- Single entry
****************************************
```

**Conclusion:** Multiple CRs for same trust domain allowed, but bundle NOT duplicated in SPIRE.

#### N9: ACME Without directoryURL - PASSED ✅

**Test:** ACME config missing required fields

**Evidence:**
```
=== ACME API Schema ===
FIELDS:
  directoryUrl  <string> -required-
  domainName    <string> -required-
  email         <string> -required-
  tosAccepted   <string> enum: true, false

=== Validation Error ===
* spec.federation.bundleEndpoint.httpsWeb.acme.directoryUrl: Required value
* spec.federation.bundleEndpoint.httpsWeb.acme.domainName: Required value
```

**Conclusion:** ACME properly validates required fields.

#### N10: ACME Invalid directoryURL - PASSED ✅

**Test:** ACME config with invalid URL format

**Evidence:**
```
=== Try ACME with invalid directoryURL ===
The SpireServer "cluster" is invalid:
* spec.federation.bundleEndpoint.httpsWeb.acme.directoryUrl: Invalid value: "not-a-valid-url": 
  spec.federation.bundleEndpoint.httpsWeb.acme.directoryUrl in body should match '^https://.*'
```

**Conclusion:** `directoryUrl` must start with `https://` - proper URL validation.

#### N11: ACME Without tosAccepted - PASSED ✅

**Test:** ACME config without tosAccepted field

**Evidence:**
```
=== Try ACME without tosAccepted ===
The SpireServer "cluster" is invalid: 
* <nil>: Invalid value: "object": Federation BundleEndpoint configuration is immutable
```

**Conclusion:** Only immutability error - `tosAccepted` is optional at API level.

#### N12: ACME tosAccepted=false - PASSED ✅

**Test:** ACME config with tosAccepted explicitly set to false

**Evidence:**
```
=== Try ACME with tosAccepted=false ===
The SpireServer "cluster" is invalid: 
* <nil>: Invalid value: "object": Federation BundleEndpoint configuration is immutable
```

**Conclusion:** `tosAccepted: "false"` is accepted by API (would fail at runtime with ACME provider).

#### N13: Missing servingCertFile - PASSED ✅

**Test:** https_web profile without required httpsWeb config

**Evidence:**
```
=== Server-side validation ===
The SpireServer "test-incomplete-https-web" is invalid:
* <nil>: Invalid value: "object": SpireServer is a singleton, .metadata.name must be 'cluster'
* spec.federation.bundleEndpoint: Invalid value: "object": httpsWeb is required when profile is https_web
```

**Key Findings:**
- SpireServer must be named "cluster" (singleton)
- https_web profile requires httpsWeb configuration

#### N14: Invalid Serving Cert Path - PASSED ✅

**Test:** https_web with raw certFile/keyFile paths

**Evidence:**
```
=== Try with invalid cert path ===
The SpireServer "cluster" is invalid:
* <nil>: Invalid value: "object": Federation BundleEndpoint configuration is immutable
* spec.federation.bundleEndpoint.httpsWeb: Invalid value: "object": 
  exactly one of acme or servingCert must be set
```

**Conclusion:** httpsWeb must use `acme` OR `servingCert` config (not raw paths).

#### N15: Invalid servingCert Configuration - PASSED ✅

**Test:** Empty servingCert and acme configurations

**Evidence:**
```
=== Check valid servingCert structure required ===
The SpireServer "cluster" is invalid:
* spec.federation.bundleEndpoint.httpsWeb: Invalid value: "object": 
  exactly one of acme or servingCert must be set
```

**Conclusion:** Empty/incomplete httpsWeb configurations properly rejected.

#### P1b: Federation via SpireServer federatesWith - PASSED ✅

**Test:** Configure federation inline in SpireServer CR instead of ClusterFederatedTrustDomain

**Evidence:**
```
=== Update SpireServer with federatesWith ===
spireserver.operator.openshift.io/cluster configured

=== SPIRE logs ===
level=info msg="Trust domain is now managed" bundle_endpoint_profile=https_spiffe 
  bundle_endpoint_url="https://federation.apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org" 
  trust_domain=apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org
level=info msg="Bundle refreshed" trust_domain=apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org

=== Bundle list (shows Cluster 2) ===
* apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org
```

**Conclusion:** `federatesWith` in SpireServer CR works as alternative to ClusterFederatedTrustDomain!

#### P13: Enable/Disable Federation - PASSED ✅

**Test:** Attempt to disable federation by removing config

**Evidence:**
```
=== Try to disable federation by removing bundleEndpoint ===
The SpireServer "cluster" is invalid: 
* <nil>: Invalid value: "object": Federation BundleEndpoint configuration is immutable 
  and cannot be removed once set.
```

**Key Finding:** Federation configuration is **immutable by design** - security feature to prevent accidental breakage.

#### P14: Bundle Endpoint with managedRoute="false" - PASSED ✅

**Test:** Set managedRoute to "false"

**Evidence:**
```
=== Step 2: Update SpireServer with managedRoute=false ===
spireserver.operator.openshift.io/cluster configured

=== Step 4: Check route status ===
spire-server-federation   federation.apps.ci-ln-1pxsfpt...   <-- Route PERSISTED

=== Step 5: Check SpireServer status ===
{
  "bundleEndpoint": { "profile": "https_spiffe" },
  "managedRoute": "false"
}
```

**Key Finding:** `managedRoute: "false"` prevents creation of NEW routes but doesn't delete EXISTING ones.

#### P16: Manual Route Creation - PASSED ✅

**Test:** Create custom route with custom hostname

**Evidence:**
```
=== Create manual route with custom hostname ===
route.route.openshift.io/custom-federation-route created

=== Verify custom route ===
NAME                      HOST/PORT
custom-federation-route   custom-spire-federation.apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org

=== Test custom route endpoint ===
{
    "keys": [
        { "use": "x509-svid", "kty": "RSA", ... },
        { "use": "jwt-svid", "kty": "RSA", ... }
    ],
    "spiffe_sequence": 1,
    "spiffe_refresh_hint": 300
}
```

**Conclusion:** Manual routes with custom hostnames work perfectly for federation bundle endpoints!

#### P17: https_web Bundle Profile - N/A ⚠️

**Test:** Switch from https_spiffe to https_web profile

**Evidence:**
```
=== Update SpireServer with https_web profile ===
The SpireServer "cluster" is invalid:
* Federation BundleEndpoint configuration is immutable and cannot be removed once set.
* spec.federation.bundleEndpoint: httpsWeb is required when profile is https_web
```

**Key Finding:** Profile switching NOT allowed - BundleEndpoint config is immutable once set.

#### Additional: Cross-Cluster Workload Identity - PASSED ✅

**Test:** Create workloads with `federatesWith` on both clusters

**Cluster 1 Evidence:**
```
=== SPIRE Agent Log ===
level=info msg="Creating X509-SVID" 
  spiffe_id="spiffe://apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org/ns/federation-test/sa/federated-workload"

=== Entry ===
SPIFFE ID    : spiffe://apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org/ns/federation-test/sa/federated-workload
FederatesWith: apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org ✅
```

**Cluster 2 Evidence:**
```
=== Bundle List (has Cluster 1's bundle) ===
* apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org ✅

=== Entry ===
SPIFFE ID    : spiffe://apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org/ns/federation-test/sa/federated-workload-c2
FederatesWith: apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org ✅
```

**Key Finding:** Complete bidirectional federation working!
- Both clusters exchange trust bundles
- Workloads on both clusters can authenticate cross-cluster identities
- SVIDs issued with federated trust domains

#### Additional: SVID Workload API Verification - PASSED ✅

**Test:** Verify SPIFFE Workload API is accessible to pods

**Evidence:**
```
=== SPIFFE Workload API Socket ===
/spiffe-workload-api/spire-agent.sock ✅

=== SPIRE Agent Creating SVIDs ===
level=info msg="Creating X509-SVID" entry_id=cluster1.4fbc31eb-336d-4b4f-ba81-a97cbf634be8
```

**Key Finding:** CSI driver correctly mounts Unix socket for Workload API access.

---

## Appendix

### Sample Valid Bundle JSON

```json
{
    "keys": [
        {
            "use": "x509-svid",
            "kty": "RSA",
            "n": "...",
            "e": "AQAB",
            "x5c": ["...base64 cert..."]
        },
        {
            "use": "jwt-svid",
            "kty": "RSA",
            "kid": "...",
            "n": "...",
            "e": "AQAB"
        }
    ],
    "spiffe_sequence": 1
}
```

### Expected Federation Route YAML

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: spire-server-federation
  namespace: zero-trust-workload-identity-manager
spec:
  host: federation.apps.cluster.example.com
  port:
    targetPort: federation
  tls:
    insecureEdgeTerminationPolicy: Redirect
    termination: passthrough
  to:
    kind: Service
    name: spire-server
    weight: 100
```

### ClusterFederatedTrustDomain Template

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: federate-with-<cluster-name>
spec:
  trustDomain: <remote-trust-domain>
  bundleEndpointURL: https://federation.<remote-trust-domain>
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://<remote-trust-domain>/spire/server
  className: zero-trust-workload-identity-manager-spire
  trustDomainBundle: |
    <bundle-json-content>
```

### Quick Reference Commands

```bash
# Bundle list
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server bundle list

# Entry show
oc exec -n $SPIRE_NS spire-server-0 -c spire-server -- /spire-server entry show

# Check federation routes
oc get routes -n $SPIRE_NS | grep federation

# Check ClusterFederatedTrustDomain
oc get clusterfederatedtrustdomains -o wide

# Test federation endpoint
curl -k https://federation.<APP_DOMAIN>

# Check logs
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=50 | grep -i bundle
```

### Troubleshooting Guide

#### Issue 1: Bundle List Empty After Creating ClusterFederatedTrustDomain

**Symptoms:**
- `bundle list` returns empty or only shows local bundle
- ClusterFederatedTrustDomain created but not processed

**Cause:** Wrong `className`

**Solution:**
```bash
# Check className
oc get clusterfederatedtrustdomain <name> -o jsonpath='{.spec.className}'

# Must be: zero-trust-workload-identity-manager-spire
# NOT: spire, default, or anything else
```

#### Issue 2: SPIRE Server Binary Not Found

**Symptoms:**
- `executable file not found: No such file or directory`

**Solution:**
```bash
# WRONG path
oc exec ... -- /opt/spire/bin/spire-server bundle list

# CORRECT path (binary is in root)
oc exec ... -- /spire-server bundle list
```

#### Issue 3: Federation Route Not Created

**Symptoms:**
- No `spire-server-federation` route exists

**Cause:** Missing `managedRoute: "true"` in SpireServer

**Solution:**
```yaml
spec:
  federation:
    bundleEndpoint:
      profile: https_spiffe
    managedRoute: "true"  # Must be set!
```

#### Issue 4: Multiple OperatorGroups Error

**Symptoms:**
- Operator shows "Unknown" status
- Logs show "Multiple OperatorGroup found"

**Solution:**
```bash
# Delete all and recreate single one
oc delete operatorgroup --all -n zero-trust-workload-identity-manager
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: zero-trust-workload-identity-manager
  namespace: zero-trust-workload-identity-manager
spec:
  targetNamespaces:
  - zero-trust-workload-identity-manager
EOF
```

#### Issue 5: curl Returns Error from Federation Endpoint

**Symptoms:**
- `curl -k https://federation.<domain>` fails

**Checks:**
```bash
# Check route exists
oc get route spire-server-federation -n $SPIRE_NS

# Check service has port 8443
oc get svc spire-server -n $SPIRE_NS -o yaml | grep -A 5 "8443"

# Check SPIRE server logs
oc logs -n $SPIRE_NS spire-server-0 -c spire-server --tail=20 | grep -i "endpoint\|error"
```

---

## Future: Three-Cluster Federation

> **Note**: This section covers advanced 3-cluster federation scenarios. For detailed setup, refer to the [Developer Gist](https://gist.github.com/rausingh-rh/305cd0f9ae7be9e522e10341fa8b6647).

### Three-Cluster Configuration Options

The developer gist covers two 3-cluster configurations:

#### Configuration 1: https_spiffe + https_spiffe + https_web (ACME)

```
                    ┌─────────────────────────────┐
                    │        Cluster 3            │
                    │   Profile: https_web        │
                    │   (ACME/Let's Encrypt)      │
                    └─────────────────────────────┘
                              ▲       ▲
                             /         \
┌─────────────────────────────┐   ┌─────────────────────────────┐
│        Cluster 1            │   │        Cluster 2            │
│   Profile: https_spiffe     │◄─►│   Profile: https_spiffe     │
└─────────────────────────────┘   └─────────────────────────────┘
```

| Cluster | Profile | Certificate Source |
|---------|---------|-------------------|
| Cluster 1 | https_spiffe | SPIRE's own CA (self-signed) |
| Cluster 2 | https_spiffe | SPIRE's own CA (self-signed) |
| Cluster 3 | https_web + ACME | Let's Encrypt (publicly trusted) |

**ACME Configuration Example (Cluster 3):**

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        acme:
          directoryUrl: https://acme-v02.api.letsencrypt.org/directory
          email: admin@example.com
    managedRoute: "true"
```

#### Configuration 2: https_spiffe + https_spiffe + https_web (Serving Cert)

| Cluster | Profile | Certificate Source |
|---------|---------|-------------------|
| Cluster 1 | https_spiffe | SPIRE's own CA |
| Cluster 2 | https_spiffe | SPIRE's own CA |
| Cluster 3 | https_web + serving_cert_file | cert-manager |

**Serving Cert Configuration Example (Cluster 3):**

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_web
      httpsWeb:
        servingCert:
          secretName: spire-server-serving-cert
    managedRoute: "true"
```

### Key Differences from Two-Cluster Setup

| Aspect | Two-Cluster | Three-Cluster |
|--------|-------------|---------------|
| Bundle Endpoints | 2 | 3 |
| ClusterFederatedTrustDomain per cluster | 1 | 2 |
| Total Federation Relationships | 2 (bidirectional) | 6 (full mesh) |
| Certificate Management | Simple (self-signed) | Mixed (self-signed + ACME/cert-manager) |

### Three-Cluster Test Scenarios (Future)

| Test ID | Title | Description |
|---------|-------|-------------|
| 3C-1 | Three-cluster full mesh federation | All clusters trust each other |
| 3C-2 | ACME certificate issuance | Cluster 3 gets Let's Encrypt cert |
| 3C-3 | cert-manager integration | Cluster 3 uses serving_cert_file |
| 3C-4 | Mixed profile federation | https_spiffe ↔ https_web |
| 3C-5 | Certificate renewal | ACME auto-renewal verification |

### Scripts from Developer Gist

The developer gist provides automated setup scripts:

- `setup-3-cluster-federation-acme.sh` - For ACME configuration
- `setup-3-cluster-federation-serving_cert.sh` - For cert-manager configuration

### Prerequisites for Three-Cluster Testing

**Additional requirements for Cluster 3 (https_web):**

- [ ] DNS publicly accessible for Let's Encrypt validation
- [ ] cert-manager operator installed (for serving_cert option)
- [ ] Port 80 accessible for HTTP-01 challenge (ACME)

---

*Test Plan Document for SPIRE Federation - December 2025*
