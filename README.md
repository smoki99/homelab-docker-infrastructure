# ☁️ Homelab Docker Infrastructure

A modular, decoupled Infrastructure as Code (IaC) setup for hosting containerized web services using Docker, Traefik, and Cloudflare Zero Trust Tunnels.

---

## 🏗️ Architecture Overview

The infrastructure is split into decoupled stacks connected via a central, shared Docker network (`proxy`). Ingress and reverse proxy services are isolated from application-specific databases and caches.

```mermaid
graph TD
    Client[Internet / Users] -->|HTTPS| CF[Cloudflare Edge Network]
    
    subgraph Host Server
        subgraph Shared Proxy Network
            CF -->|Encrypted Tunnel| CFTunnel[cloudflared Container]
            CFTunnel --> Traefik[Traefik Reverse Proxy]
            Traefik -->|Discovered via Docker Socket| App[Nextcloud App Container]
            Traefik -.->|Future Routing| FutureApp[Future Apps / Services]
        end
        
        subgraph Private Internal Network
            App -->|Database Traffic| DB[(MariaDB Container)]
            App -->|Memory Cache| Redis[(Redis Container)]
        end
        
        subgraph Storage Mounts
            App ---|HTML Dataset| HTMLData[/mnt/.../nextcloud_html\]
            DB ---|DB Dataset| DBData[/mnt/.../nextcloud_db\]
        end
    end


ONLYOFFICE_JWT_SECRET
Generate with openssl rand -hex 32