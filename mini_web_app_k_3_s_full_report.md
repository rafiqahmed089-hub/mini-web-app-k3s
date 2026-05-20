# Mini Web App on K3s - Architecture and Deployment Report

## Title: Mini Web App K3s Deployment and Architecture

### 1. Overview
This project demonstrates a lightweight web application deployed on **K3s**, a lightweight Kubernetes distribution. The application consists of a **frontend** (Nginx), a **backend API** (Flask), and a **local registry** for hosting Docker images.

### 2. Architecture Diagram
```mermaid
flowchart TD

    subgraph User_Browser
        U[Browser] -->|http://192.168.56.104:30081| FSVC[frontend-service\nType: NodePort\nPort: 80\nNodePort: 30081]
    end

    FSVC --> FP1[frontend Pod 1\nNginx\nPort 80]
    FSVC --> FP2[frontend Pod 2\nNginx\nPort 80]

    FP1 --> HTML[index.html]
    FP2 --> HTML

    HTML -->|fetch('/api')| APIREQ[/api request]

    APIREQ ==> BSVC[backend-service\nType: ClusterIP\nPort: 5000]

    BSVC --> BP1[backend Pod 1\nFlask app.py\nPort 5000]
    BSVC --> BP2[backend Pod 2\nFlask app.py\nPort 5000]

    subgraph Registry
        RSVC[registry Service\nType: NodePort\nPort: 5000\nNodePort: 30500] --> RPOD[registry Pod\nimage: registry:2\nPort 5000]
    end

    K3S[K3s + containerd] -->|pull images| RSVC
    K3S -->|pull frontend image| FP1
    K3S -->|pull frontend image| FP2
    K3S -->|pull backend image| BP1
    K3S -->|pull backend image| BP2

    style User_Browser fill:#f0f8ff,stroke:#333,stroke-width:1px
    style Registry fill:#ffe4b5,stroke:#333,stroke-width:1px
    style K3S fill:#e0ffe0,stroke:#333,stroke-width:1px
```

### 3. Steps We Followed

1. **Setup K3s** on the VM with proper networking:
   - NAT adapter for internet access
   - Host-only adapter for host-VM communication

2. **Created backend (Flask) and frontend (Nginx) Dockerfiles**:
   - Backend listens on port 5000
   - Frontend listens on port 80 and proxies `/api` requests to backend-service

3. **Created local registry inside K3s** on NodePort 30500 for hosting Docker images.

4. **Configured DNS** in the VM:
   - Fixed `Temporary failure in name resolution`
   - Added `DNS=8.8.8.8 1.1.1.1` and `FallbackDNS=8.8.4.4` in `/etc/systemd/resolved.conf`

5. **Pulled base images** (`nginx:alpine`, `python:3.11-alpine`) successfully.

6. **Built and pushed frontend/backend images** to local registry:
   - Tagged with `192.168.56.104:30500/frontend:1.1` and `backend:1.1`
   - Configured Docker and K3s containerd to allow insecure HTTP registry

7. **Fixed backend deployment YAML** indentation issues to resolve `error converting YAML to JSON`.

8. **Applied deployments** and ensured pods became `Running`:
   - Verified frontend served `index.html`
   - Verified `/api` returned `Hello from Flask backend!`

9. **Cleaned up old terminating pods** after redeployments.

### 4. Challenges Faced

1. **DNS Resolution Failure**:
   - Initially, `ping google.com` failed inside the VM
   - Solved by updating `/etc/systemd/resolved.conf` and restarting `systemd-resolved`

2. **HTTP Registry Issue**:
   - Push to `192.168.56.104:30500` failed due to HTTPS expectation
   - Solved by adding insecure registry config for Docker and K3s containerd

3. **YAML Indentation Error**:
   - `backend deployment` failed with `yaml: line 16: did not find expected key`
   - Fixed proper 2-space indentation under `containers:` and `image:`

4. **ImagePullBackOff / ErrImagePull**:
   - Occurred before configuring K3s to use HTTP registry
   - Resolved after rebuilding, pushing, and configuring `registries.yaml`

5. **Frontend Proxy**:
   - Browser `fetch('/api')` would fail until Nginx config was added to proxy `/api` to backend-service

### 5. Final Verification

- **Frontend access:** `http://192.168.56.104:30081`
- **Backend API via frontend:** `http://192.168.56.104:30081/api` → `{"message":"Hello from Flask backend!"}`
- **Pods status:** all frontend and backend pods `Running`
- **Registry access:** `curl http://192.168.56.104:30500/v2/` → `{}`

### 6. Conclusion

The mini web app is now fully deployed on K3s, with:
- Frontend Nginx pods accessible via NodePort
- Backend Flask pods accessible via ClusterIP service
- Local HTTP registry hosting images
- Proper DNS and registry configuration
- All previous issues resolved, including ImagePullBackOff and YAML syntax errors.

