# EX280 — Red Hat Certified OpenShift Administrator — 30-Day Study Plan

> **Target Version:** OpenShift Container Platform (OCP) 4.18  
> **Exam:** EX280 — Performance-based on a live OCP cluster.

This repository provides a consolidated, sequenced curriculum to prepare for the EX280 certification. This plan is derived from official Red Hat objectives and updated specifically for **OCP 4.18**.

---

## ⚠️ What Changed (Important for 4.14/4.16 Users)
If you are referencing older study guides, be aware of these critical 4.18 shifts:

| Topic | 4.14/16 Behavior | 4.18 Behavior |
| :--- | :--- | :--- |
| **Workload** | `DeploymentConfig` common | **`Deployment`** is the standard; `DC` is deprecated. |
| **Networking** | SDN or OVN | **OVN-Kubernetes** is the only supported CNI. |
| **Security** | PSA warnings | **Pod Security Admission (PSA)** is enforced. |

---

## 📅 The 30-Day Roadmap (90–150 Hours)

| Phase | Days | Focus Area |
| :--- | :--- | :--- |
| **1: Foundations** | 1–3 | CLI proficiency, cluster health, project management. |
| **2: Manifests** | 4–5 | Declarative YAML, Kustomize, dry-run strategy. |
| **3: Deployment** | 6–8 | Deployments, Helm 3, application lifecycle. |
| **4: Auth/RBAC** | 9–11 | HTPasswd, Users/Groups, `oc adm` policies. |
| **5: Networking** | 12–14| Routes (TLS), NetworkPolicy, ingress control. |
| **6: Non-HTTP** | 15–16| NodePort, LoadBalancer, non-HTTP traffic. |
| **7: Self-Service**| 17–19| ResourceQuotas, LimitRanges, templates. |
| **8: Operators** | 20–21| OLM, CSVs, Operator management. |
| **9: Security** | 22–23| SCC mapping, ServiceAccounts, Secrets. |
| **10: Storage** | 24–25| StorageClasses, PVCs, StatefulSets. |
| **11: Final Prep** | 26–30| Mock exams, speed drills, gap closure. |

---

## 📁 Repository Structure

```text
ex280-study-plan/
├── README.md               ← This index
├── 01-manage-ocp.md        ← CLI, health, projects
├── 02-resource-manifests.md← YAML, Kustomize
├── 03-deploy-apps.md       ← Deployments, Helm, Templates
├── 04-auth-rbac.md         ← HTPasswd, Users/Groups, RBAC
├── 05-network-security.md  ← Routes (TLS), NetworkPolicy
├── 06-non-http-traffic.md  ← NodePort, LoadBalancer
├── 07-self-service.md      ← Quotas, Limits, Templates
├── 08-operators.md         ← OLM, Subscriptions
├── 09-app-security.md      ← SCCs, ServiceAccounts
├── 10-storage.md           ← StorageClasses, PVCs
├── cheatsheet.md           ← Quick reference
└── mock-exams/             ← Weekly timed drills
