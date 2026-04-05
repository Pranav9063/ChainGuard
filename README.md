# Strategic Security Assessment: Go Application Modernization

**Objective:** Remediate systemic vulnerabilities and secure the software supply chain using Chainguard distroless images, demonstrating an advanced progression from bloated single-stage containers to zero-CVE deployments.

> **Note:** This implementation deliberately uses a custom Go web server built with the [Gin framework](https://github.com/gin-gonic/gin). This choice provides a real-world scenario with external dependencies, which outputs a more meaningful SBOM and supply chain analysis compared to a zero-dependency "Hello World" binary.

---

## 1. Environment & Architecture 
### Setup Decisions & Rationale
- **Operating System:** Windows with WSL2 / Docker Desktop. Rationale: Provides a fully compliant Linux-compatible execution environment without the overhead of maintaining a heavy standalone Linux VM, allowing fluid execution of the required native toolchains.
- **Local Kubernetes:** `kind` (Kubernetes IN Docker). Rationale: Extremely lightweight compared to Minikube or k3s for temporary verifications; it spins up in seconds and behaves identically to production for our testing schemas.
- **Local Registry:** `registry:2` (standard Docker Hub image). Rationale: Official stateless distribution. Allowed simulation of the complete supply chain (pushing, signing, and consuming images locally inside Kind) without managing cloud credentials.

### Tool Selection Justification
- **Vulnerability Scanner:** `grype`. Rationale: Developed by Anchore, it is heavily optimized to detect OS vulnerabilities and application-level dependencies rapidly, making it an industry standard.
- **SBOM Generator:** `syft`. Rationale: Possesses native synergy with Grype, is widely accepted, and effortlessly generates standard compliance formats (SPDX-2.3 JSON).
- **Signing Tool:** `cosign` (Sigstore). Rationale: The leading standard for container signing. Seamlessly integrates with OCI-compatible registries.

---

## 2. Deep Dive: Dockerfile Design Choices
We used three distinct containerization strategies to highlight the progression from worst to best practices:

| Metric | Baseline (`Dockerfile.single`) | Workaround (`Dockerfile.alpine`) | Solution (`Dockerfile.chainguard`) |
| :--- | :--- | :--- | :--- |
| **Image Size** | **~1.79 GB** | **~32 MB** | **~26.3 MB** |
| **CVE Count** | **756 CVEs** | **4 CVEs** | **0 Known CVEs** |
| **Attack Surface** | Full Debian OS | Minimal OS + Shell/APK | **Binary Only (Distroless)** |
| **Package Manager** | `apt` | `apk` | **None** |
| **Shell Access** | `/bin/bash` | `/bin/sh` | **None** |
| **Runs as Non-Root** | Yes (USER directive) | Yes (USER directive) | **Yes (built-in)** |

### Security Findings and Remediation Approach
- **Initial Scan (Single-Stage)**: 756 known vulnerabilities. These largely originate from unused Debian OS packages (like `libpython`, coreutils) inherently bundled into the standard `golang:1.25` developer image.
- **Remediation**: 
  - **App Level**: Attempted upgrading `go.mod` (The Go Gin bindings and dependencies were evaluated to be perfectly up-to-date against latest CVE charts).
  - **OS Level**: Scrapped the baseline image entirely. Changing the runtime base to a multi-stage Alpine build dropped vulnerabilities exponentially to **4 CVEs** (tracked largely via `zlib`). 
  - **Ultimate Patching**: Migrated the final operational runtime base entirely to `cgr.dev/chainguard/static:latest`. The final scan registered **0 Vulnerabilities**, eliminating the software's structural attack surface altogether.

---

## 3. Software Supply Chain Integrity & SBOM Insights
We moved beyond "security by obscurity" to **Security by Transparency**:

- **SBOM (Software Bill of Materials):** Generated via `syft`. The Chainguard SBOM successfully traces our components, explicitly proving that standard OS utility packages (`tzdata`, `ca-certificates-bundle`) coexist cleanly alongside our isolated `github.com/gin-gonic/gin` binary bindings. 
- **Cryptographic Provenance:** Used `cosign` to map cryptographic signatures against our built artifacts. Implementing signatures physically protects against malicious image injection between the build server and deployment cluster.

---

## 4. Q&A and Production Challenges

### Production Challenges for Traditional Container Builds
- **Bloat & Attack Vectors:** Storing compilers and core Linux utilities in production images gives attackers immediate tools to escalate privileges or exfiltrate data (Living off the Land attacks) if they bypass application logic.
- **Slow Scalability:** Pulling 1.8GB images repeatedly severely slows down Kubernetes node scaling during traffic spikes compared to rapid 26MB distroless pulls.
- **Maintenance Nightmare:** Remediating 750+ vulnerabilities manually requires unfeasible operational overhead; distroless models shift these responsibilities left directly to the base maintainer (Chainguard).

### Kubernetes Security & Best Practices Implemented
- **Least Privilege Execution:** The application was built executing under predefined restricted users (UID 65532). Standard distroless execution.
- **Defense-in-Depth Manifest:** Pod configurations enforce `allowPrivilegeEscalation: false` and explicitly `drop: [ALL]` capabilities, dramatically reducing blast radii inside shared clusters.

---

## 5. Quickstart: Build, Scan & Deploy

```bash
# ---------- 1. Build The Images ----------
docker build -f Dockerfile.single -t go-app-single:v1 .
docker build -f Dockerfile.alpine -t go-app-alpine:v1 .
docker build -f Dockerfile.chainguard -t go-app-chainguard:v1 .

# ---------- 2. Scanning ----------
grype go-app-single:v1 -o table > scan-single-report.txt
grype go-app-alpine:v1 -o table > scan-alpine-report.txt
grype go-app-chainguard:v1 -o table > scan-chainguard-report.txt

# ---------- 3. Generate SBOMs ----------
syft go-app-single:v1 -o spdx-json --file sbom-single.spdx.json
syft go-app-alpine:v1 -o spdx-json --file sbom-alpine.spdx.json
syft go-app-chainguard:v1 -o spdx-json --file sbom-chainguard.spdx.json

# ---------- 4. Sign & Push to Local Registry ----------
# NOTE: Requires a local registry running: docker run -d -p 5000:5000 registry:2
docker tag go-app-single:v1 localhost:5000/go-app-single:v1
docker tag go-app-alpine:v1 localhost:5000/go-app-alpine:v1
docker tag go-app-chainguard:v1 localhost:5000/go-app-chainguard:v1

docker push localhost:5000/go-app-single:v1
docker push localhost:5000/go-app-alpine:v1
docker push localhost:5000/go-app-chainguard:v1

# Setup Signing Key
export COSIGN_PASSWORD="chainguard"
cosign generate-key-pair

# Sign all three images directly
cosign sign --yes --key cosign.key localhost:5000/go-app-single:v1
cosign sign --yes --key cosign.key localhost:5000/go-app-alpine:v1
cosign sign --yes --key cosign.key localhost:5000/go-app-chainguard:v1

# Verify the Chainguard image signature
cosign verify --key cosign.pub localhost:5000/go-app-chainguard:v1

# ---------- 5. Standalone Validation ----------
docker run -d -p 8080:8080 --name test-app go-app-chainguard:v1
curl http://localhost:8080   # → "Hello World!"
docker stop test-app && docker rm test-app

# ---------- 6. Kubernetes Deployment ----------
# Ensure your kind cluster 'chainguard-challenge' is active
kind load docker-image go-app-chainguard:v1 --name chainguard-challenge
kubectl apply -f k8s-deployment.yaml
kubectl wait --for=condition=ready pod -l app=secure-go-app --timeout=60s

# Forward the port and test
kubectl port-forward svc/secure-go-app-service 9090:8080 &
curl http://localhost:9090   # → "Hello World!"
```

---

## 6. Conclusion
This exercise demonstrates that modern container security is not about incremental patching routines, but strict architectural decisions. By adopting minimal, distroless images and enforcing supply chain cryptographic integrity, we can build systems that are aggressively secure by design. Ultimately, security is not a feature we continuously add—it is an immutable property we trace and enforce throughout the entire DevOps lifecycle.
