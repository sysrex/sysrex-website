+++
title = "Install Node Exporter on Linux Servers"
description = "How I install Prometheus node_exporter 1.12.1 on a Linux amd64 host and check that it serves metrics on port 9100."

[taxonomies]
tags = ["Linux", "DevOps", "Monitoring"]
+++

Prometheus [node_exporter](https://github.com/prometheus/node_exporter) is what I put on a Linux host when I want CPU, memory, disk, and network metrics in Prometheus. The [RED method](/posts/red-method-monitoring/) note is the service side of the same setup. This page is the host side: a release tarball, a system user, and a systemd unit.

The commands pin [v1.12.1](https://github.com/prometheus/node_exporter/releases/tag/v1.12.1), released 14 July 2026, and the amd64 Linux build. The SHA256 below is from that release's `sha256sums.txt`. If you move the version, take the new hash from the release. Don't reuse this one. The unit is the minimum that runs. It does not set a restart policy or systemd sandboxing.

<!-- more -->

## Install the binary

```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
echo "b51d8a76aa2a9156a55d501aca6276fae09e262259a5e4e831d2c2222f084e63  node_exporter-1.12.1.linux-amd64.tar.gz" | sha256sum -c
sudo tar xzf node_exporter-1.12.1.linux-amd64.tar.gz
sudo mv node_exporter-1.12.1.linux-amd64/node_exporter /usr/local/bin/
```

The filename is explicit on purpose. A glob against `/tmp` will unpack the wrong tarball the day a second copy is sitting there.

## Run it as its own user

```bash
sudo useradd -rs /bin/false node_exporter
```

`useradd -r` creates a system user, and on most distributions a group with the same name. The unit below sets `Group=node_exporter`. If that group was not created, add it before starting the service.

## systemd unit

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

## Start it, then check the metrics

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
curl -fsS http://127.0.0.1:9100/metrics | head
```

`systemctl status` tells you the process is up. The curl tells you it is actually exporting. You want lines that start with `node_`. Prometheus scrapes that same port.
