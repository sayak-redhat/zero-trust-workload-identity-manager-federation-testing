# SPIRE Federation Testing - Two Cluster Setup

## 🎯 Overview

This repository contains comprehensive test documentation for **SPIRE Federation** functionality in the **Zero Trust Workload Identity Manager Operator** on OpenShift.

The testing validates bidirectional federation between two OpenShift clusters using SPIFFE/SPIRE trust domains.

---

## 📊 Test Results Summary

| Metric | Value |
|--------|-------|
| **Total Tests** | 35 |
| **Passed** | 33 (94%) |
| **N/A** | 1 (3%) |
| **Future** | 1 (3%) |
| **Failed** | 0 (0%) |

### ✅ Verdict: **READY FOR GA RELEASE**

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
| `Federation-Test-Plan-2Cluster.md` | Complete test plan with 35 test cases |
| `Federation-Test-Report-Dec10-2025-2cluster.md` | Test execution report with results |

---

## 🧪 Test Categories

### Positive Tests (17 Passed)

| ID Range | Category | Status |
|----------|----------|--------|
| P1-P8 | Core Federation | ✅ All PASS |
| P11-P12 | ACME Integration | ✅ All PASS |
| P13-P15 | Manual Cert/Route | ✅ All PASS |
| P16 | Scalability (50 federations) | ✅ PASS |
| P17-P19 | Configuration | ✅ All PASS |

### Negative Tests (15 Passed)

| ID Range | Category | Status |
|----------|----------|--------|
| N1-N8 | Core Negative | ✅ All PASS |
| N9-N12 | ACME Negative | ✅ All PASS |
| N13-N15 | Manual Cert Negative | ✅ All PASS |

### Special Cases

| Test | Status | Notes |
|------|--------|-------|
| P9 (ACME Auto-Renewal) | 📅 Future | Requires ~30 days to verify |
| P10 (https_web Auto-Fetch) | ⚠️ N/A | Staging ACME certs not publicly trusted |

---

## 🔑 Key Findings

### 1. Federation Profiles Tested

| Profile | Description | Status |
|---------|-------------|--------|
| `https_spiffe` | SPIRE self-signed certificates | ✅ Working |
| `https_web` (ACME) | Let's Encrypt certificates | ✅ Working |

### 2. Important Discoveries

- **SpireServer is a Singleton**: Must be named `cluster`
- **managedRoute Behavior**: Controls route creation, not deletion
- **ACME Staging**: Certificates not publicly trusted, require manual `trustDomainBundle`
- **Bundle Deletion Safety**: SPIRE prevents deletion if workloads reference the trust domain

### 3. Scalability

- ✅ Successfully tested with **50 federated trust domains**
- ✅ SPIRE server remained healthy under load

---

## 🛠️ Test Environment

| Component | Details |
|-----------|---------|
| **Platform** | OpenShift 4.x on GCP |
| **Operator** | Zero Trust Workload Identity Manager |
| **SPIRE Version** | Latest (via operator) |
| **Test Dates** | December 10-11, 2025 |

### Clusters Used

| Day | Cluster 1 | Cluster 2 |
|-----|-----------|-----------|
| Day 1 | `apps.ci-ln-1pxsfpt-72292.gcp-2.ci.openshift.org` | `apps.ci-ln-gpdipqb-72292.gcp-2.ci.openshift.org` |
| Day 2 | `apps.ci-ln-lh14qqk-72292.origin-ci-int-gce.dev.rhcloud.com` | `apps.ci-ln-7bgj5qt-72292.origin-ci-int-gce.dev.rhcloud.com` |
| Day 3 | `apps.ci-ln-jlcigs2-72292.gcp-2.ci.openshift.org` | `apps.ci-ln-ir8ikrb-72292.gcp-2.ci.openshift.org` |

---

## 📋 How to Use These Documents

### For QE Engineers
1. Use `Federation-Test-Plan-2Cluster.md` to understand test cases
2. Follow manual test commands in each test section
3. Update results in your own test report

### For Developers
1. Review negative tests for API validation requirements
2. Check key findings for known behaviors
3. Use as reference for documentation

### For Release Managers
1. Review `Federation-Test-Report-Dec10-2025-2cluster.md` for GA readiness
2. Check test statistics and pass rates
3. Review recommendations section

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
```

---

## 📞 Contact

| Role | Name |
|------|------|
| **QE Engineer** | Sayak Das |
| **Test Date** | December 10-11, 2025 |

---

## 📄 License

Internal Red Hat Testing Documentation

---

*Last Updated: December 11, 2025*

