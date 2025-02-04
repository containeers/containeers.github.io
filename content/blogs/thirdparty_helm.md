---
title: "Managing Third-Party Helm Charts"
date: 2025-01-31T18:00:00
draft: false
github_link: "https://github.com/gurusabarish/hugo-profile"
author: "Sandarsh"
tags:
  - Helm
  - Kubernetes
  - DevOps
image: /images/post.jpg
description: "Best practices for managing third-party Helm charts in your enterprise"
toc: 
---

# Managing Third-Party Helm Charts in Your Enterprise: Best Practices
Helm has become Kubernetes's de facto package manager, allowing teams to deploy applications efficiently. However, enterprises often rely on third-party Helm charts from sources like GitHub, Bitnami, or Prometheus. While these charts offer convenience, using them without proper governance can expose organizations to security risks, operational inefficiencies, and compliance issues.

Consider a scenario where you rely on the ServiceMonitor CR or specific common labels for monitoring, but the third-party Helm chart does not provide the necessary templates or customization options in `helm templates`. Additionally, if you need to manage the chart in Git and deploy it using GitOps in an air-gapped environment, maintaining consistency and security becomes even more challenging. This is where an Overlay Chart comes to the rescue.

What is a Overlay Chart?
========================

A Overlay Chart is a Helm chart that acts as a layer around a third-party Helm chart, allowing customization and compliance enforcement without modifying the upstream chart. This method provides better control, maintainability, and consistency.

Why Use Overlay Charts?
=======================

✔ Avoids direct modifications to third-party charts, making future updates easier.  
✔ Allows enterprises to apply custom configurations, security policies, and monitoring settings.  
✔ Ensures internal compliance standards are met before deploying third-party applications.

![](https://miro.medium.com/v2/resize:fit:1400/0*Gy_wQXQfn93tuyO7)

Photo by [Hannah Busing](https://unsplash.com/@hannahbusing?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

Steps for enterprise usage.
===========================

1. Use an Internal Helm Repository
===================================

Why?
====

Directly pulling third-party Helm charts from public repositories like ArtifactHub, Bitnami, Quay, etc increases security risks and dependency on external sources.

Best Practices:
===============

✔ Store third-party Helm charts in an internal repository such as Harbor, JFrog Artifactory, or AWS ECR.  
✔ Only approved, security-scanned charts should be stored in the internal registry.  
✔ Automate chart version synchronization from upstream sources but with approval processes.  
✔ Maintain audit logs of chart usage and updates.

Example: Pulling and Storing a Third-Party Chart
------------------------------------------------

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx  
helm pull ingress-nginx/ingress-nginx --version 4.12.0  
helm push ingress-nginx-4.12.0.tgz oci://my-internal-registry/myhelm
```

2. Overlay Chart with Pinned Chart Versions
============================================

Why?
====

Using the latest or unpinned versions can introduce breaking changes or security vulnerabilities.

Best Practices:
===============

✔ Always specify a pinned version in `Chart.yaml`.  
✔ Maintain a staging/testing environment before upgrading charts in production.  
✔ Use Helm Diff Plugin to compare releases before upgrades.

Below is the option on creating overlay chart and associate versions

```bash
#Sample Step to create overlay chart  
helm create ingress-nginx-overlay  
tree ingress-nginx-overlay  
ingress-nginx-overlay  
├── Chart.yaml  
├── charts  
├── templates  
│   ├── NOTES.txt  
│   ├── _helpers.tpl  
│   ├── deployment.yaml  
│   ├── hpa.yaml  
│   ├── ingress.yaml  
│   ├── service.yaml  
│   ├── serviceaccount.yaml  
│   └── tests  
│       └── test-connection.yaml  
└── values.yaml  
  
4 directories, 10 files  
#remove template files which you may not need it for eg  
find templates -type f -name "\*yaml" -exec rm {} \\;  
rm values.yaml; touch values.yaml
```

Example: Pinning a Chart Version in `Chart.yaml`
------------------------------------------------

```yaml
#Update Chart.yaml with dependencies  
dependencies:  
  - name: ingress-nginx  
    version: "4.12.0"  
    repository: "oci://my-internal-registry/myhelm"
```

Example: Chart.yaml file
------------------------

```yaml
apiVersion: v2  
name: ingress-nginx-overlay  
description: "An overlay Helm chart for customizing Ingress Nginx"  
type: application  
version: 0.1.0  # The version of this overlay chart  
appVersion: "4.12.0"  # Can track actual ingress specific versions  
  
# Declare Dependencies (Traditional Helm Repo or OCI)  
dependencies:  
  - name: ingress-nginx  
    version: "4.12.0"  # Pin a specific version to avoid breaking changes  
    repository: "oci://my-internal-registry/myhelm"  # Use OCI if applicable  
  
# Annotations for Documentation & GitOps Best Practices  
annotations:  
  category: "Networking"  
  createdBy: "Platform Team"  
  documentation: "https://kubernetes.github.io/ingress-nginx"  
  support: "support@yourcompany.com"  
  sourceRepo: "https://github.com/kubernetes/ingress-nginx"  
  oci-dependencies: "my-internal-registry/myhelm/ingress-nginx:4.12.0"  # Track OCI dependencies explicitly  
  
# Secure & Traceable Maintainers  
maintainers:  
  - name: "Platform Team"  
    email: "platform@yourcompany.com"  
    url: "https://yourcompany.com/platform-security"  
  
# Keywords for Better Searchability & Tracking  
keywords:  
  - "ingress"  
  - "nginx"  
  - "overlay"  
  
# Dependencies Are Managed with `helm dependency update`  
# This ensures we always use an internally approved version.
```

Overlay a sample extra resource configuration
---------------------------------------------

Lets consider an example you need to add an extra resource to the default chart. i.e. add an extra kubernetes service and would like to expose internal 8080 port than the default Controller Service

```yaml
#Content of resource ingress-nginx-overlay/templates/extra-controller-service.yaml  
{{- if .Values.extraControllerService.enabled }}  
apiVersion: v1  
kind: Service  
metadata:  
  name: {{ .Release.Name }}-extra-controller-service  
  namespace: {{ .Release.Namespace }}  
  labels:  
    app.kubernetes.io/name: ingress-nginx  
    app.kubernetes.io/instance: {{ .Release.Name }}  
spec:  
  type: {{ .Values.extraControllerService.type }}  
  selector:  
    app.kubernetes.io/name: ingress-nginx  
    app.kubernetes.io/instance: {{ .Release.Name }}  
  ports:  
    {{- range .Values.extraControllerService.ports }}  
    - name: {{ .name }}  
      port: {{ .port }}  
      targetPort: {{ .targetPort }}  
      protocol: {{ .protocol }}  
    {{- end }}  
{{- end }}  
  
#Update Values.yaml  
extraControllerService:  
  enabled: true  
  type: ClusterIP  # Change to ClusterIP/NodePort if required  
  ports:  
    - name: extra-http  
      port: 8080  
      targetPort: 80  
      protocol: TCP
```

Example: Helm Diff for Upgrade Validation
-----------------------------------------

```bash
helm diff upgrade ingress-nginx-overlay my-internal-repo/ingress-nginx-overlay
```

3. Scan for Security Vulnerabilities
=====================================

Why?
====

Third-party charts might introduce security vulnerabilities through container images or misconfigurations.

Best Practices:
===============

✔ Use Trivy or Grype to scan Helm charts & the associated container image for vulnerabilities before deployment.  
✔ Enforce non-root execution and minimal container privileges.  
✔ Use Gatekeeper or Kyverno to enforce security standards.

Example: Scanning a Helm Chart for Vulnerabilities
--------------------------------------------------

```bash
trivy helm my-internal-registry/myhelm/ingress-nginx
```

4. Override Default Configurations Securely
============================================

Why?
====

Default values in third-party Helm charts may not align with enterprise security and performance policies.

Best Practices:
===============

✔ Store overrides in a dedicated `values.yaml` file instead of modifying the upstream chart.  
✔ Disable unused features to reduce the attack surface.  
✔ Ensure sensitive values (e.g., passwords, API keys) are managed via Sealed Secrets or Vault.

Example: Secure `values.yaml` Override
--------------------------------------

```yaml
controller:  
  service:  
    type: ClusterIP  # Prevent exposure to public internet  
  securityContext:  
    runAsUser: 1001  
    runAsNonRoot: true  
    allowPrivilegeEscalation: false
```

5. Implement CI/CD and GitOps for Helm Deployments
===================================================

Why?
====

Manual Helm deployments lead to inconsistencies and lack of traceability.

Best Practices:
===============

✔ Use GitOps tools like ArgoCD or FluxCD to automate Helm deployments. ✔ Integrate Helm into CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins).  
✔ Enable automated rollback in case of failed upgrades.

Example: ArgoCD/Rancher Fleet Application for Helm
--------------------------------------------------

```yaml
apiVersion: argoproj.io/v1alpha1  
kind: Application  
metadata:  
  name: ingress-nginx-overlay  
  namespace: argocd  # Ensure ArgoCD is installed in this namespace  
spec:  
  project: default  # Change if using a custom ArgoCD project  
  source:  
    repoURL: "https://github.com/your-username/ingress-nginx-overlay.git"  
    targetRevision: main  # Adjust based on your branch/tag  
    path: ingress-nginx-overlay  # Path where the overlay Helm chart is stored  
    helm:  
      valueFiles:  
        - values.yaml  # Use this file to pass custom values  
  destination:  
    server: https://kubernetes.default.svc  # Deploy to the in-cluster Kubernetes API  
    namespace: ingress-nginx  # Target namespace for deployment  
  syncPolicy:  
    automated:  
      prune: true  # Automatically remove resources not in Git  
      selfHeal: true  # Auto-sync if drift is detected  
    syncOptions:  
      - CreateNamespace=true  # Ensure the namespace exists before deploying
```

6. Monitor and Audit Helm Releases
===================================

Why?
====

Lack of visibility into Helm releases can lead to undetected misconfigurations or security incidents.

Best Practices:
===============

✔ Use Prometheus, Loki, or ELK for monitoring and logging Helm deployments.  
✔ Periodically audit Helm releases using `helm history`.  
✔ Set up alerts for misconfigurations and policy violations.

Example: Checking Helm Release History
--------------------------------------

```bash
helm history ingress-nginx-overlay
```

7. Enable Rollback and Backup Strategies
=========================================

Why?
====

Failed Helm upgrades can lead to downtime and data loss.

Best Practices:
===============

✔ Enable Helm rollback capabilities.  
✔ Use Velero to back up Kubernetes resources before upgrades.  
✔ Define health checks and readiness probes for controlled rollouts.

Example: Helm Rollback Command
------------------------------

```bash
helm rollback ingress-nginx-overlay 1
```

Example: Backup Helm Resources with Velero
------------------------------------------

```bash
velero backup create ingress-nginx-overlay-backup --include-namespaces web
```

Conclusion
==========

Managing third-party Helm charts in an enterprise requires a structured approach to security, governance, and automation. The table below summarizes the key best practices:

| Best Practice                         | Key Benefit                         |  
| ------------------------------------- | ----------------------------------- | 
| Use an Internal Helm Repository       | Secure and control charts           |  
| Pin and Validate Chart Versions       | Prevent unexpected failures         |  
| Scan for Security Vulnerabilities     | Identify risks before deploy        |  
| Override Default Configurations       | Align with security policies        |  
| Implement CI/CD and GitOps            | Ensure consistency & traceability   |  
| Monitor and Audit Helm Releases       | Improve visibility & troubleshooting| 
| Enable Rollback and Backup Strategies | Reduce downtime & data loss         |  

Organizations can ensure secure, stable, and scalable Kubernetes workloads by implementing these best practices.       