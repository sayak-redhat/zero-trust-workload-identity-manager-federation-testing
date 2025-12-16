# SPIRE Federation Testing - Two Cluster Setup

## 🎯 Overview

This repository contains comprehensive test documentation for **SPIRE Federation** functionality in the **Zero Trust Workload Identity Manager Operator** on OpenShift.

The testing validates bidirectional federation between two OpenShift clusters using SPIFFE/SPIRE trust domains.

---

## 📊 Test Results Summary (Updated Dec 16, 2025)

| Metric | Value |
|--------|-------|
| **Total Tests** | 58 |
| **Passed** | 55 (95%) |
| **N/A** | 1 (2%) |
| **Pending** | 2 (3%) |
| **Failed** | 0 (0%) |

### ✅ Verdict: **READY FOR GA RELEASE**

### 🆕 Latest: STS Clusters + mTLS Testing (Dec 16, 2025)
- AWS STS (IRSA) ↔ Azure STS (Workload Identity) federation
- **mTLS with SPIFFE SVIDs** - Mutual authentication working!

---

## 🏗️ Test Architecture

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
└─────────────────────────────┘         └─────────────────────────────┘
```

---

## 📁 Repository Contents

| File | Description |
|------|-------------|
| `Federation-Test-Plan-2Cluster.md` | Complete test plan with 58+ test cases |
| `Federation-Test-Report-Dec10-2025-2cluster.md` | Test execution report with results |
| `Federation-Testing-Summary-Dec12-2025.md` | Quick summary for team sharing |
| `Manual-Federation-Testing-Guide.md` | Step-by-step testing guide |
| `Federation-Testing-Simple-Guide.md` | Simplified guide for beginners |
| `Operator-Installation-Guide.md` | Operator installation instructions |
| **`mTLS-Federation-Testing-Guide.md`** 🆕 | **mTLS testing with SPIFFE SVIDs** |

---

## 🧪 Test Categories

### Positive Tests (21/22 Passed ✅)

| ID Range | Category | Status |
|----------|----------|--------|
| P1, P1b | Core Federation (2 methods) | ✅ All PASS |
| P2-P8 | Route, Bundle, Endpoint | ✅ All PASS |
| P11-P12 | ACME Integration | ✅ All PASS |
| P13-P15 | Manual Cert/Route | ✅ All PASS |
| P16-P19 | Scalability & Config | ✅ All PASS |
| P20 | Cross-Cluster Workload | ✅ PASS |
| **P21-P22** | **Azure ↔ GCP Cross-Cloud** 🆕 | ✅ All PASS |

### Negative Tests (15/15 Passed ✅)

| ID Range | Category | Status |
|----------|----------|--------|
| N1-N8 | Core Negative | ✅ All PASS |
| N9-N12 | ACME Negative | ✅ All PASS |
| N13-N15 | Manual Cert Negative | ✅ All PASS |

### Customer Scenarios (9/10 Passed ✅)

| ID Range | Category | Status |
|----------|----------|--------|
| C1-C4 | Network/Config Issues | ✅ All PASS |
| C5-C6 | Cert Rotation/Retry | ✅ All PASS |
| C8 | Pod Restart Recovery | ✅ PASS |
| **C3-Azure, C8-Azure** | **Azure ↔ GCP Resilience** 🆕 | ✅ All PASS |

### Workload Lifecycle Tests (3/3 Passed ✅)

| ID | Test | Status | Key Finding |
|----|------|--------|-------------|
| W1 | Workload Pod Restart | ✅ PASS | New pod gets FederatesWith automatically |
| W2 | SPIRE Agent Restart | ✅ PASS | Entry persists in datastore |
| W3 | Workload Scaling (1→3) | ✅ PASS | **ALL replicas get FederatesWith (100%)** |

### STS + mTLS Tests (7/7 Passed ✅) 🆕

| ID | Test | Status | Key Finding |
|----|------|--------|-------------|
| STS-1 | AWS STS (IRSA) cluster | ✅ PASS | CCO Mode: Manual |
| STS-2 | Azure STS (WI) cluster | ✅ PASS | CCO Mode: Manual |
| STS-3 | Operator on STS with proxy | ✅ PASS | CA bundle config required |
| STS-4 | Cross-cloud STS federation | ✅ PASS | Bundle exchange successful |
| STS-5 | Workloads on STS | ✅ PASS | FederatesWith configured |
| E2E-HTTP | Cross-cloud HTTP | ✅ PASS | Azure → AWS connectivity |
| **E2E-mTLS** | **Mutual TLS with SPIFFE** | ✅ PASS | **TLSv1.3 + AES-256-GCM** |

### Pending/Special Cases

| Test | Status | Notes |
|------|--------|-------|
| P9 (ACME Auto-Renewal) | ⏳ Pending | Requires ~30 days to verify |
| P10 (https_web Auto-Fetch) | ⚠️ N/A | Staging ACME certs not publicly trusted |
| C7 (Operator Upgrade) | ⏳ Pending | Requires actual operator upgrade |

---

## 🔑 Key Findings

### 1. Federation Methods Tested

| Method | Description | Status |
|--------|-------------|--------|
| Method 1 | `SpireServer.spec.federation.federatesWith` (inline) | ✅ Working |
| Method 2 | `ClusterFederatedTrustDomain` CR (separate) | ✅ Working |

### 2. Federation Profiles Tested

| Profile | Description | Status |
|---------|-------------|--------|
| `https_spiffe` | SPIRE self-signed certificates | ✅ Working |
| `https_web` (ACME) | Let's Encrypt certificates | ✅ Working |
| `https_web` (servingCert) | cert-manager certificates | ✅ Working |

### 3. Important Discoveries

- **SpireServer is a Singleton**: Must be named `cluster`
- **Federation Profile is Immutable**: Cannot change once set (security feature)
- **managedRoute Behavior**: Controls route creation, not deletion
- **ACME Staging**: Certificates not publicly trusted, require manual `trustDomainBundle`
- **Bundle Deletion Safety**: SPIRE prevents deletion if workloads reference the trust domain
- **No Hard Federation Limit**: 55 federations created successfully (no 50 limit!)

### 4. Scalability & Resilience

- ✅ Successfully tested with **55 federated trust domains** (no hard limit)
- ✅ SPIRE server remained healthy under load
- ✅ Pod restart recovery in ~17 seconds
- ✅ Bundle sync every ~75 seconds
- ✅ Graceful handling of network failures
- ✅ **NEW:** Workload scaling: 100% of replicas get FederatesWith (W3)
- ✅ **NEW:** SPIRE agent restart: Workload entries persist (W2)

---

## 🛠️ Test Environment

| Component | Details |
|-----------|---------|
| **Platform** | OpenShift 4.x on GCP and AWS |
| **Operator** | Zero Trust Workload Identity Manager |
| **SPIRE Version** | Latest (via operator) |
| **Test Dates** | December 10-16, 2025 |

### Clusters Used

| Day | Cluster 1 | Cluster 2 | Cloud |
|-----|-----------|-----------|-------|
| Day 1-2 | `apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org` | `apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org` | GCP |
| Day 3 | `apps.ci-ln-lh14qqk-72292.origin-ci-int-gce.dev.rhcloud.com` | `apps.ci-ln-7bgj5qt-72292.origin-ci-int-gce.dev.rhcloud.com` | GCP |
| Day 4 | `apps.ci-ln-dspcs42-76ef8.aws-2.ci.openshift.org` | `apps.ci-ln-32vmc0b-72292.origin-ci-int-gce.dev.rhcloud.com` | AWS + GCP |
| Day 5 (AM) | `apps.ci-ln-rss07bt-1d09d.ci2.azure.devcluster.openshift.com` | `apps.ci-ln-gmy2nx2-72292.gcp-2.ci.openshift.org` | Azure + GCP |
| **Day 5 (PM)** 🆕 | `apps.ci-ln-g2p0vz2-76ef8.aws-4.ci.openshift.org` (STS) | `apps.ci-ln-7vs7nxk-1d09d.ci2.azure.devcluster.openshift.com` (STS) | **AWS STS + Azure STS** |

---

## 📋 How to Use These Documents

### For QE Engineers
1. Use `Federation-Test-Plan-2Cluster.md` to understand test cases
2. Follow manual test commands in each test section
3. Update results in your own test report
4. **All test cases include retest commands** - look for "Retest Commands" sections

### For Developers
1. Review negative tests for API validation requirements
2. Check key findings for known behaviors
3. Use as reference for documentation

### For Release Managers
1. Review `Federation-Test-Report-Dec10-2025-2cluster.md` for GA readiness
2. Check test statistics and pass rates
3. Review recommendations section

### For Team Sharing
1. Use `Federation-Testing-Summary-Dec12-2025.md` for quick overview
2. Share on Teams/Slack for status updates

### ✅ Prerequisites Checklist

- [x] Both clusters have Zero Trust Workload Identity Manager operator installed
- [x] Operator CSV shows "Succeeded" status
- [x] SPIRE components are running on both clusters
- [x] Network connectivity between clusters
- [x] Kubeconfig files available for both clusters

---

## 🚀 Quick Start Commands

```bash
# Check federation status
oc get clusterfederatedtrustdomains -o wide

# List trust bundles
oc exec -n zero-trust-workload-identity-manager spire-server-0 -c spire-server -- /spire-server bundle list

# Test federation endpoint
curl -k -s https://federation.<APP_DOMAIN> | head -c 200

# Check SPIRE server health
oc exec -n zero-trust-workload-identity-manager spire-server-0 -c spire-server -- /spire-server healthcheck

# Check SPIRE entries with federation
oc exec -n zero-trust-workload-identity-manager spire-server-0 -c spire-server -- /spire-server entry show
```

---

## 📈 Test Progress Timeline

| Date | Tests Executed | Cumulative Pass Rate |
|------|----------------|---------------------|
| Dec 10 | P1-P8, N1-N8 (Core Federation) | 16/16 (100%) |
| Dec 11 | P8-P12, P16-P19, N9-N12 (ACME & Config) | 29/30 (97%) |
| Dec 12 | P1b, P13-P15, P20, N13-N15, C1-C8 | 41/44 (93%) |
| Dec 16 (AM) | P21-P22, C3-Azure, C8-Azure (Azure ↔ GCP) | 45/48 (94%) |
| Dec 16 (AM) | W1-W3 (Workload Lifecycle Tests) | 48/51 (94%) |
| **Dec 16 (PM)** 🆕 | **STS-1 to STS-5, E2E-HTTP, E2E-mTLS** | **55/58 (95%)** |

---

## 📞 Contact

| Role | Name |
|------|------|
| **QE Engineer** | Sayak Das |
| **Test Dates** | December 10-16, 2025 |

---

## 📄 License

Internal Red Hat Testing Documentation

---

*Last Updated: December 16, 2025*
