# ☁️ Homelab Docker Infrastructure

A modular, decoupled Infrastructure as Code (IaC) setup for hosting web services on Docker, Traefik, and Cloudflare Zero Trust Tunnels.

---

## 🏗️ Architecture Overview

The infrastructure is split into decoupled stacks using an external shared Docker network (`proxy`). Networking services (Traefik & Cloudflared) are isolated from application-specific databases and caches.

```mermaid
graph TD
    Client[Internet / Users] -->|HTTPS| CF[Cloudflare Edge / Tunnel]
    
    subgraph Host Server
        subgraph Shared Proxy Network
            CF -->|Encrypted Tunnel| CFTunnel[cloudflared Container]
            CFTunnel --> Traefik[Traefik Reverse Proxy]
            Traefik -->|Discovered via Socket| App[Nextcloud App Container]
            Traefik -.->|Future Apps| FutureApp[Future Container / App]
        end
        
        subgraph Private Internal Network
            App -->|Database Traffic| DB[(MariaDB Container)]
            App -->|Memory Cache| Redis[(Redis Container)]
        end
        
        subgraph Storage Mounts
            App ---|HTML Dataset| HTMLData[/mnt/slow-data-pool-1/apps/nextcloud_html\]
            DB ---|DB Dataset| DBData[/mnt/slow-data-pool-1/apps/nextcloud_db\]
        end
    end