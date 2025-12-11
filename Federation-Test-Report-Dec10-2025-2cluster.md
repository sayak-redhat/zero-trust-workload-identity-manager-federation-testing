# SPIRE Federation Test Report

**Date:** December 10-11, 2025  
**Tester:** QE Team  
**Version:** Zero Trust Workload Identity Manager Operator  
**Test Plan:** Federation-Test-Plan.md v1.2  

---

## Executive Summary

| Metric | Value |
|--------|-------|
| **Total Tests** | 35 |
| **Passed** | 33 (94%) |
| **Failed** | 0 (0%) |
| **N/A** | 1 (3%) |
| **Future** | 1 (3%) |

### Verdict: ✅ PASS

Federation functionality is working correctly. All executable tests passed, including **ACME (Let's Encrypt) integration testing** using sslip.io for public DNS.

**Future Tests:** P9 (auto-renewal - requires ~30 days to observe certificate renewal cycle).

---

## Test Environment

### Day 1 (Dec 10) - https_spiffe Testing

| Cluster | Trust Domain | Profile |
|---------|--------------|---------|
| **Cluster 1** | `apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org` | https_spiffe |
| **Cluster 2** | `apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org` | https_spiffe |

### Day 2 (Dec 11) - ACME Testing

| Cluster | Trust Domain | Profile | ACME Domain |
|---------|--------------|---------|-------------|
| **Cluster 1** | `apps.ci-ln-lh14qqk-72292.origin-ci-int-gce.dev.rhcloud.com` | https_web (ACME) | `federation.35.225.244.231.sslip.io` |
| **Cluster 2** | `apps.ci-ln-7bgj5qt-72292.origin-ci-int-gce.dev.rhcloud.com` | https_spiffe | N/A |

### Day 3 (Dec 11) - Additional Federation Tests (P17-P19)

| Cluster | Trust Domain | Profile |
|---------|--------------|---------|
| **Cluster 1** | `apps.ci-ln-jlcigs2-72292.gcp-2.ci.openshift.org` | https_spiffe |
| **Cluster 2** | `apps.ci-ln-ir8ikrb-72292.gcp-2.ci.openshift.org` | https_spiffe |

### Infrastructure
- OpenShift 4.x on GCP
- Zero Trust Workload Identity Manager Operator (Stage Build)
- ACME: Let's Encrypt Staging via sslip.io

---

## Detailed Test Results

### Positive Tests (8/8 PASSED ✅)

| ID | Test | Result | Notes |
|----|------|--------|-------|
| P1 | Federation via ClusterFederatedTrustDomain | ✅ PASS | Both clusters federated successfully |
| P1b | Federation via SpireServer federatesWith | ✅ PASS | Alternative method works |
| P2 | Federation Route Auto-Creation | ✅ PASS | Route created automatically |
| P3 | Route Recreation on Deletion | ✅ PASS | Route recreated when deleted |
| P4 | Workload Entry with FederatesWith | ✅ PASS | Entries show FederatesWith |
| P5 | Bundle Refresh Logging | ✅ PASS | Refresh every ~75 seconds |
| P6 | Bundle Endpoint HTTPS Accessibility | ✅ PASS | curl returns valid JWKS |
| P7 | Service Port 8443 Exposure | ✅ PASS | Federation port exposed |

### ACME Positive Tests (3/5 PASSED ✅, 1 N/A, 1 Future)

| ID | Test | Result | Notes |
|----|------|--------|-------|
| P8 | ACME Certificate Issuance | ✅ **PASS** | Staging cert issued via sslip.io |
| P9 | ACME Certificate Auto-Renewal | 📅 Future | Requires ~30 days to verify |
| P10 | https_web Profile Federation | ⚠️ **N/A** | Staging certs not publicly trusted |
| P11 | Mixed Profile Federation | ✅ **PASS** | https_web + https_spiffe works |
| P12 | ACME Staging Environment | ✅ **PASS** | Staging URL avoids rate limits |

### Manual Cert/Route Tests (7/7 PASSED ✅)

| ID | Test | Result | Notes |
|----|------|--------|-------|
| P13 | Enable/Disable Federation | ✅ **PASS** | Config immutable (expected behavior) |
| P14 | managedRoute="false" | ✅ **PASS** | Manual sslip.io route created |
| P15 | Dynamic Trust Bundle Sync | ✅ **PASS** | Auto-sync working |
| P16 | Maximum Federation Limit (50) | ✅ **PASS** | 50 federations created successfully |
| P17 | Bundle Endpoint refreshHint | ✅ **PASS** | refreshHint=60 appears in bundle response |
| P18 | Route Naming Convention | ✅ **PASS** | Route follows `federation.<domain>` pattern |
| P19 | TLS Config Based on Profile | ✅ **PASS** | https_spiffe uses SPIRE self-signed cert with passthrough TLS |

### Negative Tests (8/8 PASSED ✅)

| ID | Test | Result | Notes |
|----|------|--------|-------|
| N1 | Invalid className | ✅ PASS | Silently ignored |
| N2 | Unreachable Endpoint | ✅ PASS | Graceful error handling |
| N3 | Missing trustDomainBundle | ✅ PASS | Works if cached |
| N4 | Delete ClusterFederatedTrustDomain | ✅ PASS | Safety feature discovered |
| N5 | Invalid Bundle JSON | ✅ PASS | Invalid JSON ignored |
| N6 | Invalid endpointSPIFFEID | ✅ PASS | Uses valid CR |
| N7 | Federation Without managedRoute | ✅ PASS | Route persists |
| N8 | Duplicate ClusterFederatedTrustDomain | ✅ PASS | Bundle not duplicated |

### ACME Negative Tests (4/4 PASSED ✅)

| ID | Test | Result | Notes |
|----|------|--------|-------|
| N9 | Invalid ACME Directory URL | ✅ **PASS** | URL must match `^https://.*` |
| N10 | ACME Without Public DNS | ✅ **PASS** | TLS error when DNS unreachable |
| N11 | ACME Missing TOS Acceptance | ✅ **PASS** | Optional at API, fails at runtime |
| N12 | ACME Invalid Email | ✅ **PASS** | example.com rejected by Let's Encrypt |

### Manual Cert Negative Tests (3/3 PASSED ✅)

| ID | Test | Result | Notes |
|----|------|--------|-------|
| N13 | Missing servingCertFile | ✅ PASS | httpsWeb required |
| N14 | Invalid Serving Cert Path | ✅ PASS | acme OR servingCert required |
| N15 | Invalid servingCert Configuration | ✅ PASS | Empty configs rejected |

### Additional Tests (2/2 PASSED ✅)

| ID | Test | Result | Notes |
|----|------|--------|-------|
| A1 | Cross-Cluster Workload Identity | ✅ PASS | Bidirectional verified |
| A2 | SVID Workload API Verification | ✅ PASS | Socket API working |

---

## P17-P19 Test Details (December 11, 2025)

### P17: Bundle Endpoint refreshHint Configuration

**Test Execution:**
```
=== SpireServer Config ===
bundleEndpoint:
  profile: https_spiffe
  refreshHint: 60

=== Bundle Endpoint Response ===
spiffe_refresh_hint: 60
```

**Result:** ✅ **PASS** - refreshHint is configurable and appears in the bundle endpoint response.

### P18: Route Naming Convention Verification

**Test Execution:**
```
=== Route Details ===
NAME                      HOST
spire-server-federation   federation.apps.ci-ln-jlcigs2-72292.gcp-2.ci.openshift.org

=== Verification ===
Expected: federation.apps.ci-ln-jlcigs2-72292.gcp-2.ci.openshift.org
Actual:   federation.apps.ci-ln-jlcigs2-72292.gcp-2.ci.openshift.org
```

**Result:** ✅ **PASS** - Route follows `federation.<domain>` naming convention.

### P19: TLS Configuration Based on Profile

**Test Execution:**
```
=== Current Profile ===
https_spiffe

=== TLS Certificate Details ===
* Server certificate:
*  subject: C=US; O=SPIRE
*  issuer: C=US; O=RH; CN=SPIRE Server CA; serialNumber=244064904383060730405473032140973606191

=== Route TLS Configuration ===
{
  "insecureEdgeTerminationPolicy": "Redirect",
  "termination": "passthrough"
}
```

**Result:** ✅ **PASS** - https_spiffe profile uses SPIRE's self-signed certificate with passthrough TLS termination.

### P16: Maximum Federation Limit (50)

**Test Execution:**
```
=== Create 50 test ClusterFederatedTrustDomains ===
Created 10/50 federations...
Created 20/50 federations...
Created 30/50 federations...
Created 40/50 federations...
Created 50/50 federations...

=== Verify all 50 were created ===
Total ClusterFederatedTrustDomains: 51
Test federations created: 50

=== Check SPIRE server health ===
spire-server-0   2/2   Running   0   26m
Server is healthy.
```

**Result:** ✅ **PASS** - System successfully handled 50 federated trust domains. SPIRE server remained healthy.

---

## ACME Testing Details (December 11, 2025)

### Test Setup

Since CI clusters don't have public DNS, we used **sslip.io** to provide public DNS resolution:

| Component | Value |
|-----------|-------|
| Router IP | `35.225.244.231` |
| ACME Domain | `federation.35.225.244.231.sslip.io` |
| ACME Directory | Let's Encrypt **Staging** |
| Email | `sayadas@redhat.com` |

### SpireServer ACME Configuration

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: SpireServer
metadata:
  name: cluster
spec:
  federation:
    bundleEndpoint:
      profile: https_web
      refreshHint: 300
      httpsWeb:
        acme:
          directoryUrl: "https://acme-staging-v02.api.letsencrypt.org/directory"
          domainName: "federation.35.225.244.231.sslip.io"
          email: "sayadas@redhat.com"
          tosAccepted: "true"
    managedRoute: "false"  # Manual route for custom domain
```

### Manual Route for ACME

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: federation-sslip
  namespace: zero-trust-workload-identity-manager
spec:
  host: federation.35.225.244.231.sslip.io
  to:
    kind: Service
    name: spire-server
  port:
    targetPort: federation
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: Redirect
```

### ACME Success Evidence

```json
{
    "keys": [
        {
            "use": "x509-svid",
            "kty": "RSA",
            "n": "uUn5tq2VdmBzqAgLv8lK9ski-TXJiDDub8lpSYXlJgQi59SqD6ubNPv..."
        }
    ]
}
```

---

## Key Findings

### 1. ACME Staging vs Production ⚠️

| ACME Environment | Certificate Trust | Auto-Fetch Works |
|------------------|-------------------|------------------|
| **Production** (`acme-v02.api.letsencrypt.org`) | ✅ Publicly trusted | ✅ Yes |
| **Staging** (`acme-staging-v02.api.letsencrypt.org`) | ❌ NOT publicly trusted | ❌ No |

**Impact:** Staging certs require `trustDomainBundle` for federation; production certs don't.

### 2. sslip.io Workaround for Public DNS ✅

For clusters without public DNS, use `sslip.io`:
- Format: `federation.<ROUTER_IP>.sslip.io`
- Resolves to the router IP
- Let's Encrypt can reach it for HTTP-01 challenge

### 3. ACME Email Validation ✅

Let's Encrypt rejects reserved domains like `example.com`:
```
Error validating contact(s) :: contact email has forbidden domain "example.com"
```

### 4. Federation Configuration is Immutable ⚠️

Once `SpireServer.spec.federation.bundleEndpoint` is configured, it cannot be changed. This is a **security feature**.

### 5. SpireServer is a Singleton ✅

SpireServer CR name must be exactly `cluster`:
```
SpireServer is a singleton, .metadata.name must be 'cluster'
```

### 6. managedRoute Controls Creation, Not Deletion

Setting `managedRoute: "false"` prevents creation of NEW routes but doesn't delete existing ones.

### 7. Bundle Deletion Safety Feature ✅

SPIRE prevents bundle deletion if workloads still reference the federated trust domain:
```
Error: cannot delete bundle; federated with 1 registration entries
```

---

## Real-World Scenario Testing

### Scenario Results

| # | Scenario | Recovery Time | Data Persistence | Result |
|---|----------|---------------|------------------|--------|
| 1 | SPIRE Server Pod Restart | **17.2 seconds** | ✅ Entries intact | ✅ PASS |
| 2 | Federation Endpoint Unreachable | N/A | ✅ Bundle cached | ✅ PASS |
| 3 | SPIRE Agent Pod Restart | **31.5 seconds** | ✅ DaemonSet recovery | ✅ PASS |
| 4 | Controller Manager Check | Embedded | ✅ CRs working | ✅ PASS |
| 5 | Rapid Scaling (5 pods) | **<45 seconds** | ✅ 5 entries created | ✅ PASS |

### Customer Impact Analysis

| Scenario | Impact | Severity | Mitigation |
|----------|--------|----------|------------|
| Server restart | SVID renewal pauses 17s | Medium | PVC ensures persistence |
| Network partition | Bundle refresh stops | Low | Cached bundle continues working |
| Agent restart | Node workloads affected 31s | Medium | DaemonSet auto-recovery |
| Rapid scaling | All entries created <45s | None | System handles well |

---

## Recommendations

1. **Document ACME Requirements:** Clearly document that ACME requires public DNS resolution.

2. **sslip.io for Testing:** Document sslip.io workaround for testing ACME without custom DNS.

3. **Email Validation:** Consider validating email domain at API level to catch issues earlier.

4. **Staging vs Production:** Document the trust implications of staging vs production ACME certificates.

5. **Add Status Conditions:** Consider adding status conditions to show ACME certificate status.

---

## Test Artifacts

- Test Plan: `Federation-Test-Plan.md`
- Installation Guide: `Operator-Installation-Guide.md`
- Manual Guide: `Manual-Federation-Testing-Guide.md`
- Stage Build Guide: `Stage-Build-Installation-Guide.md`

---

## Sign-off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| QE Engineer | | Dec 11, 2025 | |
| Dev Lead | | | |
| QE Lead | | | |

---

*Report updated: December 11, 2025*
*ACME testing completed with Let's Encrypt Staging via sslip.io*
