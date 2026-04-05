\# Strategic Security Assessment: Go Application Modernization

\*\*Candidate:\*\* Pranav

\*\*Objective:\*\* Remediate systemic vulnerabilities and secure the software supply chain using Chainguard distroless images.



\---



\## 1. Executive Summary: The "Before vs. After" Narrative

The primary goal of this challenge was to fundamentally transform the application's security posture. Instead of engaging in the "vulnerability treadmill"—the endless cycle of patching hundreds of OS-level vulnerabilities—we achieved remediation by \*\*reducing the attack surface\*\*. This approach shifts security left by eliminating vulnerabilities at build time rather than reacting to them in production.



\## 2. Environment \& Tooling Rationale

\* \*\*Virtualization:\*\* Windows Subsystem for Linux (WSL2). This provided a native Linux kernel for Docker and Kubernetes performance while maintaining developer productivity on a Windows host.

\* \*\*Orchestration:\*\* `kind` (Kubernetes IN Docker) for lightweight, reproducible cluster deployments.

\* \*\*Registry:\*\* Local Docker Registry (`registry:2`) on port 5000 to validate private image signing and supply chain workflows.



\## 3. Comparative Analysis: The 3-Point Gradient

All production-ready images were built using \*\*multi-stage Docker builds\*\* to ensure that only the compiled binary is included in the final runtime image. The statically compiled nature of Go further enables these minimal images by \*\*eliminating unnecessary runtime dependencies\*\*.



| Metric | Baseline (`golang:1.21`) | Workaround (`alpine:3.19`) | Solution (`chainguard/static`) |

| :--- | :--- | :--- | :--- |

| \*\*Image Size\*\* | \*\*1.79 GB\*\* | \*\*21 MB\*\* | \*\*16 MB\*\* |

| \*\*CVE Count\*\* | \*\*High (300+)\*\* | \*\*Medium (5-15)\*\* | \*\*Zero Known OS CVEs\*\*\* |

| \*\*Attack Surface\*\* | Full Debian OS | Minimal OS + Shell/APK | \*\*Binary Only (Distroless)\*\* |

| \*\*Package Manager\*\* | `apt` | `apk` | \*\*None\*\* |

| \*\*Shell Access\*\* | `/bin/bash` | `/bin/sh` | \*\*None\*\* |



\*\\\*At the time of scanning.\*



\### Security Findings \& Impact

This strategy \*\*reduced image size by over 99%\*\* and eliminated the majority of non-actionable CVEs.

\* \*\*The Baseline Risk:\*\* Revealed critical vulnerabilities like \*\*CVE-2023-4911 (Looney Tunables)\*\*. These exist in the "noise" of the OS; security teams waste significant resources triaging "ghost" CVEs that the application never actually uses.

\* \*\*The Alpine Limitation:\*\* While Alpine reduces size, it still includes a package manager and shell. These are frequently leveraged in \*\*"living off the land" attacks\*\* to download malware or pivot within a network.

\* \*\*The Chainguard Advantage:\*\* By removing the shell and package manager, we \*\*eliminate entire classes of vulnerabilities\*\* and drastically reduce the available exploit surface. Because there is no shell or package manager, attackers cannot easily execute commands or install additional tooling even after gaining access.



\## 4. Software Supply Chain Integrity

We moved beyond "security by obscurity" to \*\*Security by Transparency\*\*:

\* \*\*SBOM (Software Bill of Materials):\*\* Generated via `syft`. Enables compliance, detects vulnerable sub-dependencies, and supports rapid incident response.

\* \*\*Cryptographic Provenance:\*\* Used `cosign` to sign all images and attach the SBOM as a signed attestation.

\* \*\*Reproducibility:\*\* The build process is \*\*designed to be reproducible\*\*, ensuring deterministic outputs and reducing the risk of "shadow" dependencies.



\## 5. Deployment \& Validation

\### Standalone Validation

`docker run -d -p 8080:8080 go-app-chainguard:v1`  

\*\*→ Verified:\*\* Functional via HTTP 200 OK response.



\### Kubernetes Orchestration

Deployed to the `chainguard-challenge` cluster with a "Defense-in-Depth" manifest:

\* \*\*Security Context:\*\* Enforced `runAsNonRoot: true` (UID 65532), `readOnlyRootFilesystem: true`, and `allowPrivilegeEscalation: false`. 

\* \*\*Impact:\*\* These controls ensure that even if the container is compromised, the impact is limited by enforcing least privilege and preventing filesystem or privilege escalation attacks.

\* \*\*Challenge Overcome:\*\* Resolved an `ErrImageNeverPull` error by explicitly sideloading the image into the node cache via `kind load docker-image`.



\## 6. Strategic Recommendations

For production workloads, the "debuggability" of a shell is a security liability. We recommend Chainguard images for production workloads to \*\*reduce operational overhead, improve security posture, and minimize long-term vulnerability management costs.\*\*



\## 7. Conclusion

This exercise demonstrates that modern container security is not about incremental patching, but architectural decisions. By adopting minimal, distroless images and enforcing supply chain integrity, we can build systems that are secure by design. While this approach significantly improves security posture, continuous monitoring and regular scanning remain essential as new vulnerabilities are discovered over time.



Ultimately, security is not a feature we add—it is a property we design into the system from the beginning.

