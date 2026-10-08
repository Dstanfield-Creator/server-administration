# Prometheus node_exporter Setup

> Install node_exporter on Debian/Ubuntu as a sandboxed systemd service bound to the management interface, add custom metrics through the textfile collector, firewall it to the Prometheus server, and query it with useful PromQL.

**Status:** Active · **Updated:** 2026-10-08

Placeholders: `192.0.2.50` is this host's management address, `192.0.2.10` is the Prometheus server, `host01.example.com` is the name to show in the `instance` label.

## Install the Binary and Create the User

```bash
VER=1.8.2                                  # check https://github.com/prometheus/node_exporter/releases
ARCH=$(dpkg --print-architecture)          # amd64 or arm64
cd "$(mktemp -d)"
curl -fsSLO "https://github.com/prometheus/node_exporter/releases/download/v${VER}/node_exporter-${VER}.linux-${ARCH}.tar.gz"
curl -fsSLO "https://github.com/prometheus/node_exporter/releases/download/v${VER}/sha256sums.txt"
sha256sum --check --ignore-missing sha256sums.txt
tar xzf "node_exporter-${VER}.linux-${ARCH}.tar.gz"
sudo install -m 0755 "node_exporter-${VER}.linux-${ARCH}/node_exporter" /usr/local/bin/node_exporter

sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter
sudo install -d -o node_exporter -g node_exporter -m 0755 /var/lib/node_exporter/textfile
```

## systemd Unit

`/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
After=network-online.target
Wants=network-online.target

[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=192.0.2.50:9100 \
  --collector.textfile.directory=/var/lib/node_exporter/textfile \
  --collector.systemd
Restart=on-failure

NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX AF_NETLINK
CapabilityBoundingSet=
SystemCallFilter=@system-service
MemoryDenyWriteExecute=yes

[Install]
WantedBy=multi-user.target
```

Binding to `192.0.2.50:9100` rather than `:9100` keeps the exporter off any public interface. If the management address is assigned by DHCP and absent at boot, the bind fails and `Restart=on-failure` retries until it exists. Do not add `ProtectKernelTunables` or `PrivateDevices`; the collectors need `/sys` and `/dev` visible.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl -s http://192.0.2.50:9100/metrics | grep -E '^node_(uname|time_seconds)'
```

## Firewall: Prometheus Server Only

```bash
sudo ufw allow from 192.0.2.10 to any port 9100 proto tcp comment 'node_exporter from Prometheus'
```

## Textfile Collector for Custom Metrics

node_exporter reads every `*.prom` file in the textfile directory on each scrape. A half-written file produces a parse error, so always write to a temporary name and `mv` it into place; a rename within one filesystem is atomic. Example `/usr/local/bin/custom-metrics.sh` exposing the age of the last backup:

```bash
#!/usr/bin/env bash
set -euo pipefail
OUT=/var/lib/node_exporter/textfile/custom.prom
TMP="${OUT}.$$"

backup_ts=$(stat -c %Y /var/backups/last-success 2>/dev/null || echo 0)

cat > "$TMP" <<EOF
# HELP backup_last_success_timestamp_seconds Unix time of the last successful backup.
# TYPE backup_last_success_timestamp_seconds gauge
backup_last_success_timestamp_seconds ${backup_ts}
EOF

mv -f "$TMP" "$OUT"
```

Schedule it as root every 15 minutes and confirm the series appears:

```bash
sudo chmod 0755 /usr/local/bin/custom-metrics.sh
echo '*/15 * * * * root /usr/local/bin/custom-metrics.sh' | sudo tee /etc/cron.d/custom-metrics >/dev/null
sudo /usr/local/bin/custom-metrics.sh && curl -s http://192.0.2.50:9100/metrics | grep '^backup_'
```

`node_textfile_scrape_error` (1 when any file failed to parse) and `node_textfile_mtime_seconds` (per file) make stale or broken scripts visible; alert on both.

## Prometheus scrape_config

On the Prometheus server, in `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['192.0.2.50:9100']
        labels:
          instance: host01.example.com
```

```bash
curl -s -X POST http://localhost:9090/-/reload       # needs --web.enable-lifecycle on Prometheus
```

## Useful PromQL

```promql
# Predicted free bytes on / four days out, fitted over the last 6 h; alert when below zero
predict_linear(node_filesystem_avail_bytes{mountpoint="/", fstype!~"tmpfs|overlay"}[6h], 4 * 24 * 3600) < 0

# CPU busy percentage per instance over 5 minutes
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))

# Memory available as a percentage of total
100 * node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```

## Docker Hosts: cAdvisor

node_exporter reports host-level metrics only. For per-container CPU, memory, network, and block I/O, run cAdvisor alongside it, bind it to the management address, and scrape port 8080 with the same source-restricted firewall rule. Key series: `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`, `container_network_receive_bytes_total`, all labelled by container `name`.

```bash
docker run -d --name cadvisor --restart unless-stopped -p 192.0.2.50:8080:8080 \
  -v /:/rootfs:ro -v /var/run:/var/run:ro -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro -v /dev/disk/:/dev/disk:ro \
  --privileged --device=/dev/kmsg gcr.io/cadvisor/cadvisor:v0.49.1
```

## Related

- [UFW baseline](../hardening/ufw-baseline-linux.md)

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
