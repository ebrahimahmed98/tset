Copy everything in this single block to a file like README_Windows_Minikube_Kubeval.txt and follow step by step.
============================================================
README — 3 Tier Movie App on Minikube (Windows) + Kubeval
Role: Senior Kubernetes Architect & DevOps Trainer
============================================================

[Image: High-level diagram — React (web) -> Node (API) -> MongoDB; Services (react-svc/node-svc/mongo-svc), NodePorts for host access]

CONTENTS
  A. Local Environment Setup (Windows)
  B. Secrets Preparation (Base64 Encoding)
  C. YAML Validation (Kubeval)
  D. Deployment Execution (kubectl apply)
  E. Validate & Access
  F. Cleanup
  G. Appendix: Troubleshooting & Optional Ingress

----------------------------------------------------------------
A. LOCAL ENVIRONMENT SETUP (WINDOWS)
----------------------------------------------------------------
Goal: Install kubectl, Minikube, and (optionally) Chocolatey; then start a local Kubernetes cluster.

1) Install Chocolatey (optional but recommended)
   - Open PowerShell **as Administrator** and run Chocolatey’s official install script:
     # NOTE: Verify execution policy and inspect the script if required by your org.
     See: https://chocolatey.org/install for the current instructions. [1](https://chocolatey.org/install)

2) Install kubectl on Windows
   Option 1 — Chocolatey:
     choco install kubernetes-cli
     # NOTE: If you prefer winget or scoop, feel free. Ensure kubectl version within one minor of your cluster.
     Official docs (Windows methods: direct, Chocolatey, winget): https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/ [2](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

   Option 2 — Direct download (official method):
     - PowerShell:
       curl.exe -LO "https://dl.k8s.io/release/stable.txt"
       $v = Get-Content .\stable.txt
       curl.exe -LO "https://dl.k8s.io/release/$v/bin/windows/amd64/kubectl.exe"
       # Move kubectl.exe to a PATH folder and verify:
       kubectl version --client
       # NOTE: If you already have Docker Desktop, it may have added its own kubectl; prefer latest and adjust PATH per docs. [2](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

3) Install Minikube on Windows
   Option 1 — Chocolatey:
     choco install minikube
     # NOTE: Run in an admin PowerShell. This is maintained in Chocolatey community repository. [3](https://community.chocolatey.org/packages/Minikube)
   Option 2 — Download EXE (official doc page has options):
     - Minikube start docs (Windows/Chocolatey/EXE): https://minikube.sigs.k8s.io/docs/start/ (Windows tab) [4](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2Fchocolatey)

4) Start Minikube (Docker driver or Hyper V)
   - With Docker Desktop:
     minikube start --driver=docker
   - With Hyper V:
     minikube start --driver=hyperv
   # NOTE: Ensure you have ≥2 CPUs, ≥2GB RAM, and ~20GB disk free, and a container/VM runtime. [4](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2Fchocolatey)

[Image: Screenshot placeholder — “minikube start” in Windows PowerShell, showing cluster creation succeeded]

----------------------------------------------------------------
B. SECRETS PREPARATION (BASE64 ENCODING)
----------------------------------------------------------------
Goal: Create Base64 strings for sensitive values used in mongo_secret.yaml.

Windows (PowerShell) one liners:
  # Encode "movieadmin"
  [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("movieadmin"))
  # Encode "supersecret"
  [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("supersecret"))
  # Encode "movieapp"
  [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("movieapp"))
  # Example connection string (match Service name/port):
  $uri = "mongodb://movieadmin:supersecret@mongo-svc:27017/movieapp?authSource=admin"
  [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($uri))
# NOTE: Replace values per your project. Do NOT paste raw secrets into YAML; always Base64 them.
# References with examples: ShellHacks, PowerShellFAQs. [5](https://www.shellhacks.com/base64-powershell-decode-encode-one-liners-cheatsheet/)[6](https://powershellfaqs.com/convert-string-to-base64-in-powershell/)

[Image: Diagram placeholder — “Where secrets live in K8s: Secret -> env in Node -> Mongo connection”]

----------------------------------------------------------------
C. YAML VALIDATION (KUBEVAL)
----------------------------------------------------------------
Goal: Validate manifests before applying, to catch schema errors early.

Install Kubeval on Windows:
  Option 1 — Chocolatey:
    choco install kubeval
    # NOTE: Requires Chocolatey installed. [7](https://community.chocolatey.org/packages/kubeval)
  Option 2 — Download zip from GitHub releases:
    - Visit: https://github.com/instrumenta/kubeval/releases (download kubeval-windows-amd64.zip)
      Unzip and place kubeval.exe in your PATH.
      Docs: https://github.com/instrumenta/kubeval/blob/master/docs/installation.md [8](https://github.com/instrumenta/kubeval/releases)[9](https://github.com/instrumenta/kubeval/blob/master/docs/installation.md)

Run validation (from the folder containing your YAMLs):
  kubeval mongo_secret.yaml
  kubeval mongo_statefulset.yaml
  kubeval mongo_service.yaml
  kubeval node_deployment.yaml
  kubeval node_service.yaml
  kubeval react_deployment_service.yaml
# NOTE: kubeval checks schema; it cannot validate app-specific values (e.g., your image names). Fix any schema errors it reports.

[Image: Screenshot placeholder — “kubeval output: valid/invalid resources list”]

----------------------------------------------------------------
D. DEPLOYMENT EXECUTION (kubectl apply)
----------------------------------------------------------------
Goal: Create resources in the right order.

1) Ensure kubectl is pointing at Minikube:
   kubectl cluster-info
   kubectl get nodes
# NOTE: You should see a single Minikube node (Ready). If not, re-run minikube start. [2](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

2) Apply manifests (recommended order):
   kubectl apply -f mongo_secret.yaml
   kubectl apply -f mongo_statefulset.yaml
   kubectl apply -f mongo_service.yaml
   kubectl apply -f node_deployment.yaml
   kubectl apply -f node_service.yaml
   kubectl apply -f react_deployment_service.yaml

3) Watch pods until Ready:
   kubectl get pods -w

[Image: Screenshot placeholder — “Pods list showing mongo-0, node-api-*, react-web-* in Running/Ready state”]

----------------------------------------------------------------
E. VALIDATE & ACCESS
----------------------------------------------------------------
1) Check Services:
   kubectl get svc
# NOTE: You should see react-svc and node-svc as NodePort services (with ports like 30081, 30080). The mongo-svc is ClusterIP for in-cluster access.

2) Get Minikube IP:
   minikube ip
# NOTE: On Windows, use this IP with NodePort to access services from your browser or Postman. [4](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2Fchocolatey)

3) Open React UI:
   - Browser: http://<MINIKUBE_IP>:30081/
   # NOTE: If your frontend calls the API, ensure you built the React image with the correct API base URL
   # (e.g., http://<MINIKUBE_IP>:30080). React apps typically read env at build-time, not runtime.

4) Test API health:
   curl http://<MINIKUBE_IP>:30080/healthz
   # NOTE: Use Windows curl or Postman. Adjust the path to match your Node app’s health endpoint (/healthz as in the Deployment).

Alternative: minikube service helper
   minikube service react-svc --url
   minikube service node-svc --url
# NOTE: This prints a direct URL to open in your browser (handy on Windows). Behavior and usage are covered in Minikube docs. [4](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2Fchocolatey)

[Image: Diagram placeholder — “Host browser -> NodePort -> react-svc/node-svc -> Pods”]

----------------------------------------------------------------
F. CLEANUP
----------------------------------------------------------------
Delete resources:
  kubectl delete -f react_deployment_service.yaml
  kubectl delete -f node_service.yaml
  kubectl delete -f node_deployment.yaml
  kubectl delete -f mongo_service.yaml
  kubectl delete -f mongo_statefulset.yaml
  kubectl delete -f mongo_secret.yaml
# NOTE: This order avoids dangling references (deployments before services is fine; Secrets last).

Stop cluster (optional):
  minikube stop

----------------------------------------------------------------
G. APPENDIX — TROUBLESHOOTING & OPTIONAL INGRESS
----------------------------------------------------------------
Common issues:
  - “ImagePullBackOff”: Ensure image names are accessible from Minikube (local Docker or a registry). If using AWS ECR, log Docker in and retag images.
  - “CrashLoopBackOff (mongo)”: Re-check root user/pass in mongo_secret.yaml; confirm probes.
  - “React cannot reach API”: Confirm you built React with the correct API base URL (NodePort or use Ingress).

Optional — Enable Ingress (if you prefer path-based routing like /api and /)
  minikube addons enable ingress
  # NOTE: In Minikube, NGINX Ingress can be enabled as an addon. Verify controller Pods in ingress-nginx namespace. [10](https://docs.w3cub.com/kubernetes/tasks/access-application-cluster/ingress-minikube.html)

Optional — Ingress DNS (avoid editing hosts when using multiple hostnames)
  minikube addons enable ingress-dns
  # NOTE: Minikube’s ingress-dns addon lets you point your host’s DNS to Minikube IP and resolve ingress hosts automatically. [11](https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns/)

Windows/WSL2 nuances:
  - If you run Minikube in WSL2 and use Docker runtime, Ingress exposure may require extra steps; see guidance on WSL2 + Minikube ingress config. [12](https://hellokube.dev/posts/configure-minikube-ingress-on-wsl2/)

[Image: Diagram placeholder — “Ingress routing: host ‘movies.local’ -> NGINX Ingress -> react-svc (/), node-svc (/api)”]

VERSION COMPATIBILITY NOTES
  - Install kubectl version within one minor version of your cluster to avoid skew issues (Kubernetes docs). [2](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

END OF README
