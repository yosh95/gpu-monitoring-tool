# GPU Monitoring Stack with DCGM Exporter, Prometheus, and Grafana

This project provides a complete, containerized monitoring stack for NVIDIA GPUs using Docker Compose. It leverages [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) to expose detailed GPU metrics, [Prometheus](https://prometheus.io/) to collect and store them, and [Grafana](https://grafana.com/) to visualize them with a pre-configured dashboard.

This setup is designed for easy deployment and is ideal for monitoring single or multiple GPU-enabled servers.


![Grafana Dashboard Screenshot](https://grafana.com/api/dashboards/12239/images/8088/image)


*(Image source: Official GrafanaLabs Dashboards Repository)*

## Features

-   **Detailed GPU Metrics**: Track key metrics like temperature, power usage, GPU and memory utilization, SM clocks, and Tensor Core activity.
-   **Pre-configured Dashboard**: Includes a ready-to-use Grafana dashboard for immediate visualization.
-   **Easy Deployment**: Set up the entire stack with a single `docker compose` command.
-   **Flexible Profiles**: Use Docker Compose profiles to deploy components selectively.
-   **Secure Credential Management**: Grafana credentials are managed via a `.env` file, which is kept separate from version control.

## Prerequisites

Before you begin, ensure you have the following installed on your host machine(s):

-   Docker Engine
-   Docker Compose (v2 or later)
-   An NVIDIA GPU
-   [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) to enable GPU support in Docker containers.

## Project Structure

The project is structured as follows. You will create your own `.env` file from the provided example.

```
.
├── .env                  # Your local credentials (created from .env.example, not in Git)
├── .env.example          # Example credentials file
├── docker-compose.yml
├── grafana/
│   ├── dashboards/
│   │   └── nvidia-dcgm-dashboard.json
│   └── provisioning/
│       ├── dashboards/
│       │   └── dashboard.yml
│       └── datasources/
│           └── datasource.yml
└── prometheus/
    └── prometheus.yml
```

### File Contents

<details>
<summary><code>.env.example</code></summary>

```env
# Example configuration for Grafana credentials.
#
# HOW TO USE:
# 1. Copy this file to a new file named .env
#    (e.g., `cp .env.example .env`)
# 2. Edit the .env file with your actual credentials.
#
# The .env file is intentionally not committed to version control (see .gitignore)
# to keep your secrets secure.

# Default Grafana admin username
GRAFANA_ADMIN_USER=admin

# --- IMPORTANT ---
# You MUST change this to a strong and unique password for production environments.
GRAFANA_ADMIN_PASSWORD=your-strong-and-secret-password
```
</details>

<details>
<summary><code>docker-compose.yml</code></summary>

```yaml
services:

  # DCGM Exporter
  dcgm-exporter:
    profiles: ['gpu-server']
    image: nvidia/dcgm-exporter:latest
    container_name: dcgm-exporter
    restart: unless-stopped
    runtime: nvidia
    privileged: true  # Must be set true for DCGM
    pid: "host"
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=all
    ports:
      - "9400:9400"  # Port for Prometheus

  # Prometheus
  prometheus:
    profiles: ['monitor']
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"  # Prometheus Web UI port
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus  # Persistent data
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    depends_on:
      - dcgm-exporter

  # Grafana
  grafana:
    profiles: ['monitor']
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"  # Grafana Web UI port
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
      - grafana_data:/var/lib/grafana  # Persistent data
    environment:
      # Reads credentials from the .env file
      - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:

```
</details>

_Other configuration files (Prometheus, Grafana provisioning) are omitted for brevity. Refer to the file tree for their contents._

## Getting Started

### 1. Set Up Your Environment File

First, create your local environment configuration file from the provided example.

```bash
cp .env.example .env
```

Next, open the newly created `.env` file and **change `GRAFANA_ADMIN_PASSWORD` to a strong, secure password.**

```env
# .env
# ...
GRAFANA_ADMIN_PASSWORD=your-new-strong-password
```

**Note:** The `.env` file should be added to your `.gitignore` file to prevent accidentally committing sensitive information to version control.

### 2. Launch the Stack

The `docker-compose.yml` is configured with profiles (`gpu-server` and `monitor`) for flexible deployment.

#### Option A: All-in-One Setup (on a single GPU machine)

If you are running everything on a single machine with GPUs, launch all services at once.

```bash
docker compose up -d
```

This will start `dcgm-exporter`, `prometheus`, and `grafana`.

#### Option B: Distributed Setup

If you wish to run the monitoring stack on a different server from your GPU machine(s):

1.  **On each GPU server:**
    Run only the `dcgm-exporter` service. Ensure port `9400` is accessible from your monitoring server.
    ```bash
    docker compose --profile gpu-server up -d
    ```

2.  **On the monitoring server:**
    -   Modify `prometheus/prometheus.yml` to point to your GPU server's IP address.
    -   Launch the monitoring services:
        ```bash
        docker compose --profile monitor up -d
        ```

### 3. Access the Services

-   **Grafana**: Open your browser and navigate to `http://<your-server-ip>:3000`.
    -   Log in with the credentials you configured in your `.env` file.
    -   The NVIDIA DCGM Exporter dashboard will be pre-installed and available on the home page.

-   **Prometheus**: Access the web UI at `http://<your-server-ip>:9090`.
    -   You can check the status of the `dcgm-exporter` target under `Status > Targets`.

## Configuration

### Grafana Credentials

To change the Grafana admin username or password, simply edit the `.env` file and restart the Grafana container for the changes to take effect:

```bash
# After modifying .env
docker compose restart grafana
```

### Monitoring Multiple GPU Servers

To monitor multiple GPU servers, deploy the `dcgm-exporter` on each one and add their IP addresses to the `targets` list in `prometheus/prometheus.yml`.

```yaml
# prometheus/prometheus.yml
scrape_configs:
  - job_name: 'dcgm-exporter'
    static_configs:
      - targets:
        - 'gpu-server-1-ip:9400'
        - 'gpu-server-2-ip:9400'
        - 'gpu-server-3-ip:9400'
```

After updating the configuration, restart Prometheus: `docker compose restart prometheus`.

## Stopping the Stack

To stop and remove all the containers:

```bash
docker compose down
```

To also remove the data volumes (Prometheus metrics, Grafana settings), use the `-v` flag:

```bash
docker compose down -v
```

## License

This project is licensed under the Apache License, Version 2.0. See the [LICENSE](http://www.apache.org/licenses/LICENSE-2.0) file for the full text.
