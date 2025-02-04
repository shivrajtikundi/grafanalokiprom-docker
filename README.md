# Installing Prometheus, Node Exporter, and Loki with Docker

Monitoring system performance and logs is essential for managing and debugging your infrastructure. This guide walks you through installing **Prometheus**, **Node Exporter**, and **Loki** using Docker.

---

## Installing Prometheus and Node Exporter

### Step 1: Install Node Exporter

Node Exporter collects hardware and OS metrics exposed by Linux kernels. To install it, follow these steps:

1. Download the latest **Node Exporter** release from [GitHub](https://github.com/prometheus/node_exporter/releases):
   
   wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
   
2. Extract and move the binary to `/usr/local/bin/`:
   ```sh
   tar -xvf node_exporter-1.8.2.linux-amd64.tar.gz
   sudo mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
   ```
3. Create a **Node Exporter** service file:
   ```sh
   sudo nano /etc/systemd/system/node_exporter.service
   ```
   Add the following content:
   ```ini
   [Unit]
   Description=Node Exporter
   After=network.target

   [Service]
   User=ubuntu
   ExecStart=/usr/local/bin/node_exporter

   [Install]
   WantedBy=default.target
   ```
4. Reload the system daemon and start the service:
   ```sh
   sudo systemctl daemon-reload
   sudo systemctl enable node_exporter
   sudo systemctl start node_exporter
   sudo systemctl status node_exporter
   ```
5. Verify that Node Exporter is running by accessing `http://<YOUR_SERVER_IP>:9100/metrics` in your browser.

---

### Step 2: Install Prometheus with Docker

Prometheus is a powerful monitoring tool that collects and stores metrics.

Run Prometheus using Docker:
```sh
mkdir -p ~/prometheus && cd ~/prometheus
```
Create a **Prometheus configuration file**:
```sh
nano prometheus.yml
```
Add the following content:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.31.2.81:9100']
```
Run Prometheus as a Docker container:
```sh
docker run -d --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```
Access Prometheus at: `http://<YOUR_SERVER_IP>:9090`

---

## Installing Loki for Log Aggregation

Loki is a log aggregation system designed for monitoring with Grafana. Follow these steps to set it up.

1. Create a **Loki configuration file**:
   ```sh
   mkdir -p ~/loki && cd ~/loki
   nano loki-config.yml
   ```
2. Add the following Loki configuration:
   ```yaml
   auth_enabled: false

   server:
     http_listen_port: 3100

   common:
     ring:
       instance_addr: 0.0.0.0
       kvstore:
         store: inmemory
     replication_factor: 1
     path_prefix: /tmp/loki

   schema_config:
     configs:
     - from: 2020-05-15
       store: tsdb
       object_store: filesystem
       schema: v13
       index:
         prefix: index_
         period: 24h

   storage_config:
     filesystem:
       directory: /tmp/loki/chunks
   ```
3. Run Loki using Docker:
   ```sh
   docker run -d --name=loki \
     -p 3100:3100 \
     -v $(pwd)/loki-config.yml:/etc/loki/loki-config.yml \
     grafana/loki:latest -config.file=/etc/loki/loki-config.yml
   ```

---

## Setting Up Docker Logging to Loki

To send Docker logs to Loki, install the Loki Docker logging driver:
```sh
docker plugin install grafana/loki-docker-driver --alias loki --grant-all-permissions
```

### Configure Docker to Use Loki
Modify the **Docker daemon configuration file**:
```sh
sudo nano /etc/docker/daemon.json
```
Add the following configuration:
```json
{
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "http://<LOKI_SERVER_IP>:3100/loki/api/v1/push",
    "loki-retries": "5",
    "loki-batch-size": "400"
  }
}
```
Restart Docker:
```sh
sudo systemctl restart docker
```

Run a test container to verify logs are sent to Loki:
```sh
docker run -d --log-driver=loki --log-opt loki-url="http://<LOKI_SERVER_IP>:3100/loki/api/v1/push" --name test-container alpine echo "Hello Loki"
```

---

## Installing Grafana for Monitoring

Grafana is a visualization tool used to analyze metrics and logs from Prometheus and Loki.

Run Grafana with Docker:
```sh
docker run -d --name=grafana \
  -p 3000:3000 \
  -e GF_SECURITY_ADMIN_USER=admin \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  grafana/grafana
```

Access Grafana at: `http://<YOUR_SERVER_IP>:3000`

### Add Data Sources in Grafana:
1. **Loki:**
   - Navigate to **Settings** → **Data Sources** → **Add Data Source**
   - Select **Loki** and set the URL: `http://<LOKI_SERVER_IP>:3100`
2. **Prometheus:**
   - Navigate to **Settings** → **Data Sources** → **Add Data Source**
   - Select **Prometheus** and set the URL: `http://<PROMETHEUS_SERVER_IP>:9090`

---

## Conclusion

By following this guide, you have successfully:
✅ Installed **Node Exporter** for system metrics
✅ Set up **Prometheus** for monitoring
✅ Configured **Loki** for log aggregation
✅ Integrated logs and metrics with **Grafana**

Now, you can visualize system performance and logs effectively! 🚀

