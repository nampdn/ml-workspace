# ML Workspace - Architecture Specification

## Overview

ML Workspace is an all-in-one web-based development environment for machine learning, packaged as a Docker container. It provides a unified access point through an Nginx reverse proxy that routes to multiple internal services managed by Supervisor.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ML Workspace Container                        │
│                                                                        │
│  ┌──────────────┐     ┌──────────────────────────────────────────┐  │
│  │   Client     │──────│  Nginx (OpenResty) - Port 8080          │  │
│  │  (Browser)   │◄────│  - Reverse Proxy + Lua Auth               │  │
│  └──────────────┘     │  - WebSocket Support                       │  │
│                         │  - SSL/TLS Termination                      │  │
│                         └──────────────────────────────────────────┘  │
│                                      │                                 │
│                    ┌─────────────────┼────────────────┐               │
│                    │                 │                 │               │
│            ┌──────┴───┐    ┌───────┴────┐   ┌───────┴────┐          │
│            │ Supervisor │    │  Jupyter    │   │    VNC      │          │
│            │  (Master)  │◄───│  :8090     │   │  :5901      │          │
│            └───────────┘    └────────────┘   └─────────────┘          │
│                    │                 │                 │               │
│            ┌───────┼───┐    ┌───────┼────┐   ┌───────┼────┐          │
│            │  SSHD    │    │  VSCode  │   │  NetData   │          │
│            │  :22     │    │  :8054   │   │  :8050     │          │
│            └──────────┘    └──────────┘   └───────────┘              │
│                    │                 │                 │               │
│            ┌───────┼───┐    ┌───────┼────┐   ┌───────┼────┐          │
│            │  SSLH    │    │  Ungit   │   │  Glances   │          │
│            │  :8080   │    │  :8092   │   │  :8053     │          │
│            └──────────┘    └──────────┘   └───────────┘              │
│                    │                 │                 │               │
│            ┌───────┼───┐    ┌───────┼────┐   ┌───────┼────┐          │
│            │  noVNC   │    │FileBrowser│ │  Cron      │          │
│            │  :6901   │    │  :8055   │   │  (systemd) │          │
│            └──────────┘    └──────────┘   └───────────┘              │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  Persistent Volume: /workspace                                    │ │
│  │  - User projects, notebooks, data                                 │ │
│  │  - Backed up via CONFIG_BACKUP_ENABLED                           │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Architecture Layers

### 1. Container Foundation
| Component | Details |
|-----------|---------|
| Base Image | Ubuntu 20.04 |
| Init System | tini (PID 1, process group killing) |
| Process Manager | Supervisor (manages all internal services) |
| User | root (full sudo privileges) |

### 2. Runtime Environment
| Component | Version/Path |
|-----------|--------------|
| Python | 3.8.10 (Miniconda 3 at /opt/conda) |
| Node.js | 14.x |
| Conda | 4.9.2 |
| pyenv | /resources/.pyenv |
| Shell | zsh (with oh-my-zsh) |

### 3. Reverse Proxy Layer (OpenResty/Nginx)

Nginx (OpenResty flavor) acts as the single entry point on port **8080** (configurable via `WORKSPACE_PORT`). It handles:

- **Request Routing**: Directs traffic to appropriate backend services
- **Authentication**: Lua-based auth via Jupyter ping endpoint or basic auth
- **WebSocket Support**: Upgrades connections for Jupyter kernels, VNC, terminals
- **SSL/TLS**: Optional HTTPS termination
- **Base URL Rewriting**: Supports subpath deployment (e.g., behind JupyterHub)

**Routing Table:**

| URL Pattern | Backend Service | Port |
|-------------|----------------|------|
| `/` | Jupyter Notebook | 8090 |
| `/tools/vnc` | noVNC (WebSocket) | 6901 |
| `/tools/netdata` | NetData | 8050 |
| `/tools/ungit` | Ungit | 8092 |
| `/tools/glances` | Glances | 8053 |
| `/tools/vscode` | VS Code Server | 8054 |
| `/tools/filebrowser` | File Browser | 8055 |
| `/tools/<port>` | Dynamic Port Access | <port> |
| `/shared/...` | Token-protected shared resources | varies |

### 4. Service Layer (Supervisor Managed)

Supervisor orchestrates all internal services. Each service runs on a dedicated port:

| Service | Port | Purpose |
|---------|------|---------|
| Jupyter | 8090 | Primary IDE - notebooks, terminals, file browser |
| VS Code | 8054 | Web-based code editor (code-server) |
| VNC Server | 5901 | Desktop GUI (TigerVNC) |
| noVNC | 6901 | WebSocket VNC client |
| NetData | 8050 | Real-time hardware monitoring |
| Glances | 8053 | Alternative hardware monitoring |
| Ungit | 8092 | Web-based Git client |
| File Browser | 8055 | File sharing and management |
| SSHD | 22 | SSH access for remote development |
| SSLH | 8080 | SSH + HTTPS multiplexer |
| Cron | - | Scheduled tasks |
| rsyslog | - | System logging |

### 5. Data Science Stack

**Core Libraries (Conda):**
- TensorFlow, PyTorch, Keras, Scikit-learn, XGBoost
- NumPy, SciPy, Pandas, Matplotlib
- MKL-optimized BLAS, OpenMP threading

**Jupyter Ecosystem:**
- Jupyter Notebook 6.4.x
- JupyterLab 3.0.x
- nbconvert, ipython
- Extensions: toc2, jupytext, nbdime, tensorboard, codefolding

**VS Code Extensions:**
- ms-python, ms-toolsai.jupyter
- prettier, code-runner, eslint

---

## Startup Flow

```
docker run mltooling/ml-workspace
        │
        ▼
┌─────────────────────────────────┐
│ /tini (PID 1)                  │
│   └─ python docker-entrypoint.py│
└─────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────┐
│ docker-entrypoint.py             │
│ 1. Resolve WORKSPACE_BASE_URL  │
│ 2. Set MAX_NUM_THREADS (auto)  │
│ 3. Check EXECUTE_CODE env var  │
│ 4. Call run_workspace.py       │
└─────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────┐
│ run_workspace.py                 │
│ 1. Copy tutorials (if empty)   │
│ 2. Restore config backup        │
│ 3. Configure SSH                 │
│ 4. Configure Nginx (template)   │
│ 5. Configure tools              │
│ 6. Configure cron               │
│ 7. Run custom scripts           │
│ 8. Start supervisord            │
└─────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────┐
│ supervisord                      │
│ Spawns all managed services:   │
│ - nginx, jupyter, vncserver,   │
│ - novnc, sshd, sslh,           │
│ - netdata, glances, ungit,     │
│ - vscode, filebrowser, cron,   │
│ - rsyslog                        │
└─────────────────────────────────┘
```

---

## Configuration via Environment Variables

| Variable | Default | Purpose |
|-----------|---------|---------|
| `WORKSPACE_BASE_URL` | `/` | Base URL path for subpath deployment |
| `WORKSPACE_PORT` | `8080` | Main container port |
| `WORKSPACE_SSL_ENABLED` | `false` | Enable HTTPS |
| `AUTHENTICATE_VIA_JUPYTER` | `false` | Token/cookie auth via Jupyter |
| `WORKSPACE_AUTH_USER` | - | Basic auth username |
| `WORKSPACE_AUTH_PASSWORD` | - | Basic auth password |
| `CONFIG_BACKUP_ENABLED` | `true` | Backup/restore user config |
| `SHARED_LINKS_ENABLED` | `true` | Enable token-based sharing |
| `INCLUDE_TUTORIALS` | `true` | Copy tutorials on first run |
| `MAX_NUM_THREADS` | `auto` | Thread count for MKL/OMP |
| `SHUTDOWN_INACTIVE_KERNELS` | `false` | Auto-shutdown idle kernels |
| `EXECUTE_CODE` | - | Run as job mode |
| `VNC_PW` | `vncpassword` | VNC password |
| `VNC_RESOLUTION` | `1600x900` | Desktop resolution |

---

## Image Flavors

| Flavor | Docker Image | Description |
|--------|--------------|-------------|
| Full | `mltooling/ml-workspace` | Complete ML stack with all libraries |
| Minimal | `mltooling/ml-workspace-minimal` | Core tools, no heavy ML libs |
| Light | (intermediate) | Subset of full, no heavy GUI tools |
| GPU | `mltooling/ml-workspace-gpu` | CUDA 11.2, GPU-ready ML libs |
| R | `mltooling/ml-workspace-r` | R runtime + RStudio |
| Spark | `mltooling/ml-workspace-spark` | Spark + Hadoop + Zeppelin |

---

## Directory Structure

```
/
├── /workspace              # Persistent user data (volume mount point)
├── /resources             # Static resources, configs, scripts
│   ├── /branding         # Logos, favicons
│   ├── /config           # System configs (xrdp, apt)
│   ├── /icons            # Desktop icons for tools
│   ├── /jupyter          # Jupyter configs, extensions
│   ├── /libraries        # requirements*.txt
│   ├── /nginx            # nginx.conf, lua plugins
│   ├── /novnc            # noVNC web client
│   ├── /netdata          # NetData configs
│   ├── /scripts          # Python config/startup scripts
│   ├── /ssh               # SSH client/server configs
│   ├── /supervisor       # supervisord.conf + program confs
│   ├── /tools            # Tool installer shell scripts
│   ├── /tests            # Test notebooks and scripts
│   ├── /tutorials        # Welcome/tutorial notebooks
│   ├── /licenses         # Package license info
│   └── /reports          # Security scan reports
├── /opt/conda            # Miniconda installation
├── /opt/node             # Node.js binaries
├── /etc/nginx            # Nginx configuration
├── /etc/supervisor       # Supervisor configuration
├── /etc/ssh               # SSH configuration
├── /var/log/supervisor   # Supervisor logs
└── /var/log/nginx        # Nginx logs
```

---

## Security Model

1. **Authentication**: Token-based (Jupyter) or Basic Auth (Nginx)
2. **Shared Links**: SHA1-hashed tokens for file/port sharing
3. **SSL/TLS**: Optional self-signed or mounted certificates
4. **Isolation**: Docker container boundaries
5. **SSH**: Key-based passwordless access for remote development

---

## Deployment Targets

| Platform | Method |
|-----------|--------|
| Local Docker | `docker run -p 8080:8080 mltooling/ml-workspace` |
| Google Cloud Run | See `deployment/google-cloud-run/` |
| Play-with-Docker | See `deployment/play-with-docker/docker-compose.yml` |
| Kubernetes | Via ML Hub (JupyterHub-based multi-user) |
| GPU | `docker run --gpus all mltooling/ml-workspace-gpu` |

---

## Key Design Decisions

1. **Single Entry Point**: All tools accessible through port 8080 via Nginx routing
2. **Supervisor Orchestration**: Centralized process management ensures all services start and recover automatically
3. **Lua-based Auth**: Nginx uses OpenResty Lua to authenticate requests against Jupyter before allowing tool access
4. **Dynamic Configuration**: Nginx config is templated at runtime based on environment variables
5. **Volume Strategy**: `/workspace` is the sole persistent volume; other data is ephemeral
6. **Thread Optimization**: Auto-detection of CPU quota in Docker to set optimal threading for ML workloads