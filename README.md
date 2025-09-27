# GPU Monitoring Stack with DCGM Exporter, Prometheus, and Grafana

This project provides a complete, containerized monitoring stack for NVIDIA GPUs using Docker Compose. It leverages [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) to expose detailed GPU metrics, [Prometheus](https://prometheus.io/) to collect and store them, and [Grafana](https://grafana.com/) to visualize them with a pre-configured dashboard.

This setup is designed for easy deployment and is ideal for monitoring single or multiple GPU-enabled servers.


![Grafana Dashboard Screenshot](https://grafana.com/api/dashboards/12239/images/8088/image)

*(Image source: Official GrafanaLabs Dashboards Repository)*

## Features

-   **Detailed GPU Metrics**: Track key metrics like temperature, power usage, GPU and memory utilization, SM clocks, and Tensor Core activity.
-   **Pre-configured Dashboard**: Includes a ready-to-use Grafana dashboard for immediate visualization.
-   **Easy Deployment**: Set up the entire stack with a single `docker compose` command.
-   **Flexible Profiles**: Use Docker Compose profiles to deploy components selectively. This allows you to run `dcgm-exporter` on a GPU server and the monitoring components (`prometheus`/`grafana`) on a separate machine.

## Prerequisites

Before you begin, ensure you have the following installed on your host machine(s):

-   Docker Engine
-   Docker Compose (v2 or later)
-   An NVIDIA GPU
-   [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) to enable GPU support in Docker containers.

## Project Structure

Create the following directory structure and files for the project:

```
.
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
      - GF_SECURITY_ADMIN_USER=admin  # WARNING: Change in production
      - GF_SECURITY_ADMIN_PASSWORD=grafana-user  # WARNING: Change in production
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:

```
</details>

<details>
<summary><code>prometheus/prometheus.yml</code></summary>

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'dcgm-exporter'
    static_configs:
      - targets: ['dcgm-exporter:9400']
```
</details>

<details>
<summary><code>grafana/provisioning/datasources/datasource.yml</code></summary>

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```
</details>

<details>
<summary><code>grafana/provisioning/dashboards/dashboard.yml</code></summary>

```yaml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
```
</details>

<details>
<summary><code>grafana/dashboards/nvidia-dcgm-dashboard.json</code></summary>

_Place the large JSON blob you provided for the Grafana dashboard into this file._

</details>


## Getting Started

### 1. Clone or Create the Project Files

Clone this repository or manually create the files and directories as described in the **Project Structure** section above.

### 2. Launch the Stack

The `docker-compose.yml` is configured with profiles (`gpu-server` and `monitor`) for flexible deployment.

#### Option A: All-in-One Setup (on a single GPU machine)

If you are running everything on a single machine that has GPUs, you can launch all services at once.

```bash
docker compose up -d
```

This will start `dcgm-exporter`, `prometheus`, and `grafana`.

#### Option B: Distributed Setup

If you want to run the monitoring stack on a different server from your GPU machine(s), follow these steps:

1.  **On each GPU server:**
    Run only the `dcgm-exporter` service. Ensure its port `9400` is accessible from your monitoring server.
    ```bash
    docker compose --profile gpu-server up -d
    ```

2.  **On the monitoring server:**
    -   Modify `prometheus/prometheus.yml` to point to your GPU server's IP address:
        ```yaml
        # prometheus/prometheus.yml
        # ...
        static_configs:
          - targets: ['<your-gpu-server-ip>:9400'] # Add more targets if needed
        ```
    -   Launch the monitoring services:
        ```bash
        docker compose --profile monitor up -d
        ```

### 3. Access the Services

-   **Grafana**: Open your browser and navigate to `http://<your-server-ip>:3000`.
    -   Default username: `admin`
    -   Default password: `grafana-user`
    -   The NVIDIA DCGM Exporter dashboard will be pre-installed and available on the home page.

-   **Prometheus**: Access the web UI at `http://<your-server-ip>:9090`.
    -   You can check the status of the `dcgm-exporter` target under `Status > Targets`.

## Configuration

### Grafana Credentials

**It is strongly recommended to change the default Grafana admin password.** You can do this by modifying the environment variables in `docker-compose.yml` before the first launch:

```yaml
# docker-compose.yml
# ...
  grafana:
    environment:
      - GF_SECURITY_ADMIN_USER=your_new_user
      - GF_SECURITY_ADMIN_PASSWORD=your_strong_password
```

### Monitoring Multiple GPU Servers

To monitor multiple GPU servers, deploy the `dcgm-exporter` on each one (using `docker compose --profile gpu-server up -d`) and add their IP addresses to the `targets` list in `prometheus/prometheus.yml`:

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

To stop and remove all the containers, run:

```bash
docker compose down
```

To preserve the data stored in volumes (Prometheus metrics, Grafana settings), omit the `-v` flag. To remove the data as well, use:

```bash
docker compose down -v
```

## License

This project is licensed under the Apache License, Version 2.0. See the [LICENSE](http://www.apache.org/licenses/LICENSE-2.0) file for the full text.
