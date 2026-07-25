# ☁️ Self-Hosted Nextcloud Infrastructure

A secure, high-performance, containerized Nextcloud deployment running on Docker, managed via Traefik reverse proxy and exposed securely through a Cloudflare Zero Trust Tunnel.

---

## 🏗️ Architecture Overview

Traffic is routed securely through Cloudflare's Edge Network into a local `cloudflared` container, passing through Traefik to the Nextcloud application. Internal database and caching services are strictly isolated on a private Docker network.

```mermaid
graph TD
    Client[Internet / Users] -->|HTTPS| CF[Cloudflare Edge / Tunnel]
    
    subgraph Host Server
        subgraph Proxy Network
            CF -->|Encrypted Tunnel| CFTunnel[cloudflared Container]
            CFTunnel --> Traefik[Traefik Reverse Proxy]
            Traefik --> App[Nextcloud App Container]
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

    