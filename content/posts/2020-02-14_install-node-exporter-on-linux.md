+++
title = "Install Node Exporter on Linux Servers"

[taxonomies]
tags = ["Linux"]
+++

### Installing node Exporter on Linux - updated for newer versions

`cd /tmp`

Now lets run the copied URL with wget command

`wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz`

Unzip the downloaded the file using below command

`sudo tar xvfz node_exporter-*.*-amd64.tar.gz`

Move the binary file of node exporter to `/usr/local/bin` location

`sudo mv node_exporter-*.*-amd64/node_exporter /usr/local/bin/`


Create a node_exporter user to run the node exporter service

`sudo useradd -rs /bin/false node_exporter`

Create a node_exporter service file in the /etc/systemd/system directory


`sudo nano /etc/systemd/system/node_exporter.service`


```bash
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
Now lets start and enable the node_exporter service using below commands

```
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```
