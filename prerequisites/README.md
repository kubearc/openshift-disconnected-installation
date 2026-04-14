# 📋 Prerequisites – OpenShift Disconnected Installation

Before starting the OpenShift disconnected installation, it is important to ensure that all prerequisites are properly met.

This guide covers the **basic requirements, tools, and knowledge** needed to successfully follow this course.

---

## 🎯 Why Prerequisites Matter

In disconnected environments:
- There is **no internet access**
- Every component must be **planned and verified in advance**
- Missing even a small requirement can **break the entire setup**

---

## 📊 Prerequisites Overview

| Category              | Requirement                          | Description |
|----------------------|--------------------------------------|-------------|
| 🧠 Knowledge         | Linux Basics                         | Basic commands, file system, permissions |
| 🧠 Knowledge         | Container Concepts                   | Understanding of containers and images |
| 🧠 Knowledge         | Kubernetes Basics                    | Pods, services, basic architecture |
| 💻 System            | Linux Machine                        | RHEL / CentOS / Ubuntu (recommended) |
| 💻 System            | Minimum Resources                    | At least 64 GB RAM, 20 VCPU (lab setup) |
| 🌐 Network           | Internal Network Setup               | Proper DNS and hostname resolution |
| 🧰 Tools             | Podman / Docker                      | For container image handling |
| 🧰 Tools             | oc CLI                               | OpenShift command-line tool |
| 🧰 Tools             | openshift-install                    | Installer binary |
| 📦 Registry          | Private Image Registry               | Required for disconnected image storage |
| 📦 Registry          | Image Mirroring Tools                | oc mirror|

---

## 🔧 Tools to Download (Before Disconnect)

Make sure to download all required tools and binaries **before moving to a disconnected environment**:

- OpenShift Installer (`openshift-install`)
- OpenShift CLI (`oc`)
- Container tools (Podman / Docker)
- Required container images

---

## ⚠️ Important Notes

- All dependencies must be **locally available**
- Ensure **time synchronization (NTP)** across systems
- Verify **DNS and hostname resolution**
- Maintain proper **network connectivity within the cluster**

---

## 🚀 Next Step

Once all prerequisites are ready, proceed to:

👉 `../lab-setup`

---

💡 Tip: Take time to verify each prerequisite carefully — it will save hours of troubleshooting later.
