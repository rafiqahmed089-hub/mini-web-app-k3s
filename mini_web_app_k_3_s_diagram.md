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
    style K3S fill:#e0ffe0,stroke:#333,stroke-width:1p