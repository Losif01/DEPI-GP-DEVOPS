
# Nextcloud Private Infrastructure 

## Overview

This repository defines the current bare-metal/containerized baseline for our private Nextcloud infrastructure. The current environment is running on an Ubuntu host using Docker Compose, prometheus and grafana were used for monitoring, using the industry standard dashboard `Node exporter full`

## 1. Current Architecture & Tech Stack

The current stack is a declarative microservice architecture, intentionally avoiding the Nextcloud "All-In-One" (AIO) image to allow for granular control and an easier migration to K8s pods.

- **Host OS:** Ubuntu Server
    
- **Container Runtime:** Docker Engine
    
- **Orchestration:** Docker Compose (Targeting migration to K8s)
    
- **Application Layer:** Nextcloud (PHP-FPM / Apache)
    
- **Database Layer:** MariaDB
    
- **Network / Ingress:** Tailscale (WireGuard mesh VPN) routing directly to the host on port 8080.
	
- **Monitoring:** there exists stacks for both prometheus and grafana at port 3000.

## 2. Producing the Current Environment

To spin up the existing baseline for testing or development, follow this standard workflow on the Ubuntu host.

### Step 2.1: Infrastructure Deployment

Navigate to the directory containing the `docker-compose.yml` and bring up the stack:
*that's it.*

```bash 
docker compose up -d
```

_Wait ~60 seconds for MariaDB to initialize and the application container to bind._

### Step 2.2: Network Overlay Configuration

The infrastructure currently bypasses traditional reverse proxies (like Nginx/Traefik) and public domains. All ingress is handled via a Tailscale subnet.

1. Install and authenticate Tailscale on the host:
    
    
    ``` bash
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up
    ```
    
2. Retrieve the host's Tailscale IP that was sent to you in mail:(`100.X.X.X`):

### Step 2.3: Application Configuration (Trusted Domains)

By default, the Nextcloud container will block traffic originating from the Tailscale interface. You must inject the VPN IP into the application's config.

Run the following command against the active container, replacing `100.X.X.X` with the server's Tailscale IP:

```bash
docker compose exec --user www-data app php occ config:system:set trusted_domains 1 --value="100.X.X.X"
```

## 3. Storage and State Management

For the upcoming Kubernetes migration, note the following persistence targets currently handled by Docker volumes. These will need to be mapped to Persistent Volumes (PV) and Persistent Volume Claims (PVC) in the cluster:

- **Database Data:** MariaDB storage (critical for user accounts and file metadata).
    
- **Nextcloud HTML/Config:** Contains the `config.php` file and installed app configurations.
    
- **Nextcloud Data:** The primary storage pool containing actual user files and the audit `nextcloud.log`.
    

## 4. Kubernetes Migration Notes (Handoff)

As we transition this stack into a Kubernetes cluster, here are the key architectural changes and challenges to anticipate:

- **Ingress Controller:** The current setup maps the Tailscale IP directly to port `8080` on the host. In K8s, we will need to determine if Tailscale will run as a sidecar container, a Subnet Router, or if we will use the Tailscale Kubernetes Operator to handle Ingress resources natively.
    
- **Manifest Conversion:** The current `docker-compose.yml` needs to be translated into standard K8s resources (Deployments, Services, ConfigMaps, Secrets). We should consider utilizing Helm to template the Nextcloud deployment.
    
- **Admin Access (`occ`):** Currently, admin tasks are executed via `docker compose exec`. We will need to document the equivalent `kubectl exec` commands for future cluster administration.
	
- **The Server Has a Skill Issue:** it only has 8GB of ram, if the cluster will be done on the same machine, it better be a lighter alternative to K8s (like K3s, Kubernetes in docker (KIND) or using distrobox, although more complex and painful)
